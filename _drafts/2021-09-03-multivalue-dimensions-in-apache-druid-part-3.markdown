---
layout: post
title:  "Multi-Value Dimensions in Apache Druid (Part 3)"
categories: blog apache druid imply
---

In which we continue the quest about multi-value handling and try to answer the question: What is it good for?

This is part 3 of the miniseries about multi-value dimensions in Apache Druid. The previous posts can be found here:
- [part 1](/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/)
- [part 2](/2021/08/29/multivalue-dimensions-in-apache-druid-part-2/)

Let's assume I am running an Italian restaurant. Each time one of my patrons visits, I am noting down which items they consume as part of a menu, in order.

I might end up getting something like this:
```json
{ "date": "2021-09-25", "customer": "Fangjin", "orders": [ "pizza", "tiramisu", "espresso", "espresso" ] }
{ "date": "2021-09-25", "customer": "Gian", "orders": [ "pizza", "espresso", "tiramisu" ] }
{ "date": "2021-09-25", "customer": "Vadim", "orders": [ "pizza", "tiramisu" ] }
{ "date": "2021-09-25", "customer": "Rachel", "orders": [ "pizza", "tiramisu", "espresso" ] }
{ "date": "2021-09-25", "customer": "Xiaolan", "orders": [ "pizza", "espresso" ] }
{ "date": "2021-09-25", "customer": "Jessy", "orders": [ "pizza", "espresso", "espresso" ] }
```
Probably none of these customers will ever do these exact orders in real life, but let's keep this for educational purposes only.

Also note that I am intentionally leaving the question open whether the pizza has pineapples.

Ingest this data sample into Druid by copypasting it into the inline ingestion wizard. The previous articles in the series explain how to do that.

Create virtual copies of the `orders` dimension using the transform step:

![](-transform)

Don't worry that the order of the fields looks all messed up, we'll fix that in a moment. Continue the ingestion wizard, entering `day` as the segment granularity value, and proceed all the way to the screen where you can edit the JSON spec.

