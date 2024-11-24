---
title: Block Entra ID User playbook for Microsoft Sentinel
date: 2024-11-24 5:00:00
categories: [Microsoft Sentinel, Playbook]
tags: [Azure,Logic App,Automation,Guide,Security,Remidiation,SecurityAutomation,SIEM,SOAR]
layout: post
toc: true
comments: false
math: false
mermaid: false
published: true
---

This is a simple playbook designed to block an Entra ID user by updating the user's `accountEnabled` property via the Graph API.

A year ago, when I wanted to complete this exercise, I initially planned to modify the existing Microsoft Sentinel community block playbook. However, after some time, I decided to recreate it with my own touch.

My goal was to create a straightforward playbook that blocks users, adds a comment with the results to the incident, and avoids unnecessary features (to me) like email notifications. The aim is to provide a minimalistic template containing only the core features, which can easily be extended based on specific needs.

## Playbook overview

1. Incident-triggered Logic App
2. Retrieve accounts from the incident
3. Loop through the accounts
4. Block the user
5. Gather results
6. Post the results to the incident as a comment

The Logic App uses a system-assigned managed identity for connections to both Sentinel and the Graph API.
## Create a incident trigger logic app

1. Create a Logic App.
2. Open the Logic App and navigate to **Settings → Identity**.
3. Enable the system-assigned managed identity.
   ![Picture 1. Turn on System Assigned Managed Identity](/assets/img/2024-11-24-block-entraid-user-playbook/Picture-1-SystemAssigenedMI.png)
5. Edit the Logic App.
6. Add "Microsoft Sentinel Incident" as a trigger.
7. Retrieve account entities from the incident trigger.
8. Initialize empty arrays for `ErrorArray` and `SuccessArray`.
   ![Image 2. Trigger and empty arrays](/assets/img/2024-11-24-block-entraid-user-playbook/Picture-2-trigger-emptyarrays.png)
7. Add HTTP Action with following properties
	- URI: `https://graph.microsoft.com/v1.0/users/`(Append "`Accounts Microsoft Entra ID user ID`" from dynamic content.)
	- Method: `PATCH`
	- BODY:
        ```json
        {  
        "accountEnabled": false
        }
        ```
	**Note**: Ensure you add `false` as an expression!
- Advanced Parameters for Authentication:
	- Authentication type: `Managed Identity`
	- Managed Identity: `System-assigned managed identity`
	- Audience: `https://graph.microsoft.com`
Here's an example of an HTTP request action:  
![Picture 3. HTTP block user action](/assets/img/2024-11-24-block-entraid-user-playbook/Picture-3-HTTP.png)


Don’t worry about the "For Each" loop. Since `Accounts Microsoft Entra ID user ID` is an array, the Logic App will automatically create a "For Each" loop to iterate through it.

At this point, your setup should look something like this:
![Picture 4. Start of an for each loop with http request](/assets/img/2024-11-24-block-entraid-user-playbook/Picture-4-foreach-start.png)

8. Next, create an `Append to Array` action, and append the following text to the `SuccessArray`:
   ```
   User <b>ACCOUNT NAME HERE</b> was successfully disabled.<br />
   ```
    Ensure you add the **Account Name** property from dynamic content between the `<b></b>` tags (replacing the `ACCOUNT NAME HERE` text).

9. Add a parallel branch between the `HTTP` action and the `Append to Array variable` action. In this parallel branch, include a `Parse JSON` action with the following schema:
    ```json
    {
        "type": "object",
        "properties": {
            "error": {
                "type": "object",
                "properties": {
                    "code": {
                        "type": "string"
                    },
                    "message": {
                        "type": "string"
                    },
                    "innerError": {
                        "type": "object",
                        "properties": {
                            "date": {
                                "type": "string"
                            },
                            "request-id": {
                                "type": "string"
                            },
                            "client-request-id": {
                                "type": "string"
                            }
                        }
                    }
                }
            }
        }
    }
    ```

    Use the HTTP action's body as the content for the `Parse JSON` action.

    Your `Parse JSON` action should look like this:  
![Picture 5. Parse JSON action](/assets/img/2024-11-24-block-entraid-user-playbook/Picture-5-ParseJSON.png)
10. Open the `Parse JSON` action and navigate to the **Settings** tab.
11. Configure the **Run After** setting: expand the HTTP action and select only **Has failed**. This allows you to handle HTTP failures.  
    ![Picture 6. Parse json Run after](/assets/img/2024-11-24-block-entraid-user-playbook/Picture-6-ParseJSON-RunAfter.png)
12. Add an "Append to Array" action after the `Parse JSON` action.
13. Update the `ErrorArray` with the following value:
    ```text
    User <b>ACCOUNT NAME HERE </b> was <b><em>not disabled</em></b>. <br /><b>Error message:BODY MESSAGE HERE</b> <br />
    ```
    Ensure you replace `ACCOUNT NAME HERE` with the **Account Name** from dynamic content and update the error message by replacing `BODY MESSAGE HERE` with the **Body message** from the `Parse JSON` action in dynamic content.

    The entire "For Each" loop (including the `Append to Array Variable - ErrorArray`) should look like this:  
    ![Picture 7. the whole for each loop](/assets/img/2024-11-24-block-entraid-user-playbook/Picture-7-ForEach.png)

