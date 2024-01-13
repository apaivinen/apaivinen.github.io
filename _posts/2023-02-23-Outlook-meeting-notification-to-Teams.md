---
title: Outlook meeting notification to Teams
date: 2023-02-23 5:00:00 
categories: [Power Automate, Personal Automation]
tags: [Power automate, Flow, Automation, Teams, Outlook, Personal Automation, Tutorial, Meeting]
layout: post
toc: true
comments: false
math: false
mermaid: false
---

I wanted to have my outlook client/webmail to be closed but still receive notifications about meetings so here's small Power Automate flow what I created.  
The logic itself is quite simple:
1. Trigger: When an upcoming event is starting soon
	- Look-Ahead time 15 minutes
2. Check if meeting have reminders on
3. Post a message to me reminding about the meeting
4. wait until the meeting is supposed to start
5. Post a new message to me notifying that the meeting starts now

So lets start with the flow actions

# Start of flow
As trigger I'm using When an upcoming event is starting soon (V3) with Look-Ahead time 15mins. 
Right after I'm using `Get My Profile (V2)` action to get my email address dynamically. 
`Subject` variable stores meeting subject since we cannot access it straight from the trigger body inside the condition.

![Picture 1. Start of flow](/assets/img/2023-02-23-Outlook-meeting-notification-to-Teams/1-startOfFlow.png)

# Condition
Next is the condition. I'm using personal calendar events without notification(I don't want notifications about them) so that's why I'm checking if notifications are turned on. 
This is just a simple `Is reminder on` is equal to `true` condition. `Is reminder on` is provided by the trigger output. Also don't forget that you need to add `true` as an expression.  

If reminders are enabled then process the flow further.

![Picture 2. Condition](/assets/img/2023-02-23-Outlook-meeting-notification-to-Teams/2-Condition.png)

# Messages and delay
Both of the messages are pretty simple. Nothing fancy there

![Picture 3. Teams messages](/assets/img/2023-02-23-Outlook-meeting-notification-to-Teams/3-Messages.png)

But the delay in between of messages... that's another story. At first I was thinking since the flow starts 15 minutes before then I could use 15 minutes of delay in between of the messages. 

I was wrong since I'm using free fow lisence which have limitations. The flow can start anywhere between 0-5 minutes late. I was constantly joining atleast 3 minutes late to the meetings which annoyed me a bit.  
After two weeks I got the urge to start fixing the logic.  

At first I was thinking there must be some kind of simple calculation which can give me neccessary information to create dynamic value to delay function. And by simple I mean something like `Meeting start time` minus `Current time` and take that result to delay action. Well that's the end result but it wasn't too simple to create. 

My investigation lead me to use Tick function.

## Expression to get seconds for Delay
Basically I needed to convert both meeting start time and current time to ticks and calculate the difference of them. Result of the calculation needed to be converted to seconds which I could use on Delay Action. [Here's more information about ticks](https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference#ticks)


Full expression:
```
div(sub(ticks(triggerOutputs()?['body/start']),ticks(utcNow())),10000000)
```

Expression breakdown from inside out:  
`Meeting start time` and `utcNow()` are converted to ticks which are inside of [sub-function](https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference#sub) to get substracted value as result.  

And the result of sub-function is inside of [div-function](https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference#div) so ticks can be converted to seconds by divinding ticks by 10000000

Cheatsheet for getting specific values  

| Divided by   | To get  |
| ------------ | ------- |
| 864000000000 | Days    |
| 36000000000  | Hours   |
| 600000000    | Minutes |
| 10000000     | Seconds |


![Picture 4. Delay](/assets/img/2023-02-23-Outlook-meeting-notification-to-Teams/4-Delay.png)

With this nested expression result I could create Count value for Delay function. I'm using seconds since that's the most simple way to achieve correct time for the last meeting notification. I haven't checked this but I have a feeling that minutes wouldn't work unless result is converted to remove decimals which adds more complexity for no reason. 

# End result
And here's the whole flow 

![Picture 5. Whole flow](/assets/img/2023-02-23-Outlook-meeting-notification-to-Teams/5-wholeFlow.png)

This wasn't too complicated. The hardest thing was to get seconds for the delay but mostly this is really basic stuff.
For me this was one of the greatest automation to keep me away from outlook client/webmail to better focus on tasks I'm doing.

Here's a post which was the most fruitful for me to get information how to deal with the ticks  [How to calculate difference between two times in Power Automate (tomriha.com)](https://tomriha.com/how-to-calculate-difference-between-two-times-in-power-automate/)

Here's a [link to my solution on github](https://github.com/apaivinen/powerautomate/tree/main/Outlook%20meeting%20notifications%20to%20teams) for easy import. Don't forget to configure your connections!