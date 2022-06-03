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

An additional nuisance is that unique counts don't work well with rollups. For the example above, it would be great if we could have just one row of data per 15 minute interval[^1], show, and episode. After all, we are not interested in the individual user IDs, just the unique counts.

[^1]: You might think: why 15 minutes and not just 1 hour? One of my customers pointed out why: 15 minute intervals work better with international timezones because those are not always aligned by hour. India, for instance, is 30 minutes off, and Nepal is even 45 minutes off. With 15 minute aggregates, you can get hourly sums for any of those timezones, too!) 

Is there a way to avoid crunching the detail data every single time, and maybe even enable rollup?

### Fast Approximation with Set Operations: Theta Sketches

Theta Sketches are a probabilistic data structure to enable fast approximate analysis of big data. Druid's implementation relies on the [Apache DataSketches](https://datasketches.apache.org/) library, and you can find a detailed explanation of how they work in the docs there.

Theta Sketches have a few nice properties:

- They give you a **fast approximate estimate** for the distinct count of items that you put into them.
- They are **mergeable**. This means we can work with rolled-up data and merge the sketches over various time intervals. Thus, we can take advantage of Druid's rollup feature.
- What's even more, theta sketches support **set operations**. Given two theta sketches over subsets of the data, we can compute the union, intersection, or set difference of these two. This gives us the ability to answer the questions above about the number of visitors that watched a specific combination of episodes.

There is a lot of advanced math behind theta sketches. But with Druid, you do not need to bother about the complex algorithms - theta sketches just work!

### Creating a Data Sample

Sample file:

```csv
date,uid,show,episode
2022-05-19,alice,Game of Thrones,1
2022-05-19,alice,Game of Thrones,2
2022-05-19,alice,Game of Thrones,1
2022-05-19,bob,Bridgerton,1
2022-05-20,alice,Game of Thrones,1
2022-05-20,carol,Bridgerton,2
2022-05-20,dan,Bridgerton,1
2022-05-21,alice,Game of Thrones,1
2022-05-21,carol,Bridgerton,1
2022-05-21,erin,Game of Thrones,1
2022-05-21,alice,Bridgerton,1
2022-05-22,bob,Game of Thrones,1
2022-05-22,bob,Bridgerton,1
2022-05-22,carol,Bridgerton,2
2022-05-22,bob,Bridgerton,1
2022-05-22,erin,Game of Thrones,1
2022-05-22,erin,Bridgerton,2
2022-05-23,erin,Game of Thrones,1
2022-05-23,alice,Game of Thrones,1
```

- enable rollup
- episode has been selected as a metric, convert into a string dimension
- uid, convert into a theta sketch metric -> we are not interested in the individual names, only counts
- query granularity: day
- segment granularity: day



---

"This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che" by Medical Heritage Library, Inc. is licensed under [CC BY-NC-SA 2.0](https://creativecommons.org/licenses/by-nc-sa/2.0/?ref=openverse&atype=html)
