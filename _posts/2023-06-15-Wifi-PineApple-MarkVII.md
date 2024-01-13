---
title: WiFi PineApple Mark VII
date: 2023-06-15 5:00:00
categories: [Hacking, Penetration testing]
tags: [WiFi,PenTesting,Penetration Testing,Security,Learning,Hak5,Tools, Hacking]
layout: post
toc: true
comments: false
math: false
mermaid: false
published: false
---
So I got my hands to a WiFi PineApple Mark VII and I've been studying how to use it and it's potential. I have always knew that in the end WiFi is vulnerable but when we are talking about normal consumer setups I really didn't know how bad the situation is before now. 

Lets go through introduction of WiFi PineApple in this post. In separate posts I'll go through the features and how to get started.

>**Disclaimer: Ethical hacking content**  
>This post is intended for educational and demonstration purposes only. 
> 
>Hacking, without proper consent, is illegal and unethical. This post explicitly does not promote or encourage any illegal activities, including unauthorized access to devices, networks, or systems.  
>
>ALWAYS practice ethical hacking techniques within the boundaries of your own networks, devices, and environments or have appropriate permission and written consent from the owner of your target.
>
>__Stay ethical, stay safe!__
{: .prompt-warning }

# What is WiFi PineApple?
WiFi PineApple Mark VII is a wireless penetration testing and auditing tool developed by Hak5. It is designed to assist in assessing the security of wireless networks. The WiFi PineApple leverages various wireless attack techniques to simulate real-world scenarios and identify potential vulnerabilities.

The WiFi Pineapple Mark VII is equipped with multiple radios that support different wireless protocols, such as 2.4GHz and 5GHz Wi-Fi. It has a user-friendly web interface that allows users to configure and control its functionalities.

One of the key features of the WiFi Pineapple Mark VII is its ability to perform "evil twin" attacks. In an evil twin attack, the device creates a rogue access point that mimics a legitimate Wi-Fi network. When unsuspecting users connect to this rogue access point, the WiFi Pineapple can intercept their network traffic, allowing the tester to analyse it for potential security weaknesses.

Additionally, the WiFi Pineapple Mark VII can capture and analyse network traffic, conduct Man-in-the-Middle (MitM) attacks, perform advanced Wi-Fi reconnaissance, and execute various other wireless security assessments. It provides security professionals with insights into network vulnerabilities, weaknesses in encryption protocols, and potential attack vectors that could be exploited by malicious actors.

It is important to note that the WiFi Pineapple Mark VII, like any other penetration testing tool, should only be used with proper authorization and consent. Unlawful and unauthorized usage can lead to legal consequences. Ethical hackers and security professionals typically employ such tools in controlled environments to identify and address security vulnerabilities, helping organizations strengthen their defences against real-world attacks.

## Specs
- 2.4 GHz 802.11 b/g/n (5 GHz/ac with separate MK7AC WiFi Adapter)  
- Single Core MIPS Network SoC
- Three Dedicated Role-based Radios
    _With three high gain antennas_
- USB-C Power/Ethernet Port, USB 2.0 Host Port,  
- Single RGB LED Indicator
- 256 MB RAM, 2 GB EMMC

# Capabilities

When you log in to a WiFi Pineapple you are directed to a dashboard that provides an overview of current status

![Picture 2. Dashboard](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/2-Dashboard.png)

From the left-side menu, we can find all the relevant categories/functions available to us

![Picture 3. Menu](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/3-Menu.png)

## Campaigns
Basically, campaigns allows you to automate your WiFi audits and testing.

There's three different campaign modes and their descriptions
1. Reconnaissance  - Monitor Only
	- Passively monitor client device and access point activity within a defined region of the WiFi Environment
2. Client Device Assessment - Passive
	- Identify client devices susceptible to basic rogue access points or evil twin attacks. Uses passive PineAP mode to mimic access points only upon direct request. Depending on filter configuration, client devices may be allowed to associate with the  WiFi PineApple
3. Client Device Assessment - Active
	- Same as previous client device assessment but uses active PineAP mode to broadcast an SSID pool, mimicking all access points listed. New access points may be dynamically added to the pool. Depending on filter configuration, client devices may be allowed to associate with the WiFi PineApple

