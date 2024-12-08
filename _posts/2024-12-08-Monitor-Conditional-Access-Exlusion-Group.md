---
title: Monitor conditional access exlusion group changes in Microsoft Sentinel
date: 2024-12-08 5:00:00
categories: [Microsoft Sentinel, Analytic Rule]
tags: [Azure,Guide,Security,SecurityAutomation,SIEM,SOAR,Entra ID, Conditional Access, Analytic Rule, Monitoring, Watchlist]
layout: post
toc: true
comments: false
math: false
mermaid: false
published: true
---

I've seen Entra ID directories containing varying numbers of groups, ranging from just a few dozen in smaller environments to tens of thousands in larger ones. Amid this "group jungle," certain groups are considered sensitive, such as privileged administrator groups, critical application role groups, and conditional access policy groups, among others. In this post, I’ll share my approach to monitoring these sensitive groups using Microsoft Sentinel.

For this example I'll focus on monitoring my tenants **Conditional Access exclusion group**. This group excludes members from Conditional Access policies, and it's essential to track any changes to its membership. While excluding users from Conditional Access can be legitimate with proper justification, it could also indicate misconfigurations or even malicious actions.

So here's my approach to monitoring Conditional Access exclusion group membership changes. This example is simple and can be easily adapted for other use cases as well.

## Step 1: Prepare a Watchlist

First, we'll create a **CSV file** to define the group(s) you want to monitor. In my example the CSV file includes a single column with a header and the **Object IDs** of the groups you wish to track.

### CSV File Contents

Save your file as `MonitoredGroupMembershipChanges.csv`, and structure it like this:

```
GroupObjectId
000000-Your-Object-ID-Here
000000-Your-Object-ID-Here
```


### Navigating to Watchlist in Microsoft Sentinel

1. Open **Microsoft Sentinel** and navigate to the **Watchlist** section.
2. Start creating a new watchlist.
#### General Tab:
- **Name and Alias**: Enter a descriptive name, such as `MonitoredGroupMembershipChanges`. In this example I am using the same name for both fields for consistency.
- **Description**: Add a clear description explaining the watchlist's purpose, e.g., _"Tracks membership changes for sensitive Conditional Access exclusion groups."_ Description really helps down in the line to identify the purpose for you and for others.
#### Source Tab:

Specify the following parameters:
- **Source**: Local file
- **File Type**: CSV file with header (.csv)
- **Number of lines before row with headings**: 0
- **Upload File**: Select and upload your `MonitoredGroupMembershipChanges.csv` file.
- **SearchKey**: Set this to `GroupObjectId`.

![Picture 1. Watchlist source tab](/assets/img/2024-12-08-Monitor-Conditional-Access-Exlusion-Group/1-Watchlist-Settings-Source.png)

> **Tip**: If you renamed the header column in your CSV file, update the `SearchKey` value to match it.

On the right side of the **Source** tab, you'll see a preview of your CSV file. Confirm that the preview matches your file's contents.

Now you can **Review + Create** to complete the process and save your watchlist

## Step 2: Prepare an Analytic Rule

### Navigating to Analytics in Microsoft Sentinel

1. Open **Microsoft Sentinel** and navigate to the **Analytics** section.
2. Start creating a new **Scheduled Query Rule**.
#### General Tab:
- **Name**: Specify a meaningful name, e.g., _"Conditional Access Exclusion Group Membership Changes"_.
- **Description**: Add a detailed description explaining the purpose of the rule. I've added description in bellow 

```
This query monitors Entra ID audit logs for group membership changes (adding or removing members) in groups specified in the "MonitoredGroupMembershipChanges" watchlist. It checks if the TargetResources field in the logs contains any group listed in the watchlist and captures details about the activity, including the initiator, initiator IP address and target user. This rule helps track critical changes to groups used in conditional access exclusions.
```

- **Severity**: Specify the severity level of the incident based on its potential impact.
- **MITTRE ATT&CK**: Specify relevant tactics and techniques for the incident. For this example, I am using **Command and Control** and **Defense Evasion** tactics.

#### Set rule logic Tab:
- **Rule Query**: Paste the query below, modifying `MonitoredGroupMembershipChanges` to match the name of your watchlist.

```js
let watchlistItems = _GetWatchlist("MonitoredGroupMembershipChanges") | project SearchKey;

AuditLogs
| where ActivityDisplayName in ("Add member to group", "Remove member from group")
| where TargetResources has_any (watchlistItems)
| extend 
    TargetUPN = tostring(parse_json(TargetResources)[0].userPrincipalName), 
    InitiatedByUPN = tostring(parse_json(InitiatedBy).user.userPrincipalName),
    InitiatedByIp = tostring(parse_json(InitiatedBy).user.ipAddress)
| project TimeGenerated, ActivityDisplayName,InitiatedByUPN, InitiatedByIp, TargetUPN, InitiatedBy, TargetResources
```

>**Query explanation**
>1. Create the `watchlistItems` variable, which contains the `SearchKey` values from the **MonitoredGroupMembershipChanges** watchlist.
>2. Query **AuditLogs** and filter events where the **ActivityDisplayName** equals "**Add member to group**" or "**Remove member from group**."
>3. Further filter the results by checking if **TargetResources** property values contains any of the watchlist **Object ID** values.
>4. Extract the properties `TargetUPN`, `InitiatedByUPN`, and `InitiatedByIP`.
>5. Project the relevant properties for further analysis.

- **Alert enhancements**
	- **Entity mapping**
		- Map the **account** entity as **FullName** using the **TargetUPN** property.
		- Map the **IP** entity as **Address** using the **InitiatedByIP** property.
- **Query Scheduling**: I am using the default run query every 5 hours and lookup data from the last 5 hours. Adjust the schedule as needed based on your requirements.
- **Alert treshold**: Set the threshold to `0`.
- **Event grouping**: Group all events into a single alert.

Here's a example of rule query with Entity mapping:
![Picture 2. Ruel query and entity mapping example](/assets/img/2024-12-08-Monitor-Conditional-Access-Exlusion-Group/2-AnalyticRuleWizard-RuleLogic.png)

#### Finish the rule
The rest of the settings in the **Incident Settings** and **Automated Response** tabs can be customized based on your preferences. In this example, I'll simply create the rule with the current settings without modifying **Incident Settings** or **Automated Response**.
### Testing the Rule

To test the rule:

1. Add a member to a group being monitored.
2. Once the audit log data is ingested into the **Log Analytics workspace** and the rule runs, an incident should be generated.

> **Tip**: For quicker testing, consider lowering the **Query Scheduling** interval or creating a Near Real-Time (NRT) rule.

## Closing thoughts

This example provides the basics for reading group ID from watchlist for searching any membership changes from AuditLogs table. I kept the CSV and the Analytic rule as simple as possible so this could be used as a base for other group membership monitoring aswell.

Hopefully you got some ideas from this one for your specific requirements!

-Anssi

# Sources
- [Watchlists in Microsoft Sentinel](https://learn.microsoft.com/en-us/azure/sentinel/watchlists)  
- [Create scheduled analytics rules in Microsoft Sentinel ](https://learn.microsoft.com/en-us/azure/sentinel/create-analytics-rules?tabs=azure-portal)  
