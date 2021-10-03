---
layout: post
title:  "Multi-Value Dimensions in Apache Druid (Part 3)"
categories: blog apache druid imply
---

Let's do some customer segmentation in out little restaurant!

This is part 3 of the miniseries about multi-value dimensions in Apache Druid. The previous posts can be found here:
- [part 1](/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/)
- [part 2](/2021/08/29/multivalue-dimensions-in-apache-druid-part-2/)
- [part 3](/2021/09/25/multivalue-dimensions-in-apache-druid-part-3/)

Last time we were able to learn a lot about orders in a fictional Italian restaurant. We managed to isolate groups of customer visits with the same items, the same number of the same items, and even the exact same sequence of orders.

Marketeers are very interested in this kind of information. They map it to customer segments, in order to create tailored offerings for different segments. Ideally, this breaks down to very small groups of customers with similar preferences.

Let's see if Druid could help a marketeer!

We are using the `ristorante` datasource from [part 3](/2021/09/25/multivalue-dimensions-in-apache-druid-part-3/), and start with the analysis how many customers bought each item:

![](/assets/2021-10-03-1-groupby-orders_set.jpeg)

Let's find out who the customers were that ordered each basket. I hinted at it last time: this is where array aggregation functions come in. The `ARRAY_AGG` function creates a SQL array from all the values in a group by bucket:

![](/assets/2021-10-03-2-groupby-orders_set-with-array_agg.jpeg)

And just like that, we got the list of customers for each dish!

SQL arrays can be a bit unwieldy, though. For client software that cannot handle those, there is the new `STRING AGG` function in Druid that concatenates the values into a string, using a configurable delimiter.

![](/assets/2021-10-03-3-groupby-orders_set-with-string_agg.jpeg)

And this solves the mystery of the `STRING_AGG` function!

## Learnings

- Multi value functions and array aggregation functions do different things but complement each other.
- Together, they are powerful for order and basket analysis and for customer segmentation.
