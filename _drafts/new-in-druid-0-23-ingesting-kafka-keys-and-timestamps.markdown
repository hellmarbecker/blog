---
layout: post
title:  "New in Druid 0.23: Ingesting Kafka Keys and Timestamps"
categories: blog imply druid kafka eventstreaming tutorial
---

Among the many new features in [Druid 0.23](https://github.com/apache/druid/releases/tag/druid-0.23.0) is the ability to handle Kafka metadata upon ingestion. This includes

- the Kafka timestamp
- the message key
- Kafka message headers.

While customers of [Imply](https://imply.io/) have been able to take advantage of this functionality for a few months already, it is still officially considered an alpha feature. But why would we want this anyway? 

Let's have a look at this today.

## Why does it matter?

