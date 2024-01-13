---
title: Revoke user sign-in sessions from Entra ID using a Logic App in Azure Sentinel
date: 2024-01-13 5:00:00
categories: [Microsoft Sentinel, Logic App]
tags: [Azure,Logic App,Automation,Guide,Security,Session,SecurityAutomation,SIEM,SOAR]
layout: post
toc: true
comments: false
math: false
mermaid: false
published: true
---
# Revoke user sign-in sessions from Entra ID using a Logic App in Azure Sentinel

The original **Revoke-AADSignInSessions playbook** from the **Azure Sentinel repository**, provided by the **Microsoft Entra ID solution**, had some minor issues. Specifically, the incident-triggered playbook couldn't be attached to Sentinel's Automation Rule, preventing it from being used automatically when an incident is created. However, when I began working on it, I realized that instead of merely fixing the trigger, I could enhance the solution to better suit my requirements.

Original playbook outline: 
1. Trigger: When incident is triggered
2. Get Account entities from incident 
3. Loop through accounts 
4. Revoke user session (Contains hard-coded expression for creating User Principal Name)
5. Get users Manager 
6. Send an email to notify manager about revoked sessions 
7. Add comment to sentinel incident (Contains hard-coded expression for creating User Principal Name)

Since updating the trigger, it **removes all references from subsequent actions**, and I needed to **reconfigure every action**. Consequently, I decided to create a **much more simplified version of this playbook without email notifications**.

My current solution is as follows
1. Trigger: When incident is triggered
2. Get account entities from incident
3. Loop through accounts
4. Compose UPN
5. Revoke user session
6. Add comment to sentinel incident

You might look at this and think, "Yeah, it's just one step less than the original. Was it worth it?"

Well, yes. Although it's a subtle change, there are some quality-of-life improvements that allow me to modify this "template playbook" and refine it for future needs. The original version had hard-coded expressions to create **User Principal Names (UPNs)** for several actions. However, I wanted to consolidate this into one **compose action** where I create the UPN, which can then be used in the subsequent actions. This way, if I need to add a new action down the line, I can simply reference the existing compose action without re-creating the expressions that generate the UPN.

Additionally, I decided to remove the email functionality. Most of the time, email notifications confuse users (managers) more than they help. It's important to note that revoking user sessions isn't a significant remediation action, as affected user simply need to re-authenticate. By eliminating the email functionality, I also got rid of the **Office 365 connector** used for sending emails and a separate **Office connector** for retrieving users' manager information.

Here's a link to the [Revoke-EntraIDUserSignInSessions playbook templates in my GitHub repository](https://github.com/apaivinen/sentinel-playbooks/tree/main/Revoke-EntraIDUserSignInSessions). Below, you'll find a walkthrough of the **Incident-triggered playbook**. Additionally, there's an **entity-triggered playbook** available, but it's significantly more simplified than incident one. I won't delve into the details of the entity-triggered playbook since its core actions align with those of the incident-triggered playbook.

If you deploy any of those **Revoke-EntraIDUserSignInSessions**, it will create the following **Azure resources**:
- Logic App
- Managed identity for Logic App
- Microsoft Sentinel API connection for Managed Identity
# Incident triggered playbook specifications

The current playbook utilizes the **Sentinel connector** to retrieve incident and account details, as well as to update incidents with comments. Additionally, a **System Managed Identity** is employed by the logic app for authentication with the Sentinel connector and Graph API. The necessary permissions for this managed identity are as follows:
- Graph API: **User.ReadWrite.All** (see powershell bellow)
- Sentinel role in Azure: **Microsoft Sentinel Responder**