The WiFi PineApple stores campaign reports as JSON or HTML, and they can be automatically sent from the device via email or [Cloud C2](https://shop.hak5.org/products/c2) which is Hak5 control service.

Cloud c2 is a [self-hosted control service](https://docs.hak5.org/cloud-c2/getting-started/cloud-c-basics)  for managing Hak5 gear and it's [free for students and enthusiasts](https://shop.hak5.org/products/c2) as non-commercial perpetual License. [Commercial licences](https://shop.hak5.org/products/c2) are available for businesses as one-time purchase.


### Creating a campaign
When creating a new campaign, there's a basic wizard that asks for a campaign name, campaign mode (one of three modes described above), monitoring settings, which include scan duration (30 sec - 15 min) and which radios (2.4 GHz and/or 5 GHz).

Next, there are allow and deny list configurations. The allow list basically allows connections from devices that are listed, while all others will be denied. In my case, I'm using a deny list which blocks all listed clients and allows any other devices.

![Picture 4. Campaign Client Filter](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/4-CampaignClientFilter.png)

The next step is to configure SSID Filter, which follows the same principle as client filter. 

Allow List - Allow only associations for the SSIDs listed. All other SSIDs associations will be blocked.

Deny List - Deny associations for the listed SSIDs. All other SSID associations are accepted.

![Picture 5. Campaign Report Interval](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/5-CampaignReportInterval.png)

Reports are available as JSON or HTML format. I chose HTML for easy readability, but it's great to have JSON file also available as well, since that can be easily accessed via scripts

![Picture 6. Campaign Report Format](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/6-CampaignReportFormat.png)

Last step is to select where to store log files and configure data exfiltration (sending logs by email or use Cloud C2). Default log save location is `/root/loot`. Email reporting is also available but I'm not configuring it for my demos. Additionally, I haven't created Cloud C2 instance yet so I can't use it. Self-hosted Cloud C2 will be a topic for future posts.

![Picture 7. Campaign List](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/7-CampaignList.png)

In the end, a campaign is a bash script that can be edited afterward. However, this is much more technical compared to a previous wizard, as it requires modifying the script itself.

## PineAP
PineAP is your rogue access point. Here you can configure core functionalities of the Wifi PineApple. 

> **Control access with Filters** 
>Limit your engagement by configuring access by filters. Limit to specific clients or SSIDs, or exclude specific clients or SSIDs.
>
>**Impersonate APs**
>Explicitly advertise lists of access points to instigate clients into connecting to previously saved networks
>
>**Open AP**
>Serve a basic, unencrypted Open access point, or automatically impersonate _any_ Open access point requested by a client.
>
>**Evil WPA**
>Serve a new WPA network, or copy an existing WPA network. 
>Capture partial handshakes to crack the WPA keys of unknown networks.
>
>**Evil Enterprise**
>Serve a WPA-Enterprise network with optional key exchange degradation.  Coupled with automatic authorization of all accounts, identify misconfigured enterprise clients and capture credentials.
> 
> source: [PineAP - WiFi Pineapple Mark VII (hak5.org)](https://docs.hak5.org/wifi-pineapple/ui-overview/pineap#pineap-capabilities)

![Picture 8. PineAP](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/8-PineAP.png)


## Recon
Recon gives you an overview of wireless networks within range of the WiFi PineApple. Here's an example of my current wireless landscape with censorship mode turned on to obscure SSID's & Mac addresses.

![Picture 9. Recon](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/9-Recon.png)

By clicking am AP or Client from Access Points or Clients list, a right-side menu pops up and allows me to initiate more actions such as cloning AP, deauthenticating clients, and capturing handshakes.

![Picture 10. Client details](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/10-clientDetails.png)

## Modules & Packages
Modules and Packages extend Wifi PineApple's capabilities with community-build features. Modules are basically Graphical UI for existing tools. 

Here's a list of current out-of-the-box modules available, which allow you to expand WiFi Pineapple's capabilities with useful tools like [nmap](https://nmap.org/) or [Hcxdumptool](https://github.com/ZerBea/hcxdumptool).

![Picture 11. Available Modules](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/11-AvailableModules.png)

Module list from Github: [GitHub - hak5/pineapple-modules: The Official WiFi Pineapple Module Repository for the WiFi Pineapple Mark VII](https://github.com/hak5/pineapple-modules)

Packages are tools and drivers for WiFi Pineapple. Packages often contain command-line utilities that can be accessed via SSH or via the Web Terminal.  

At the time of writing the Wifi PineApple itself can fetch 5784 packages. 

![Picture 12. Manage Packages](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/12-ManagePackages.png)

Installing new modules is really easy since you can simply click the Install button, wait for couple of seconds, and the module you selected is installed to your device. 

![Picture 13. Installed Modules](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/13-InstalledModules.png)

Depending on the module, there might be dependencies that need to be downloaded. However, you will be informed about this when opening the module for the first time. Here's an example of Evil Portals module

![Picture 14. Install Dependencies](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/14-InstallDependencies.png)

We'll go through evil portal capabilities in a future post.

And if there isn't a module or package for your specific needs there's nothing what could prevent you to develop one. Hak5 offers developer documentation for the WiFi PineApple. At the time of writing their [Developer Resources](https://docs.hak5.org/wifi-pineapple/developer-documentation/developer-resources) is under development. Current developer resources are still in their [Github repository](https://hak5.github.io/mk7-docs/).

## Settings

In the settings, you can configure basic things such as time zone, software updates, networking, WiFi configuration, LED configuration.

There's also advanced tab that contains notable features. For this post,  I have already used Censorship Mode, which transforms SSIDs and addresses into random fruits. This is great for presenting live data as you don't have to obscure the screenshots or video. 

![Picture 16. Settings, advanced tab](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/15-SettingsAdvancedTab.png)
I also like the idea of Cartography view, which visualises the recon data. However, in practice, I'm not sure how to feel about it. See the 2D and 3D examples bellow.

### 2D
![Picture 16. Cartography 2D View](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/16-Cartography2d.png)

### 3D
![Picture #. Cartography 3D view](/assets/img/2023-06-15-Wifi-PineApple-MarkVII/17-Cartography3d.png)

Both views can be rotated with mouse, and 3D view rotates along the X and Y axis. It's a neat feature to visualize networks and clients around the PineApple, but there's no additional information shown unless you hover on a dot, and then it only shows the device mac address. Therefore, the cartography view does not replace the basic list view, at least not yet.

So, this is the Hak5 WiFi PineApple Mark VII. In future posts, I'll go through how to use it 

# More info
[Hak5 shop](https://shop.hak5.org/products/wifi-pineapple)
[WiFi PineApple Mark VII documentation](https://docs.hak5.org/wifi-pineapple/)  
[Github, Hak5 official moduyles](https://github.com/hak5/pineapple-modules)  


