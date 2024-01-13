---
title: Pairing Ikea Fyrtur blinds with Home Assistant
date: 2023-03-22 5:00:00
categories: [Home Assistant, Device Configuration]
tags: [Home Assistant,Device Configuration,Home Automation,Blueprint,Zigbee,deCONZ,IOT,Automation]
layout: post
toc: true
comments: false
math: false
mermaid: false
---
I've been using Ikea Fyrtur block-oud roller blinds for six months for couple of windows and generally I'm pleased with them. For rest of  the windows I've been just adjusting blinds manually and sometimes it has felt like this should be automated for some windows. Especially on those windows which are critical for privacy in the evenings etc. 

I just ordered and got two more Fyrtur blinds and I remembered it wasn't as straight forward to get them to work on Home Assistant as some other zigbee devices would be so I went to check my documentation about the first time installing these blinds.

I noticed that even the six month old documentation wasn't really up to date with current Fyrtur blind version so here's the guide for 2023

## Current setup
Home Assistant (2023.3.5) running on Intel NUC (nuc7pjyh)  
Zigbee stick is [Conbee II](https://www.phoscon.de/en/conbee2) and Home Assistant is using [deCONZ integration](https://www.home-assistant.io/integrations/deconz) for communicating with Zigbee devices that are connected to the Conbee gateway.  

I don't have the official Ikea bridge. Or any other bridge. Everything is handled by Conbee & Home Assistant.

# Fyrtur block-out roller blind pairing & configuration
## Pairing Fyrtur blinds

Start by inserting fully charged battery to the blinds.

Pairing these blinds require that you start searching for new lights in Phoscon web app. So go to deCONZ, log in to phoscon, navigate to lights and press `Add new lights`.

When I started the pairing process by pressing two buttons on the blinds the search couldn't find blinds at all. I didn't have this problem six months ago.

After a while of debugging the situation I noticed that deCONZ Application was able to see new device but Phoscon was not able to pair it.   
So this is what I saw from deCONZ application, a new node named `FYRTUR block-out roller blind`.  

![Picture 1. FYRTUR block-out roller blind node in deCONZ application](/assets/img/2023-03-22-ikea-fyrtur-blinds-with-home-assistant/1-NewBlindNode.png)

I right clicked on the node and pressed `Read Simple descriptors` which triggered node name change. For me it appeared as "Window covering device 14". Apparently I've tried to pair this couple of times U+1F605   


![Picture 2. Read simple descriptors from the blind](/assets/img/2023-03-22-ikea-fyrtur-blinds-with-home-assistant/2-BlindReadSimpleDescriptors.png)

And when I switched to Phoscon UI and saw the search was successful. Now I had `Window covering device 14` in list of devices.   
Next step is to rename device for easier recognition in Home Assistant. I'm renaming it to Terrace door blind.

![Picture 3. Phoscon list of devices](/assets/img/2023-03-22-ikea-fyrtur-blinds-with-home-assistant/3-ListOfDevices.png)

## Fyrtur blind configurations 
The first time I paired Fyrtur blinds six months ago I couldn't get accurate information about blind position or battery information. I'm assuming these new blinds have the same issue and this can be fixed with fairly simple configurations through deCONZ App.  

So let's get back to deCONZ app and configure the blinds.   

Select Panels from top menu and add Bind Dropbox to your selection

![Picture 4. Adding bind dropbox](/assets/img/2023-03-22-ikea-fyrtur-blinds-with-home-assistant/4-AddBindDropbox.png)

Now find your ConBee gateway node which for me is named to `Configuration tool 1`. Also find your renamed blind node (Mine is `Terrace door blind`).  
Expand both nodes, blind and gateway by selectin right circle as active.

![Picture 5. Blind & Gateway nodes expanded](/assets/img/2023-03-22-ikea-fyrtur-blinds-with-home-assistant/5-ExpandedNodes.png)

Next do the following
1. Open `bind dropbox` tab
2. Drag & Drop `Window Covering` to `source` from blind node
3. Drag & Drop `Power Configuration` to `destination` from gateway node
4. Press `Bind`

![Picture 6. Binding mappings](/assets/img/2023-03-22-ikea-fyrtur-blinds-with-home-assistant/6-BindingMappings.png)

Next select blind node as active and change to `Cluster infor` tab.  
Scroll down the attribute list and find `Current Position Lift Percentage`. It should be under `Window Covering Information`

![Picture 7. Window covering cluster information](/assets/img/2023-03-22-ikea-fyrtur-blinds-with-home-assistant/7-ClusterInfo.png)

Double-click `Current Position Lift Percentage` and enter following values to `Reporting Configuration`.
Min report interval = 1
Max report interval = 300
Reportable Change = 1

And press `Write Config` button to save the values.

![Picture 8. Attribute editor for position reporting intervals](/assets/img/2023-03-22-ikea-fyrtur-blinds-with-home-assistant/8-AttributeEditor.png)

Now you should be done.

Go to home assistant and you are able to use the blinds and track blind position & battery.
In my case I had to restart HA to get entities updated in to Home Assistant and I could start using them in  automations.

## Pairing the TRADFRI Open/close remote

I actually never tested the remotes which I got with the first blinds. Now I decided to take two remotes in use to get even more control to home automations.  

With the provided TRADFRI open/close remote I had the same problem as with the blinds that Phoscon app could not find it before I right clicked on the node and selected "Read simple descriptors".  

![Picture 9. Remote node read simple descriptors](/assets/img/2023-03-22-ikea-fyrtur-blinds-with-home-assistant/9-remoteReadSimpleDescriptors.png)

No other configurations needed.  
I know it's not reporting battery information as it should but from brief investigation I found out a post where people says it's a common problem and decided it doesn't matter for the moment that I can't track battery. Probably need to fix it at some point in the future.  

Also with this one I also needed to restart Home Assistant so I could use the remote in autmations.

I briefly had an issue that I wasn't able to get any button up presses to pass to home assistant. Only down presses and long holds was registering. I have no idea what was going on but after I took the battery out it started to work as it should.

I also created a modification of existing blueprint to have "easy to use" one automation for this open/close remote. Here's a [link to my github](https://github.com/apaivinen/homeassistant/blob/master/blueprints/automation/homeassistant/ikea_fyrtur_2_button_remote.yaml) to the blueprint.

![Picture 10. Blueprint overview](/assets/img/2023-03-22-ikea-fyrtur-blinds-with-home-assistant/10-blueprint.png)


# Closing thoughts

In the end the major problems was quite easy to fix AFTER YOU FOUND OUT WHAT'S THE REAL ISSUE. In total this process for the first blind took me today over 2 hours. The second blind was paired, configured and added to automations with in 15 minutes.

There were couple of helpful community posts and one major blog post which really helped a lot. I also found out that there's a [modification out there to make these blinds more silent](https://riesinger.dev/posts/silencing-fyrtur-blinds-with-custom-firmware/). Maybe this is something what I need to check out in the future also.

# Sources

More information about attribute reporting & source and destination mapping [https://github.com/dresden-elektronik/deconz-rest-plugin/issues/1121#issuecomment-524617659](https://github.com/dresden-elektronik/deconz-rest-plugin/issues/1121#issuecomment-524617659)  

Awesome post describing the whole pairing & configuration process [https://riesinger.dev/posts/ikea-fyrtur-homeassistant-deconz/](https://riesinger.dev/posts/ikea-fyrtur-homeassistant-deconz/)  

Original blueprint made for ZHA (not deCONZ) [https://community.home-assistant.io/t/zha-ikea-open-close-switch-for-covers-e-g-kadrilj-fyrtur/258904](https://community.home-assistant.io/t/zha-ikea-open-close-switch-for-covers-e-g-kadrilj-fyrtur/258904)  

bonus: Ikea Fyrtur mods [https://github.com/mjuhanne](https://github.com/mjuhanne)
