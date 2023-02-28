---
title: Email summary to Teams
date: 2023-01-17 5:00:00 
categories: [power automate, personal automation]
tags: [power automate, Flow, automation, teams, outlook, personal automation, tutorial]
layout: post
toc: true
comments: false
math: false
mermaid: false
---

As part of my new less uninterrupted workflow mental mode I wanted to keep my outlook closed as much as possible. In ideal case I open my email just once a day and don't mind too much about it later.  
There's situations where I might get "urgent" emails which require attention during the same day. If the matter is really urgent then it won't arrive by email. For those messages which would need attention before I stop working for the day I created a flow which gives me email summary of unread emails arrived on the same day before 13:00.

So here's what I have done

# First part of the flow
As a trigger i'm using Recurrence set to run at 13:00 every day.  

Next is Current Time action so I can use it to check if it's a weekend and filter out those pesky weekends. I don't need a notification during the saturday or sunday.

I'm passing this current time to Compose where I'm using [dayOfWeek](https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference#dayOfWeek) function to parse out the day of week.   
Here's the full expression
```
dayOfWeek(string(body('Current_time_')))
```
So I'm passing Current time value as string to dayOfWeek-function which gives me day of the week as a number (0 = Sunday, 1 = Monday... 6 = Saturday)

Next action is a simple String which contains start of my teams message. Message itself needs to be easy to read so There's a bit HTML [unordered list](https://www.w3schools.com/tags/tag_ul.asp) formatting going on
```html
<br /><ul>
```

And the fourth action is just a simple Get my profile (v2) so I don't need to type my email to the teams message recepient (and so this is easy to share for others).


![Picture 1. Start of the flow](/assets/img/2023-01-17-Email-summary-to-teams/1-startOfFlow.png)

# Check if it's a week day
Next we go to Condition which checks if it's a week day.  
Condition itself is quite simple. Check if Compose - DayOfWeek output is not 0 or 6.
If yes, process flow further.  
If no then terminate the flow. You could also this empy but I wanted to add Terminate status as cancelled to indicate the flow have been cancelled and not successed or failed. 

![Picture 2. Condition, week day check](/assets/img/2023-01-17-Email-summary-to-teams/2-weekDayCondition.png)

If yes is where all the real magic happen. 

Get emails (V3) action is fetching emails with following settings:
- Folder: `Inbox`
- Fetch Only Unread Messages: `Yes`
- Include Attachments: `No`
- Search query: `received: @{formatDateTime(utcNow(),'yyy-MM-dd')}`
- Importance: `Any`
- Only with Attachments: `No`

The trickies part was to get Search query to work properly. Power Automate shows there's a Received Time property but you need to use it's internal name `Received`. I'm using [utcNow](https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference#utcnow)-function inside [formatDateTime](https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference#formatDateTime)-function to get `yyy-MM-dd` date to filter out all of todays emails.

![Picture 3. Get emails action](/assets/img/2023-01-17-Email-summary-to-teams/3-GetEmails.png)

Next I'm checking if there's any new emails. Wishful thinking that there won't be any new emails and I can use Terminate action status as cancelled in `If no` block.  
Condition is just a basic [empty](https://learn.microsoft.com/en-us/azure/logic-apps/workflow-definition-language-functions-reference#empty)-function check to check if Get emails body contains values. Get emails body is an array so we can use empty-function which returns true(if it's empty) or false(if there's content) value. Remember, you need to use `True`expression to compare empty value.   
Here's the expression:  
```
empty(outputs('Get_emails_(V3)')?['body/value'])
```
![Picture 4. Check if there's emails](/assets/img/2023-01-17-Email-summary-to-teams/4-ConditionEmailsFound.png)

If Get emails have content then append relevant information to `MessageContent`-variable which was initialized at start of the flow.  
```html
<li>@{items('Apply_to_each')?['subject']} (from: @{items('Apply_to_each')?['from']})</li>
```
I will append html list item which contains `Subject (from: sendersEmailAddress)` as a value.

# End of the flow

Last part of the flow is to close our MessageContent so Teams can render the message correctly and send the results to you.

Just a simple `</ul>` tag to `apped to string variable` action to close out the html.

And the last action is basic Post message in a chat or channel action where I send a message to myself. Message content is basically just a brief header & `MessageContent`-variable content.

![Picture 5. End of flow](/assets/img/2023-01-17-Email-summary-to-teams/5-endOfFlow.png)

# End result
And here's the whole flow 
![Picture 6. Full flow](/assets/img/2023-01-17-Email-summary-to-teams/6-fullFlow.png)

This was fairly easy flow to create but really handy to me. I've been using this for a one month already and I notice a huge difference on how much less there's distractions since outlook is not making sounds or showing icons/blinking tabs.
And yes I could make outlook rules to filter out emails but what's the fun in that. Let's face it. If you create a rule to move messages to folders out of sight out of mind most of the time you just forget them there.

Here's an example how it looks on teams  
![Picture 7. Teams message](/assets/img/2023-01-17-Email-summary-to-teams/7-TeamsMessage.png)

Here's a [github link to my solution](https://github.com/apaivinen/powerautomate/tree/main/Email%20summary%20to%20teams) if you don't want to create everything by yourself. Don't forget to update connections during the import!


