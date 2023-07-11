---
layout: post
title: "Analyzing GitHub Stars with Imply Polaris"
categories: blog druid imply polaris sql tutorial
---

## why this all

## the dataset

### dataset definition

for various repos that are either competitive or complementary with druid

such as

- other rtolap
- streaming platforms
- stream processors
- frontend (bi tools)

then get

- the user
- the repo
- when it was starred

### how to get the data

github api

this warrants another blog but for now just hint at the limitations

you will not get more than 40k stars which will soon become important

## loading the data into polaris

show a data sample

## first visualization

this shows only the new stars for every point in time

but what we really want: the growth of stars over time. let's do this with monthly resolution

## trying a self join

recur to the earlier blog about emulating window functions

show the query

![Visualization: Cumulative Sums with Self Join]()

highlight how the superset line drops to zerom why is that?
remember the 40k limit? so we don't get new entries after a certain date and the join has nothing to join against

## so let's build up a calendar dimension instead, shall we

this is a case for using the date_expand function with unnest. we want a list of all months from the min to max date in our data, let's try this:

canvas-monthly-FAIL.sql

but this gives an error

