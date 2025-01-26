---
title: Inactive External Users Report Logic App
date: 2025-01-26 5:00:00
categories: [Entra ID, Logic App]
tags: [Azure,Guide,Security,Entra ID, Reporting]
layout: post
toc: true
comments: false
math: false
mermaid: false
published: true
---
This one is an old logic app which I have made somewhere between 2019-2020 but more modern & simplified version of it.

Basically this will create a report of all guest users which have not been active for 1 year. The report is generated as HTML table and posted to a teams channel message. 

## Solution Overview

The logic is fairly simple:

1. **Get all guest users and include signInActivity poperty.**
2. **Checks if guest user had last succesfull sign in over 365 days ago,  if yes, append the information to a string variable as an HTML table row.**
3. **At the end, send the HTML table via Teams channel message.**

Let's get started with the guide.
## Requirements
- Azure subscription
- Resource group
- Microsoft Graph PowerShell module
- Automation/service user account with Teams license

## Create a logic app
- Start by creating a new **Logic App** in your Azure subscription.
- Select the resource group and region of your choice.
- Name your Logic App according to your naming convention. For this example, we'll use **LA-Entra-Report-Inactive-Guests**.

After the Logic App has been created, enable the Managed Identity:

1. Open the Logic App in the Azure portal.
2. From the left navigation menu, go to **Settings** → **Identity**.
3. Enable the **System-assigned** identity.
4. Take note of the **Object (Principal) ID** because you'll need it in the next step to assign the necessary permissions.
### Assign Permissions to Managed Identity

