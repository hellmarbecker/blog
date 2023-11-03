---
layout: post
title:  "New in Imply Polaris: Data Retention Policy"
categories: blog imply polaris druid data_lifecycle
twitter:
  image: /assets/2023-09-24-02.jpg
---

[Apache Druid](https://druid.apache.org/) has always had built-in data lifecycle management by way of [retention rules](https://druid.apache.org/docs/latest/operations/rule-configuration/). Specifying fixed time intervals or relative periods, you would tell Druid to retain only data segments that are not older than _x_ days.

The [mid-August release](https://docs.imply.io/polaris/release#20230816) of Polaris brings retention management to Imply Polaris, the fully managed analytics service powered by Druid. You can set the retention policy by table. Here is how it's done:

In the _Tables_ view, select the `...` menu for the table that you want to set the retention policy for.

![Tables view with context menu](/assets/2023-09-24-01.jpg)

In the _Edit table_ screen, find the barrel icon with `Data retention` next to it. Select `Specific`, and enter the desired period. The format is [ISO-8601 duration](https://en.wikipedia.org/wiki/ISO_8601#Durations), so for instance, `P7D` means 7 days (before the current date.) Any data that is older (by primary timestamp) will be scheduled for deletion after 30 days.

![Table editor with retention menu](/assets/2023-09-24-02.jpg)

Then hit `Update` to apply the changes.

## Conclusion

- Data retention management is now available in Polaris.
- Unlike Druid default (which retains data in deep storage indefinitely), data dropped from Polaris will be deleted permanently after 30 days.
