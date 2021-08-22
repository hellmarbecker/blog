---
layout: post
title:  "Creating PagerDuty Alerts with Imply Pivot"
categories: blog apache druid imply tutorial pivot operations
---

In an [earlier post](https://blog.hellmar-becker.de/2021/07/31/sending-automatic-email-reports-from-imply-pivot/) I discussed how to enable the [alerting](https://docs.imply.io/latest/alerts/) capabilities of [Imply Pivot](https://imply.io/product/imply-pivot) and send out alerts via email.

Today I am going to connect Pivot to [PagerDuty](https://www.pagerduty.com/), as an example of a custom [webhook](https://web.archive.org/web/20120413121142/http://wiki.webhooks.org/w/page/13385124/FrontPage) configuration. You need a PagerDuty account for this, but you can use the free 14 day trial for the tutorial. You also need to set up a few things in PagerDuty so you can use the Events API to receive alerts.

## Preparing PagerDuty

In order to send alerts to PagerDuty, create a service in PagerDuty and an integration with the [Events API v2](https://developer.pagerduty.com/docs/events-api-v2/overview/). You find this in the Service Directory of PagerDuty:

![Service Directory](/assets/2021-08-22-pd1.jpg)

After creating a service, add a new Events API integration to that service:

![Service Directory](/assets/2021-08-22-pd2.jpeg)
![Service Directory](/assets/2021-08-22-pd3.jpeg)

Go to the settings for the integration

![Integration Settings](/assets/2021-08-22-pd4.jpeg)

and note down the Integration Key - you will need it in the next step.

![Integration Key](/assets/2021-08-22-pd5.jpeg)

## Configuring the alert

In Pivot, set up an alert and configure it to use a webhook. For the `"routing_key"` field, use the integration key that you noted down before.

The URL for the Events API is `https://events.pagerduty.com/v2/enqueue` and your payload should look like this:

```json
{ 
  "payload": { 
    "summary": "%summary%",
    "severity": "info",
    "source": "Imply News"
  },
  "routing_key": "<insert your integration key>",
  "event_action": "trigger"
}
```
You can set this up during alert creation, or edit an existing alert and configure the Delivery options. You also have the opportunity to send a test alert:

![Pivot Alert Delivery Options](/assets/2021-08-22-pivot.jpg)

If you got everything right, you can see the alerts coming into your PagerDuty account:

![Alerts incoming](/assets/2021-08-22-pdalert.jpg)