For this automation, we need **AuditLog.Read.All** and **User.Read.All** permissions.  
Optionally you could assing **User.ReadWrite.All** If you want to use this automation for deleting the inactive users. Refer to the official documentation on [Microsoft Learn](https://learn.microsoft.com/) for more details.  
- [List users - Microsoft Learn](https://learn.microsoft.com/en-us/graph/api/user-list?view=graph-rest-1.0&tabs=http)
- [Delete a user - Microsoft Learn](https://learn.microsoft.com/en-us/graph/api/user-delete?view=graph-rest-1.0&tabs=http)

```powershell
# Add the correct 'Object (principal) ID' for the Managed Identity
$ObjectId = "MIObjectID"

# Add the correct Graph scopes to grant (multiple scopes)
$graphScopes = @(
    "AuditLog.Read.All", 
    "User.Read.All"
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

## Create the Logic

Open your Logic App in **Edit** mode, and let's begin building the workflow.

### Step 1: Variables Setup

1. Select a trigger, schedule works for this reportin automation
2. **Create a "Get past time" action**    
    - Set your desired timeframe to look back on. For this example, I'll use **12 months**. This interval represents what is the treshold for considering as inactive user.
3. **Initialize a string variable**    
    - Name the variable **tableRows**.
    - This variable will be used to store the HTML table row data for the list of inactive users.
4. **Initialize an boolean variable**    
    - Name the variable **breakLoop**.
    - This variable will track when HTTP - Get Guest users have succesfully got the results (HTTP Code: 200).

At this point, your Logic App should look like this:

![Picture 1. Variables](/assets/img/2025-01-26-inactive-external-users-report-logic-app/1-Variables.png)

### Step 2: Get guest users

This section is more complicated than it should be due to instability or issues with the API.  
For some reason, the GET request to `https://graph.microsoft.com/v1.0/users` results in the error **"Insufficient privileges to complete the operation,"** even though the app has the required permissions. You can find more details about this error at the end of the post.  
To work around this issue, we are using a **Do Until** loop to repeatedly send the HTTP request until it succeeds or reaches a maximum of 60 attempts.

1. Create a Until loop
2. Loop until **breakLoop** variable is equal to `true`  
![Picture 2. Until.png](/assets/img/2025-01-26-inactive-external-users-report-logic-app/2-until.png) 

3. Create HTTP action with following configurations
	- URI: `https://graph.microsoft.com/v1.0/users` 
	- Method: `GET`
	- Headers: `ConsistencyLevel`:`Eventual`
	- Queries:
		- `$filter`:`userType eq 'Guest'` 
		- `$select`:`displayName,userPrincipalName,signInActivity`
	- Advanced parameters:
		- Authentication
			- Authentication Type: `Managed Identity`
			- Managed Identity: `LA-Entra-Report-Inactive-Guests`
			- Audience: `https://graph.microsoft.com`  
![Picture 3. HTTP - Get Guest users.png](/assets/img/2025-01-26-inactive-external-users-report-logic-app/3-http.png)  
4. Create Set Variable and select breakLoop for variable and true for the value.
5. Create a paraller branch in between of HTTP request and Set variable.
6. Add Delay action to paraller branch with 5 seconds delay.
7. Go to Settings tab for Delay action.
8. **Untick** Is successful and tick **rest** of the options.

Your Do Until should be similar to this:  
![Picture 4. Do until done.png](/assets/img/2025-01-26-inactive-external-users-report-logic-app/4-until-done.png)

Outside of do-until add Parse JSON action where you add body from HTTP - Get Guest users and following schema:

```json
{
    "type": "object",
    "properties": {
        "statusCode": {
            "type": "integer"
        },
        "headers": {
            "type": "object",
            "properties": {
                "Cache-Control": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "Transfer-Encoding": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "Vary": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "Strict-Transport-Security": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "request-id": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "client-request-id": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "x-ms-ags-diagnostic": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "x-ms-resource-unit": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "OData-Version": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "Date": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "Content-Type": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "Content-Length": {
                    "type": [
                        "string",
                        "null"
                    ]
                }
            }
        },
        "body": {
            "type": "object",
            "properties": {
                "@@odata.context": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "value": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "displayName": {
                                "type": [
                                    "string",
                                    "null"
                                ]
                            },
                            "userPrincipalName": {
                                "type": [
                                    "string",
                                    "null"
                                ]
                            },
                            "id": {
                                "type": [
                                    "string",
                                    "null"
                                ]
                            },
                            "signInActivity": {
                                "type": "object",
                                "properties": {
                                    "lastSignInDateTime": {
                                        "type": [
                                            "string",
                                            "null"
                                        ]
                                    },
                                    "lastSignInRequestId": {
                                        "type": [
                                            "string",
                                            "null"
                                        ]
                                    },
                                    "lastNonInteractiveSignInDateTime": {
                                        "type": [
                                            "string",
                                            "null"
                                        ]
                                    },
                                    "lastNonInteractiveSignInRequestId": {
                                        "type": [
                                            "string",
                                            "null"
                                        ]
                                    },
                                    "lastSuccessfulSignInDateTime": {
                                        "type": [
                                            "string",
                                            "null"
                                        ]
                                    },
                                    "lastSuccessfulSignInRequestId": {
                                        "type": [
                                            "string",
                                            "null"
                                        ]
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

### Step 3: Check for inactive users

1. Create a condition where you compare `lastSignInDateTime` property from Parse JSON action. The condition is `lastSignInDateTime` is less than `Past Time` action. Technically `lastSignInDateTime` is `items('For_each')?['signInActivity']?['lastSignInDateTime']`. The for each is created automatically when the **lastSignInDateTime** is selected.  
![Picture 5. Condition.png](/assets/img/2025-01-26-inactive-external-users-report-logic-app/5-condition.png)  

2. In the true section create Convert time zone action with following properties
	- Base Time: lastSignInDateTime property from Parse JSON (`item()?['signInActivity']?['lastSignInDateTime']`)
	- Select Time Zone: (UTC) Coordinated Universal Time
	- Destination Time Zone: Your Time Zone
	- Time Unit: Your preferred time unit, I added mine as `dd.MM.yyyy HH:mm`  
![Picture 6. Convert time zone.png](/assets/img/2025-01-26-inactive-external-users-report-logic-app/6-convert-time-zone.png)  
3. Create Append to string variable which appends to tableRows variable with following content:

```html
<tr>
	<td>item()?['displayName']</td>
	<td>item()?['userPrincipalName']</td>
	<td>item()?['id']</td>
	<td>body('Convert_time_zone')</td>
</tr>
```

First three are from Parse JSON action and the last one is from Convert Time Zone action you created in the previous step.  
![Picture 7. append to string.png](/assets/img/2025-01-26-inactive-external-users-report-logic-app/7-append-to-string.png)


Your logic for checking inactive guest users should look similar to this:  
![Picture 8. check inactive users logic.png](/assets/img/2025-01-26-inactive-external-users-report-logic-app/8-check-logic.png)

### Step 4: The message

Now there's two more actions left.

Second to last action is Compose with following content:

```html
<table style="width:100%; border-collapse: collapse; border: 1px solid #ddd; font-family: Arial, sans-serif;">  
	<thead>  
		<tr style="background-color: #f2f2f2; color: #333;">  
		<th style="border: 1px solid #ddd; padding: 8px; text-align: left;">Display Name</th>  
		<th style="border: 1px solid #ddd; padding: 8px; text-align: left;">User Principal Name</th>  
		<th style="border: 1px solid #ddd; padding: 8px; text-align: left;">ID</th>  
		<th style="border: 1px solid #ddd; padding: 8px; text-align: left;">Last Interractive Sign-In</th>  
		</tr>  
	</thead>  
	<tbody>  
	    variables('tableRows')
	</tbody>  
</table>
```

 Make sure you added tableRows variable in between of tbody tags.  
![Picture 9. Compose.png](/assets/img/2025-01-26-inactive-external-users-report-logic-app/9-compose.png)  
For the last action, create a "Post Message in a Chat or Channel"
	- **Post As:** User
	- **Post In:** Channel
	- **Team:** Select your team
	- **Channel:** Select your channel
	- **Message:** Use the **Outputs** of your previously created **Compose** action.
	- You can optionally add a subject for the message under **Advanced Parameters** if needed.  

![Picture 10. Post Message in a chat or channel.png](/assets/img/2025-01-26-inactive-external-users-report-logic-app/10-post-message.png)

And finally, that's it! Your notification logic is now complete. You should receive Teams message when there are inactive guest users.

Here's an example of teams message:  
![Picture 11. Example message.png](/assets/img/2025-01-26-inactive-external-users-report-logic-app/11-example-message.png)


### Overview of full logic app

![[Picture 12. overview-1.png]](/assets/img/2025-01-26-inactive-external-users-report-logic-app/12-overview-1.png)  
![[Picture 13. overview-2.png]](/assets/img/2025-01-26-inactive-external-users-report-logic-app/13-overview-2.png)  
![[Picture 14. overview-3.png]](/assets/img/2025-01-26-inactive-external-users-report-logic-app/14-overview-3.png)  

## Possible Modifications
You could add an HTTP request to delete users within the checking logic (where the "Append to string" action is). However, I have personally created a separate Logic App for deleting users, which you can [find on my GitHub](https://github.com/apaivinen/Entra-ID-Logic-Apps/tree/main/Report_inactive_external_users).
## Closing thoughts

This Logic App provides an alternative approach to checking inactive users compared to the PowerShell scripts I have commonly seen in customer environments.

Developing this Logic App was a bit challenging. During the process, I encountered severe authorization issues, which required creating workarounds to handle random "unauthorized" request errors. Other than that, the overall implementation was fairly straightforward.

This post serves as a walkthrough of the reporting Logic App. I have created a separate Logic App specifically for deleting inactive users. Both Logic Apps, along with deployment scripts and Bicep templates, are available on my GitHub, complete with detailed instructions on how to deploy them to your environment.

This post is kind of a walkthrough of report logic app. I made delete inactive user logic to another logic app on the side. Both logic apps can be found on my github with deployment script and bicep templates with detailed descriptions how to deploy them to your environment. [GitHub](https://github.com/apaivinen/Entra-ID-Logic-Apps/tree/main/Report_inactive_external_users)

Hopefully this helps you manage your guest users.

-Anssi

## Sources
- [List users - Microsoft Learn](https://learn.microsoft.com/en-us/graph/api/user-list?view=graph-rest-1.0&tabs=http)  
- [Delete a user - Microsoft Learn](https://learn.microsoft.com/en-us/graph/api/user-delete?view=graph-rest-1.0&tabs=http)  
- [How to manage inactive user accounts - Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-manage-inactive-user-accounts#how-to-detect-inactive-user-accounts)  
- [Github, apaivinen/Entra-ID-Logic-Apps](https://github.com/apaivinen/Entra-ID-Logic-Apps/tree/main/Report_inactive_external_users)  

## Extra: Unauthorized Error in Detail for Those Who Are Wondering

I have assigned the following permissions to my user-managed identity:
- **User.ReadWrite.All** (I used this with the delete Logic App as well, which is why it includes ReadWrite access)
- **AuditLog.Read.All**

It does not matter whether the managed identity is user-assigned or system-assigned.

I also tried adding the **User Administrator** role to the managed identity. During development, I went overboard and even assigned the **Global Administrator** role to see if it would have any effect. No, none at all.

The HTTP request in question:
- URI: `https://graph.microsoft.com/v1.0/users`
- Method: `GET`
- Queries:
	- `$filter`:`userType eq 'Guest'`
	- `$select`:`displayName,userPrincipalName,signInActivity`
- Authentication:
	- The managed identity
	- Audience: `https://graph.microsoft.com`

Despite having the permissions mentioned above, there is a high chance of encountering the following error during the HTTP request:

```json
{
    "statusCode": 403,
    "headers": {
        "Cache-Control": "no-cache",
        "Transfer-Encoding": "chunked",
        "Vary": "Accept-Encoding",
        "Strict-Transport-Security": "max-age=31536000",
        "request-id": "f428ca8a-d83c-44e7-9939-5fb35e669716",
        "client-request-id": "f428ca8a-d83c-44e7-9939-5fb35e669716",
        "x-ms-ags-diagnostic": "{\"ServerInfo\":{\"DataCenter\":\"West Europe\",\"Slice\":\"E\",\"Ring\":\"5\",\"ScaleUnit\":\"010\",\"RoleInstance\":\"AM4PEPF000278F7\"}}",
        "x-ms-resource-unit": "1",
        "Date": "Sun, 26 Jan 2025 18:06:23 GMT",
        "Content-Type": "application/json",
        "Content-Length": "266"
    },
    "body": {
        "error": {
            "code": "Authorization_RequestDenied",
            "message": "Insufficient privileges to complete the operation.",
            "innerError": {
                "date": "2025-01-26T18:06:23",
                "request-id": "f428ca8a-d83c-44e7-9939-5fb35e669716",
                "client-request-id": "f428ca8a-d83c-44e7-9939-5fb35e669716"
            }
        }
    }
}
```

Debugging this issue was a nightmare, as the error appeared randomly. I spent a considerable amount of time searching for information related to the problem but couldn't find any relevant posts or mentions within a reasonable timeframe.

Even assigning the **User Administrator** and **Global Administrator** roles on top of the necessary Graph permissions did not resolve the issue. Eventually, I decided to leave it as is and implemented a **do-until** workaround. It's not the most elegant solution, but it works.