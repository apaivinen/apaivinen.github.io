---
title: Sharepoint Online Organization assets cheatsheet
date: 2023-04-25 5:00:00
categories: [Sharepoint Online, Powershell]
tags: [Sharepoint Online,Powershell,Modern work,Cheatsheet,Guide,Tutorial]
layout: post
toc: true
comments: false
math: false
mermaid: false
---

This is oldie but goldie. From time to time I'm referring to this documentation when implementing organization assets or consulting a co-worker about it. 

# Good to know
- For the organization assets library to appear to a user in PowerPoint on the web, the user must be assigned a license to Office 365 E3 or E5. Users who use the Word, Excel, or PowerPoint desktop app also need Microsoft 365 Apps Version 2002 or later. (The organization assets library is not available in Word on the web or Excel on the web.)
- Allow up to 24 hours for the organization assets library to appear to a user in the desktop apps.
- Users need at least read permissions on the root site for your organization for the organization assets library to appear in the desktop apps.
- All organization asset libraries must be on the same site.
- You need to have` Sharepoint Online Management Shell` installed
- You need to have `Global admin` **OR** `Sharepoint admin` role
- Adding an organization assets library will enable a content delivery network (CDN) for your organization to provide fast and reliable performance for shared assets. You'll be prompted to enable a CDN for each organization asset library you add. 

# Setup
1. Create new site collection (Communication site) named for example `assets` which creates site collection `.../sites/assets`
2. Create new document library called `Templates` to `assets`
3. Create new document library called `Pictures` to `assets`
4. Modify `asset` site permissions
	- Add `Everyone except external users` group to site visitors group
	- Add relevant users to owners / members group
		- Users who are managing Pictures & Templates 
5. Connect to Sharepoint Online Management Shell and run following commands
	1. Pictures: `Add-SPOOrgAssetsLibrary -LibraryUrl <URL> -OrgAssetType ImageDocumentLibrary -CdnType Private` DONT FORGET TO UPDATE `LibraryUrl`
	2. Templates: `Add-SPOOrgAssetsLibrary -LibraryUrl <URL> -OrgAssetType OfficeTemplateLibrary -CdnType Private` DONT FORGET TO UPDATE `LibraryUrl`
6. Upload files to `Template` and `Pictures` folder

For more information about command options and CDN check sources. 
# Sources:
Command docs: [Add-SPOOrgAssetsLibrary (Microsoft.Online.SharePoint.PowerShell) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/sharepoint-online/add-spoorgassetslibrary?view=sharepoint-ps)  
Org asset docs: [Create an organization assets library - SharePoint in Microsoft 365 | Microsoft Learn](https://learn.microsoft.com/en-us/sharepoint/organization-assets-library)