14. Finally, add the `Add comment to incident (V3)` action outside of the "**For Each**" loop as the last action in the Logic App.
15. Provide the **Incident ARM ID** to the comment action.
16. Add the following expressions to the comment section:
    ```js
    if(empty(variables('SuccessArray')),'',join(variables('SuccessArray'),','))

    if(empty(variables('ErrorArray')),'',join(variables('ErrorArray'),','))
    ```

    **Explanation of the expressions:**  
    The `if()` expression is used to check whether the arrays are empty by leveraging the `empty()` function:
    - If the array is empty, display nothing (`''`).
    - If the array contains data, use the `join()` function to concatenate the array elements into a string, with each value separated by a comma.

    Here’s an example of how the comment action should look:  
    ![Picture 8. Comment action](/assets/img/2024-11-24-block-entraid-user-playbook/Picture-8-Comment.png)

    Basically the logic app should look like this  
    ![Picture 9. Full logic app](/assets/img/2024-11-24-block-entraid-user-playbook/Picture-9-FullLogicApp.png)

17. Save the Logic App.
18. Assign permissions to the managed identity (see the PowerShell commands below).
19. Assign the **Microsoft Sentinel Responder** role to the managed identity, if applicable.
20. Test the playbook from Sentinel.
### Assign Permission to Managed identity

Here’s a PowerShell script that assigns the required permissions to the managed identity for the HTTP call.  
The script utilizes the **MgGraph** PowerShell module.

```powershell
# Add the correct 'Object (principal) ID' for the Managed Identity
$ObjectId = "4ef75f41-6964-48df-8a86-ada069b0f5b2"

# Add the correct Graph scopes to grant (multiple scopes)
$graphScopes = @(
    "User.ManageIdentities.All", 
    "User.EnableDisableAccount.All"
)

# Connect to Microsoft Graph
Connect-MgGraph -Scope AppRoleAssignment.ReadWrite.All

# Get the Graph Service Principal
$graph = Get-MgServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'"

# Loop through each scope and assign the role
foreach ($graphScope in $graphScopes) {
    # Find the corresponding AppRole for the current scope
    $graphAppRole = $graph.AppRoles | Where-Object { $_.Value -eq $graphScope }

    if ($graphAppRole) {
        # Prepare the AppRole Assignment
        $appRoleAssignment = @{
            "principalId" = $ObjectId
            "resourceId"  = $graph.Id
            "appRoleId"   = $graphAppRole.Id
        }

        # Assign the role
        New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $ObjectId -BodyParameter $appRoleAssignment | Format-List
        Write-Host "Assigned $graphScope to Managed Identity $ObjectId"
    } else {
        Write-Warning "AppRole for scope '$graphScope' not found."
    }
}

```
## Future Improvements

If desired, you can easily create alert triggers and entity playbooks using this guide. To create an alert trigger, simply copy & change the Logic App trigger from **incident** to **alert**.  
For an entity trigger, the Logic App only requires one trigger (**entity**) and one action (**HTTP**).

If you also wish, you can create an unblock playbook by copying the block Logic App and changing `accountEnabled` to `true` (remember to use `true` as an expression!).
## Wrapping Up

This is a simple demo for blocking a user from signing in to their Entra ID Account. I developed this using a test analytic rule (which creates incidents for testing purporses), and this rule provides the user object ID that I can use in the HTTP call. You wouldn't want this to fail in the case of a true positive incident, so make sure you test it with multiple use cases to determine how to retrieve the user object ID. In the worst case, you may need to adjust all analytic rules that provide account entities or define separate logic in the Logic App to correctly retrieve the user's object ID.

Also, keep in mind the permissions. I used the least privileged permissions available, but depending on your use case, they may not be sufficient for certain roles. Refer to the [update user](https://learn.microsoft.com/en-us/graph/api/user-update?view=graph-rest-1.0&tabs=http) documentation and the [who can perform sensitive actions](https://learn.microsoft.com/en-us/graph/api/resources/users?view=graph-rest-1.0#who-can-perform-sensitive-actions) table for more information.

Here’s the link to my [Block user Bicep templates on GitHub](https://github.com/apaivinen/sentinel-playbooks/tree/main/Block-EntraIDUser), and in the next post, I will explain an alternative way to block users.

I hope you found this helpful!

-Anssi

## Sources
- [Graph API, Update user](https://learn.microsoft.com/en-us/graph/api/user-update?view=graph-rest-1.0&tabs=http)
- [Logic app expression, If()](https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference#if)
- [Logic app expression, Empty()](https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference#empty)
- [Logic app expression, Join()](https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference#join)
- [Powershell, MgGraph module](https://learn.microsoft.com/en-us/powershell/microsoftgraph/installation?view=graph-powershell-1.0)