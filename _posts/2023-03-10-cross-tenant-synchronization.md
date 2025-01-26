---
title: Cross-tenant synchronization
date: 2023-03-10 5:00:00 
categories: [Entra ID, External identities]
tags: [Entra ID,External identities,Synchronization,Learning,Tutorial,Identities]
layout: post
toc: true
comments: false
math: false
mermaid: false
---

I've been dealing with multi-tenant environments for couple of years now and most of the time there's a need to synchronize only specific users from one tenant to another. For me the most common solution has been so so far Logic App which reads specific Azure Active Directory group members from tenant A and invites them to tenant B as a guest. Second solution has been Access Packages from Entitled management in Azure Active Directory. Earlier this year Microsoft has released [Cross-tenant synchronization](https://learn.microsoft.com/en-us/azure/active-directory/multi-tenant-organizations/cross-tenant-synchronization-overview) as PREVIEW and this eliminates logic apps and access packages in most of my cases. 

Here's the copy & pasted pitch why you care about [cross-tenant synchronization](https://learn.microsoft.com/en-us/azure/active-directory/multi-tenant-organizations/cross-tenant-synchronization-overview)
>Here are the primary goals of cross-tenant synchronization:
>-   Seamless collaboration for a multi-tenant organization
>-   Automate lifecycle management of B2B collaboration users in a multi-tenant organization
>-   Automatically remove B2B accounts when a user leaves the organization
>
>Who should use?
>-   Organizations that own multiple Azure AD tenants and want to streamline intra-organization cross-tenant application access.
>-   Cross-tenant synchronization is **not** currently suitable for use across organizational boundaries.
>
>Benefits
>With cross-tenant synchronization, you can do the following:
>-   Automatically create B2B collaboration users within your organization and provide them access to the applications they need, without creating and maintaining custom scripts.
>-   Improve the user experience and ensure that users can access resources, without receiving an invitation email and having to accept a consent prompt in each tenant.
>-   Automatically update users and remove them when they leave the organization.


# Requirements

Source tenant
-   Azure AD Premium P1 or P2 license. For more information, see [License requirements](https://learn.microsoft.com/en-us/azure/active-directory/multi-tenant-organizations/cross-tenant-synchronization-overview#license-requirements).
	- Using this feature requires Azure AD Premium P1 licenses. Each user who is synchronized with cross-tenant synchronization must have a P1 license in their home/source tenant.
-   [Security Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#security-administrator) role to configure cross-tenant access settings.
-   [Hybrid Identity Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#hybrid-identity-administrator) role to configure cross-tenant synchronization.
-   [Cloud Application Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#cloud-application-administrator) or [Application Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#application-administrator) role to assign users to a configuration and to delete a configuration.

Target tenant
-   Azure AD Premium P1 or P2 license. For more information, see [License requirements](https://learn.microsoft.com/en-us/azure/active-directory/multi-tenant-organizations/cross-tenant-synchronization-overview#license-requirements).
	- Cross-tenant sync relies on the Azure AD External Identities billing model. To understand the external identities licensing model, see [MAU billing model for Azure AD External Identities](https://learn.microsoft.com/en-us/azure/active-directory/external-identities/external-identities-pricing). You will also need at least one Azure AD Premium P1 license in the target tenant to enable auto-redemption.
-   [Security Administrator](https://learn.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#security-administrator) role to configure cross-tenant access settings.

Source: [Cross-tenant sync prerequisites](https://learn.microsoft.com/en-us/azure/active-directory/multi-tenant-organizations/cross-tenant-synchronization-configure#prerequisites)
For more license requirements check [What is a cross-tenant synchronization in Azure Active Directory? (preview) - Microsoft Entra | Microsoft Learn](https://learn.microsoft.com/en-us/azure/active-directory/multi-tenant-organizations/cross-tenant-synchronization-overview#license-requirements)

You also need to know which members need to be synchronized from source tenant to target tenant which means you need a A-AD group with users.  
And you need tenant IDs of both tenants for creating configurations.  

# Scenario
I have a Contoso demo tenant where I have only specific users which I want to be synchronized to my main developer tenant.  
Synchronized members need to go to specific group in my dev tenant.

So basically I need to configure synchronization so users are sync'ed as members and create a dynamic group in target tenant to pickup those users.

# Configuring tenants
## Prepare target tenant
First lets enable target tenant to accept synchronized members.  
Login to [Entra](https://entra.microsoft.com) and head to External Identities -> Cross-Tenant Access Settings.  
In Organizational Settings tab select Add organization  

![Picture 1. start adding organization](/assets/img/2023-03-10-cross-tenant-synchronization/1-ConfigureTargetTenant.png)

Here you need to input source tenant ID or Domain name. I'm going to use Tenant ID from overview page of Azure Active Directory in Entra. 

![Picture 2. Add organization](/assets/img/2023-03-10-cross-tenant-synchronization/2-AddOrganization.png)

When you press Add your tenant should have appeared on the list bellow.  
Then you need to configure Inbound access to allow users to be syncronized from source tenant.  
Inbound access column press "Inherited from default"  

![Picture 3. Access inbound settings](/assets/img/2023-03-10-cross-tenant-synchronization/3-OrganizationSettings.png)

And go to Cross-teant sync (preview) tab & add tick to "Allow users sync into this tenant".   

![Picture 4. Inbound Cross-tenant sync settings](/assets/img/2023-03-10-cross-tenant-synchronization/4-AllowCrossTenantSync.png)

Next I'm going to enable couple of trust settings to get seamless synchronization from users point of view.  
In Trust Settings tab (Inbound access settings) select Customized settings and tick "Trust multifactor authentication from Azure AD tenants" so users don't have to enroll new MFA for target tenant.  

Also I'm suppressing consent promts for users to get seamless experience.  

![Picture 5. Inbound trust settings](/assets/img/2023-03-10-cross-tenant-synchronization/5-TrustSettings.png)

Next step is to configure source tenant.

## Configuring source tenant
In source tenant at Entra create a group (or use existing group) which contains users you want to synchronize to target tenant.

Now you need to enable invitation redemption in source tenant also.  
In Entra go to External Identities --> Cross-tenant access settings --> Organization settings tab. 

Add a new organization and enter target tenant ID to the dialog box. After you press save go to Outbound access settings of the target tenant you just added.

Select Trust settings tab and enable Suppress consent prompts...  and press save  

![Picture 6. Outbound trust settings](/assets/img/2023-03-10-cross-tenant-synchronization/6-SourceTenant-outboundtrust.png))

Next in Cross-tenant synchronization (Preview) and select Configurations tab. In Configurations tab press "New configuration".

![Picture 7. Add new configuration](/assets/img/2023-03-10-cross-tenant-synchronization/7-NewConfiguration.png)

Give a name to configurations and press "Create"

![Picture 8. New configuration](/assets/img/2023-03-10-cross-tenant-synchronization/8-NewConfigurationScreen.png)

After configuration is created you should see following overview

![Picture 9. Configuration overview](/assets/img/2023-03-10-cross-tenant-synchronization/9-ConfigurationOverview.png)

If you are not redirected to your configuration overview you can find it from Cross-tenant synchronization (preview) --> Configurations tab.

Select Provisioning to open provisioning settings.  
Use automatic as provisioning mode for automatic synchronization.  
Currently Authentication method is pre-filled with Cross tenant synchronization policy.  
Fill the tenant id with your target tenant ID and press Test Connection.   
When test is successful press Save

![Picture 10. Testing and saving Provisioning connection](/assets/img/2023-03-10-cross-tenant-synchronization/10-ProvisionSettings.png)

After saving provisioning settings you have Mappings & Settings available where you can configure custom user attribute mapping, notification from failures and accidental deletion prevetion.  
I'm going with default settings but keep in mind you could do some attribute mapping if you wanted to.  

![Picture 11. Provisioning settings](/assets/img/2023-03-10-cross-tenant-synchronization/11-ProvisioningMappinsSettings.png)

Next step is to select a group which content you want to synchronize to target tenant.  
Go to Users and groups uner Manage, select Add user/group and add the group you have decided to synchronize.  

![Picture 12. Add group which contains members to be synchronized](/assets/img/2023-03-10-cross-tenant-synchronization/12-addGroup.png)

Last step in source tenant is to go to Overview tab and press "Start provisioning".

![Picture 13. Start provisioning](/assets/img/2023-03-10-cross-tenant-synchronization/13-startProvisioning.png)

Now you should have synchronization up and running! 

Keep in mind provisioning is on fixed interval of 40 minutes.  
You can also press Provision on demand and select users to synchronize them manually.

# Dynamic group for synchronized users in target tenant

In target tenant I want to use dynamic group to gather up every synchronized users from my source tenant.  
For this demonstration I'm just using email domain to create a rule to dynamic group (`(user.mail -contains "@xxxxxxxxxx.OnMicrosoft.com")`). 

And here are the results.  
Top picture is a group from my source tenant and bottom picture is a dynamic group from my target tenant.  

![Picture 14. Content of both tenants sync groups](/assets/img/2023-03-10-cross-tenant-synchronization/14-GroupsMembers.png)


# More info & sources 

[What is a multi-tenant organization in Azure Active Directory? - Microsoft Entra | Microsoft Learn](https://learn.microsoft.com/en-us/azure/active-directory/multi-tenant-organizations/overview)  
[What is a cross-tenant synchronization in Azure Active Directory? (preview) - Microsoft Entra | Microsoft Learn](https://learn.microsoft.com/en-us/azure/active-directory/multi-tenant-organizations/cross-tenant-synchronization-overview)  
[Configure cross-tenant synchronization (preview) from portal- Microsoft Entra | Microsoft Learn](https://learn.microsoft.com/en-us/azure/active-directory/multi-tenant-organizations/cross-tenant-synchronization-configure)  
Edit: Here's awesome video from John Savill explaining more in depth about Cross-tenant synchronization: [Azure AD Cross-Tenant Sync](https://www.youtube.com/watch?v=z0J5kteqUVQ)