## Powershell to assign Graph permissions
You need to have [AzureAD Powershell module](https://learn.microsoft.com/en-us/powershell/azure/active-directory/install-adv2?view=azureadps-2.0) installed
```powershell
$MIGuid = "<Enter your managed identity guid here>"
$MI = Get-AzureADServicePrincipal -ObjectId $MIGuid

$GraphAppId = "00000003-0000-0000-c000-000000000000"
$PermissionName = "User.ReadWrite.All" 

$GraphServicePrincipal = Get-AzureADServicePrincipal -Filter "appId eq '$GraphAppId'"
$AppRole = $GraphServicePrincipal.AppRoles | Where-Object {$_.Value -eq $PermissionName -and $_.AllowedMemberTypes -contains "Application"}
New-AzureAdServiceAppRoleAssignment -ObjectId $MI.ObjectId -PrincipalId $MI.ObjectId `
-ResourceId $GraphServicePrincipal.ObjectId -Id $AppRole.Id
```

## Logic App
Here's an outline of the logic app

![Picture 1. Outline of Revoke EntraID User Sign In Sessions Incident](/assets/img/2024-01-13-Revoke-User-Sign-In-Sessions-by-Logic-App-Sentinel-Playbook/Revoke-EntraIDUserSignInSessions-Incident-outline.png)
### Microsoft Sentinel Incident
This is a **trigger action** that starts the automation. I haven't made any configurations, so it is in its default settings.

### Entities - Get Accounts
This is an action provided by Microsoft to **list all account entities from an incident**. The action requires **Entities** object (`@{triggerBody()?['object']?['properties']?['relatedEntities']}`) from the **Microsoft Sentinel Incident** trigger. 

![Picture 2. Entities Get Accounts action](/assets/img/2024-01-13-Revoke-User-Sign-In-Sessions-by-Logic-App-Sentinel-Playbook/1-Entities-Get-Accounts.png)
### For Each
Loop through **Accounts** object (`@{body('Entities_-_Get_Accounts')?['Accounts']}`) from **Entities - Get Accounts** action.

![Picture 3. For each loop](/assets/img/2024-01-13-Revoke-User-Sign-In-Sessions-by-Logic-App-Sentinel-Playbook/2-For-Each.png)
### Compose - Concat UPN
This is a **compose action** that creates the user's **User Principal Name (UPN)**, which can be used in later steps. I am using the **concat()** expression to create the UPN.

The **Graph API** requires either the user's UPN or their **Object ID** for revoking sessions. Unfortunately, the **Entities - Get Accounts** action does not provide either of these required pieces of information. Therefore, we need to craft them, and this is where the **Compose** action with **concat** comes into play.

Here's what the **Entities - Get Accounts** action returns:
```json
[
  {
    "accountName": "sus-user",
    "upnSuffix": "yourDomain.onmicrosoft.com",
    "friendlyName": "sus-user",
    "Type": "account",
    "Name": "sus-user"
  }
]
```

There are **accountName** and **upnSuffix** available. All we need to do is combine those with an "@" in between, and voilà, we have a **User Principal Name (UPN)**. `concat(items('For_each')?['Name'], '@', items('for_each')?['UPNSuffix'])`

![Picture 4. Compose action](/assets/img/2024-01-13-Revoke-User-Sign-In-Sessions-by-Logic-App-Sentinel-Playbook/3-Compose-concat-UPN.png)

### HTTP - Revoke sessions
Here's an **HTTP request** to the **Graph API endpoint** for revoking user sessions:

According to the documentation ([user: revokeSignInSessions - Microsoft Graph v1.0 | Microsoft Learn](https://learn.microsoft.com/en-us/graph/api/user-revokesigninsessions?view=graph-rest-1.0&tabs=http)), the **HTTP request** needs to be a **POST** request to the `/users/{id | userPrincipalName}/revokeSignInSessions` endpoint.

As you can see, the **least privileged permission level** required for this operation is **User.ReadWrite.All**. This is why the managed identity must be assigned the **User.ReadWrite.All** permission level.

Configure following parameters
- URI `https://graph.microsoft.com/v1.0/users/@{outputs('Compose_-_concat_UPN')}/revokeSignInSessions` 
- Method is POST
- Header: 
	- Key: Content Type 
	- Value: application/json
- From Advanced parameters select Authentication
- Authentication type: Managed Identity
- Managed Identity: System-assigned managed identity
- Audience: `https://graph.microsoft.com`

![Picture 5. HTTP request](/assets/img/2024-01-13-Revoke-User-Sign-In-Sessions-by-Logic-App-Sentinel-Playbook/4-HTTP-Revoke-sessions.png)
### Add Comments to incident (V3)
The last action is for **adding a comment to the incident** to provide feedback that something has been done.
1. **Specify the incident ARM ID**: Use the **Incident ARM ID parameter** `@{triggerBody()?['object']?['id']}` from the trigger action. This is how you specify which incident the comment is made for.
2. **Specify your comment message**: It should be informative and precise. In my case, I'm using the output from the **Compose - Concat UPN** action to specify which user sign-in sessions were revoked.
 
![Picture 6. Add comment to incident](/assets/img/2024-01-13-Revoke-User-Sign-In-Sessions-by-Logic-App-Sentinel-Playbook/5-Add-comment-to-incident.png)

# Entity triggered playbook overview

In essence, the **Entity-triggered revoke user sign-in sessions** playbook is similar to the incident-triggered version, but it is specifically targeted to a single user. Here's the playbook outline:
1. Trigger: Microsoft Sentinel entity
2. Compose - concat UPN
3. HTTP - Revoke sessions
4. Add comment to incident (V3)

![Picture 7. Entity triggered playbook outline](/assets/img/2024-01-13-Revoke-User-Sign-In-Sessions-by-Logic-App-Sentinel-Playbook/Entity-trigger-revoke-sessions.png)