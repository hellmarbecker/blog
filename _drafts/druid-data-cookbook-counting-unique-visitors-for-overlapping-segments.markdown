---
layout: post
title:  "Druid Data Cookbook: Counting Unique Visitors for Overlapping Segments"
categories: blog druid imply statistics datasketches tutorial
---
![Druid Cookbook](/assets/2021-12-21-elf.jpg)

## The Problem of Getting Unique Counts

A common problem in clickstream analytics is counting unique things, like visitors or sessions. Generally this involves scanning through all detail data, because unique counts **do not add up** as you aggregate the numbers.

For instance, we might be interested in the number of visitors that watched episodes of a TV show. Let's say we found that at a given day, 1000 (unique) visitors watched episode 1 and 800 visitors watched episode 2. But we might have more questions, such as:

- How many visitors watched _both_ episodes?
- How many visitors are there that watched _at least one_ of the episodes?
- How many visitors watched episode 1 _but not_ episode 2?

The answer is: there is no way to tell by just looking at the aggregated numbers. We will have to go back to the detail data and scan every single row. If the data volume is high enough, this may take long, meaning that an interactive data exploration is not possible.

Or is there a better way?

## Fast Approximation with Set Operations: Theta Sketches




---

"This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che" by Medical Heritage Library, Inc. is licensed under [CC BY-NC-SA 2.0](https://creativecommons.org/licenses/by-nc-sa/2.0/?ref=openverse&atype=html)
