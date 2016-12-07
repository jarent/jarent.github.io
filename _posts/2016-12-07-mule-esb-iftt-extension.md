---
layout: post
title: Mule ESB - IFTTT extension
---

I was trying to use [IFTTT](https://ifttt.com/discover) for quite a long time. The idea, functionality and easy setup are really promising. Based on the number of recipes (or applets after recent rebranding) and connected channels (or services) I think it is also very popular.

![ifttt-recipes](/images/ifttt-extension/ifttt-recipes.png "IFTTT recipes")

Unfortunately I always found some details missing in predefined IFTTT services. The restrictions of single **'if-this-then-that'** condition connecting only up to two services are too big constraint for my needs. I'm a programmer, used to being able to implement complicated decision logic and connect multiple services.

On the other hand I'm also lazy and would like to avoid a headache of implementing and maintaining all the services connections. If only I could create a plug-in that will extend IFTTT functionality and adjust it to my ideas...

It turned out that IFTTT fulfilled my wish with [Maker service](https://ifttt.com/maker)

Thanks to Maker service I can create **'inbound'** applet that will call extension service URL with received data from first IFTTT service. Later extension service will call second **'outbound'** applet and trigger second service. Thanks to this I can reuse already existing connections from IFTTT platform and expend it with my custom logic including additional services calls. There is no better way to show how it works than create a small proof of concept.

### Automated SMS bot

My 'pet project' is to connect bot built with [API.AI](https://api.ai/) to my Android Phone SMS. I want to have autoresponder for SMS texts regarding stuff I posted for sale to Craigslist (that's pretty much the only time I'm using SMS)

---
**Disclaimer:** My approach is good only for personal and limited use. If you plan to build something similar for business usage I suggest using Twilio programmable SMS - there is a great tutorial [how to connect dedicated SMS number with API.AI](https://docs.api.ai/docs/twilio-integration)

---
Going back to my project. I have to make final decision - what should I use for designing and exposing RESTful API, implementing business logic with two service calls to API.AI and IFTTT trigger channel? No surprise there - **Mule ESB** fits perfectly. Below I enlisted all the steps and components required to implement fully functional solution.

### API.AI bot

It took me a while to configure the agent according to my needs, probably because I was doing it for the first time. API.AI documentation was very helpful.

Following tutorial I created [agent](https://docs.api.ai/docs/concept-agents).

After that I added two [entities](https://docs.api.ai/docs/concept-entities)

* @for-sale - with list of items posted to Craigslist, together with their synonyms
* @for-sale-offer - composite entity with @for-sale and amount of offer

Then I added some example queries grouped in two [intents](https://docs.api.ai/docs/concept-intents)

* check-for-sale - sample questions for item availability
* price-offer - samples questions with price offering

While testing the agent using built-in console I noticed that it is better to process multi-sentences text as separate queries. Thanks to keeping [context](https://docs.api.ai/docs/concept-intents) API.AI engine is able to keep price-offer questions associated with an item mentioned earlier in check-for-sale question.

![sms-agent-intent](/images/ifttt-extension/sms-agent-intent.png "SMS Agent Intent")

For sample SMS text:

*Do you still have dryer? I will take it for 20 bucks*

thanks to processing it as two separate queries, API.AI first is able to recognize **dryer** as **@for-sale** item. Then correlating first with second query thanks to **sessionId** (phone number fits perfectly as sessionId for SMS texts) adds **20 dollars** as **offered price**, completing **@for-sale-offer** entity. I was really glad to see that API.AI works so good.

### SMS agent API

Next I designed API for my IFTTT plug-in using **RAML**. I had to remember about some restrictions of [Maker service](https://ifttt.com/maker) - there is no way to add any custom HTTP headers to the call, only request body can be customized. Because of that I had to put both data and tokens into request body.

All I needed was one operation (**/smsagent/query**) that will allow to POST payload with:

* **sms text**, **sms from number** - sourced from IFTTT [Android SMS service](https://ifttt.com/android_messages)
* **API AI token** - client token from API.AI agent settings pages
* **IFTTT maker trigger event name and key** - from IFTTT maker service page

sample request:

{% highlight json %}
{"sms":{"text":"Hi. How are you?",
        "fronNumber":123456789
        },
"apiAiToken":"asdfsdfsdfdsf",
"iftttMakerTrigger":{
        "eventName":"smsAgentEvent",
        "key":"sfsdfsdfdsf"
        }      
}
{% endhighlight %}

Remember to replace **apiAiToken**, **eventName** and **key** with your private values. Sample requests can be executed from [API console](http://ec2-54-84-32-224.compute-1.amazonaws.com:8080/console).

### SMS agent application ###

To implement SMS agent API I created new **sms-agent** Mule application with flow implementing **post:/smsagent/query** operation:

![sms-agent-flow](/images/ifttt-extension/sms-agent-flow.png "SMS Agent Flow")

Mule application source code was uploaded to [github repository](https://github.com/jarent/sms-agent).

Flow is straightforward. I implemented it using BDD approach I had written about in [previous post]({% post_url 2016-06-02-munit-spock-tests-as-specification %}), so now I can just present **SmsAgentSpec** test execution report to describe details of application functionality:

![sms-agent-spec-report](/images/ifttt-extension/sms-agent-spec-report.png "SMS Agent Spec Report")

HTML version of report can be checked [here](https://rawgit.com/jarent/sms-agent/master/build/spock-reports/index.html)

After creating an app I deployed it on AWS EC2 node running **Mule ESB 3.8.0 CE**. The app is uploaded to S3 bucket and after that CodeDeploy pipeline automatically put the app into **/apps** folder. Details of the configuration can be checked in [here](https://github.com/jarent/sms-agent/tree/master/src/assembly).

### Outbound SMS applet

IFTTT Applet for outbound SMS should have **'Maker'** trigger in **this** part, and **'Android Phone SMS'** in **that** part of 'if-this-than-that'.

![outbound-applet](/images/ifttt-extension/outbound-applet.png "Outbound Applet")

The value set up for **'eventName'** will have to be later provided in [Inbound SMS applet](#inbound-sms-applet), together with **Maker trigger key**. You can check your key by going to Maker service [settings page](https://ifttt.com/services/maker/settings). There is an URL on that page starting with https://maker.ifttt.com/use/ that will show detail page with your personal key and how the service should be triggered.

**[Sms-agent](#sms-agent-application)** flow before triggering IFTTT maker channel is setting up *API.AI bot response* as **Value1** and *from phone number* as **Value2** - those ingredients should be used to fill Phone Android SMS **text** and **phone number** fields.


### Inbound SMS applet

At the end I added first component in the whole process - inbound SMS applet. Receiving **Android Phone SMS** should be in **this** part, and Maker service in **that** part of **'if-this-then-that'** rule.

![inbound-applet](/images/ifttt-extension/inbound-applet.png "Outbound Applet")

**Maker service** settings:

* **URL** - Public endpoint URL for sample deployment of [SMS agent application](#sms-agent-application)  is: *"http://ec2-54-84-32-224.compute-1.amazonaws.com:8080/api/smsagent/query"*. It can be reused because **API.AI** and **IFTTT Maker** tokens and **event name** are provided in the request:
* **Method** - *"POST"*
* **Content-Type** - *"application/json"*
* **Body** - [SMS agent application](#sms-agent-application) valid JSON request with text and fromNumber filled using **Text** and **FromNumber** ingredients from Andoid Phone SMS service.

{% highlight json %}
{ "sms":
        { "text": " { { Text } }",
        "fromNumber": { { FromNumber } }
        },
"apiAiToken": "fill with personal API AI token",
"iftttMakerTrigger":
        { "eventName": "fill with event name set up in Outbound applet",
        "key": "fill with personal IFTTT Maker key >>" }
}
{% endhighlight %}


### How does it work?

Final end-to-end flow looks like this:

**Android SMS -> IFTTT -> sms-agent app on Mule ESB -> API.AI -> IFTTT -> Android SMS**

Totally five integration steps, so testing and debugging was tricky. Couple of tips:

* IFTTT maker service execution can be delayed around 15 minutes. If you are in hurry and want to check your changes immediately, use **'Check now'** option from applets settings
* If still nothing happened, go to **View Activity log**. It allows to check time and result of applet run. Unfortunately, in case of problems it will not provide error details - that is big problem, because you have to figure out by yourself what was wrong with HTTP call (like you see on the picture below there is no error details).

![applet-activity-log](/images/ifttt-extension/applet-activity-log.png "Activity Log")

Everything else runs perfectly fine.

It is time to post couple of items on 'Craigslist' and see how the solution works with text messages from real people.
