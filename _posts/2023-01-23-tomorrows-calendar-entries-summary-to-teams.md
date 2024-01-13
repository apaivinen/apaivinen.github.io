---
title: Tomorrows calendar entries summary notification to teams chat as flow bot
date: 2023-01-23 5:00:00 
categories: [Power Automate, Personal Automation]
tags: [Power Automate, Flow, Automation, Teams, Outlook, Personal Automation, Tutorial,Email]
layout: post
toc: true
comments: false
math: false
mermaid: false
---

I wanted to create a summary of tomorrows meetings so I could avoid to check the calendar before I call it for the day. Yet another attempt to have reasons not to open outlook as often or keep it open all the time. Here's a [link](https://github.com/apaivinen/powerautomate) to my Github repository where you can find the flow for importing. Rest of the post is for detailed description of the flow itself.

I started to think of the logic and what it should be. 
1. Get calendar events for tomorrow every day except weekends (we don't want to have work releated messages during the weekend.)
2. filter out weekends or "empty" days
3. Create a summary of meetings.
4. post Summary to teams chat 

# Start of flow

This is quite simple but when I started to work on this I noticed couple of pitfalls mainly releated to time.

Trigger is just a simple Recurrence. I used 1 day interval at 13:00 (UTC +2 Helsinki).

Then I created a group of variables(and composes). 
1. Initialize Variable - Tomorrow 
   This is used for getting tomorrows date for next couple of composes and checking if it's not a weekend.  To get tomorrows date I used addDays and utcNow expressions. 
   Value is `addDays(utcNow(),1,'yyyy-MM-dd')`
2. Compose - StartTime 
   This is used to filter events starting from midnight. I used to concat previous tomorrow variable with T00:00:00.0000000Z to get working filter for tomorrow midnight
   Value is `concat(variables('Tomorrow'),'T00:00:00.0000000Z')`
3. Compose - EndTime
   This is almost the same as Start time but just adding T19:00:00.0000000Z to end of date to get full realistic date range. This is for me the latest realistic time to have any kind of webinar to listen. Usually I just try to get my hands on recordings rather than spend my evenings in webinars. 
   Value is `concat(variables('Tomorrow'),'T19:00:00.0000000Z')`
4. Initialize Variable - MessageContent
   Just a simple string variable where's a line break and html list start tag. This is getting more information appended to it when reading meeting information.
   Value is `<br /><ul>`
   
![Picture 1. Start of flow](/assets/img/2023-01-23-tomorrows-calendar-entries-summary-to-teams/1-StartOfFlow.png)

# Condition, check if it's week day
Next is a check for saturday & sunday.  
This was quite simple since there's a dayOfWeek expression which gives you a number representing days of a week. FOR FUTURE REFERENCE: week starts from sunday which is 0 and ends to saturday which is 6. This was a surprise for me since here week starts from monday and ends to sunday. 
So basically I'm just comparing if dayOfWeek is not equal to 6 or 0.
Expression which to compare `dayOfWeek(variables('Tomorrow'))`
If dayOfWeek is 6 or 0 go to no block and just terminate the flow as cancelled.

![Picture 2. Condition, weekday](/assets/img/2023-01-23-tomorrows-calendar-entries-summary-to-teams/2-ConditionWeekDay.png)

Next is Get my profile (V2) for getting  email address for teams message recepient without typing it myself. This is just to get the flow more dynamic. Less messing with the actions when importing  the flow.

# Get events tomorrows events
Get events (V4) is getting filtered events from my Calendar. I'm using start time and end time composes for this filter. Here's the Filter Query `Start/DateTime ge '@{outputs('Compose_-_StartTime')}' AND Start/DateTime le '@{outputs('Compose_-_EndTime')}'`

And I'm sorting the entries by time since I wanted to have logical & simple summary. Order by `Start/DateTime asc`

![[3-GetEvents.png]]

# Check if there's meetings and process them

Next in line is check if there's no events. Since Get Events returns array every time even when there's no event to show I used empty expression to check for empty array. If `empty(outputs('Get_events_(V4)')?['body/value'])` is not equal to `true`
Condition terminates as cancelled if there's no meetings. If meetings are found then loop them trough and append to MessageContent variable.

Here's the content I'm appending to Message content
```html
<li><b>@{body('Convert_time_zone')}</b> - @{items('Apply_to_each_-_Loop_through_meetings')?['subject']} </li>
```

So basically I wanted to have each meeting start time and meeting name to be in summary. I used Convert Time Zone to convert Get events Start time to finnish time format `HH:mm`
Then I added html list item entry with  "***timestamp*** - Meeting Name" to Append to string variable value. 

![Picture 4. Check if there's meetings](/assets/img/2023-01-23-tomorrows-calendar-entries-summary-to-teams/4-MeetingsFound.png)

# End of flow

Next step out side of meeting loop there needs to be html ul end tag to be appended so html will be complete. 
Append to string variable - MessageContent value: `</ul>`

and last but not least post the results to teams chat as flow bot. Recepient is mail address from Get my profile action. And message I just included a bit text and MessageContent variable.

![Picture 5. End of flow](/assets/img/2023-01-23-tomorrows-calendar-entries-summary-to-teams/5-EndOfFlow.png)

That's it. 

I know I could have used more (nested) expressions to eliminate composes and tomorrow variable but I wanted to do this like the way I did to keep it simple. For me there's a lot more effort to decipher huge nested expressions vs. using variables and nesting them together.


# Notice when importing

Don't forget to check connections for 
- Get my profile
- Get events
- Post message in a chat or channel

Also check your calendar ID from Get events!

And [here's a github link to my solution](https://github.com/apaivinen/powerautomate/tree/main/Tomorrows%20calendar%20entries%20summary%20notification)