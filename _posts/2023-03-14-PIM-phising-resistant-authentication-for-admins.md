---
title: PIM & Strong authentication for admins
date: 2023-03-14 5:00:00
categories: [Azure Active Directory, Privileged Identity Management]
tags: [azure active directory,entra,privileged identity management,PIM,mfa,strong authentication,security key,FIDO2,microsoft,learning,tutorial]
layout: post
toc: true
comments: false
math: false
mermaid: false
---

# PIM & Phishing resistant authentication for admins

Microsoft have released to preview conditional access for PIM to enforce phishing resistant MFA request for elevated users. It's quite simple to configure and the benefits are just great. 

Let's pretend you have a security administrator role or global administrator in PIM and users have rights to elevate themself to a admin. What happens if that users account is compromised? Yes you can have an approval process for elevation but that relies on other party to accept the request which isn't smooth experience in day to day operations.

Now with this conditional access you could have  automatically strong authentication to be enforced when users are trying to elevate themselves to a administrator role.

So here's a demo how to configure hardware/security key to a administrator role.  
Keep in mind this feature requires phishing resistant MFA enabled and configured for users.

In Entra go to Protect & Secure --> Conditional Access --> Authentication context. Create a new context

![Picture 1. Add authentication context](/assets/img/2023-03-14-PIM-phising-resistant-authentication-for-admins/1-AddAuthContext.png)

Next step is to create a new conditional access policy.  
Give it a name and select Cloud apps or actions --> Select what this policy applies to: Authentication context and tick the context you just created in last step.

![Picture 2. New conditional access policy, authentication context](/assets/img/2023-03-14-PIM-phising-resistant-authentication-for-admins/2-NewCAPolicyAddContext.png)

Next step is to configure Grant section

![Picture 3. New conditional access policy, Grant](/assets/img/2023-03-14-PIM-phising-resistant-authentication-for-admins/3-NewCAPolicyAddGrant.png)

now when I go to and activate a role there's a banner which says "Addional verification required. Click to continue -->".   
Also note that you cannot activate the role since the buttons are greyed out.  

![Picture 4. Activating PIM Role](/assets/img/2023-03-14-PIM-phising-resistant-authentication-for-admins/4-ActivatingPimRole.png)

When I click the banner I'm redirected to a login screen where I need to use Windows Hello or a security key

![Picture 4. Verify your identity screen](/assets/img/2023-03-14-PIM-phising-resistant-authentication-for-admins/5-verifyIdentity.png)

After you have authenticated via FIDO2 key you'll be redirected back to PIM and you can activate your role as you would do it previously.

This is simple to configure and doesn't bring any overhead to administratorss while enhancing the security aspect.

# More info at
https://learn.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-how-to-change-default-settings