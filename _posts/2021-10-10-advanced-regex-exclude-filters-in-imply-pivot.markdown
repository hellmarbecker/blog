---
layout: post
title:  "Advanced Regex Exclude Filters in Imply Pivot"
categories: blog apache druid imply tutorial pivot regex
---

In this short post I want to take a closer look at Imply Pivot and how to address a common scenario for analytics: Let's assume we want to filter on a dimension value, but we want to **exclude** rather than include a set of dimension values.

This is easy! you may say. Just use the built in exclusion function that you can use in a dimension filter:

![Static exclusion filter](/assets/2021-10-10-1-exclude-static.jpeg)

But not so fast! While the filter dialog features a search box to find values by prefix or included value, you will eventually end up with a _static list of values_. For an _inclusion_ filter you could instead enter a substring ("Contains"), or even a regular expression. It would be nice to have a similar function to exclude values based on a rule expression!

For instance, I am looking at the [quickstart Wikipedia](https://docs.imply.io/latest/quickstart/#load-a-data-file) data set and I would like to exclude all the pages that start with either `Wikipedia:` or `User:`. But every day there can be new users and pages, so it is important to keep the filter general.

The solution lies in using regular expressions. In order to exclude certain words, the regular expression engine supports [negative lookaheads](https://www.regular-expressions.info/lookaround.html). The phrase `(?!abc)`, in a regular expression, matches any string that is _not followed by_ "abc". To exclude occurrences at the beginning, we prepend this by the `^` symbol. Also, because we want to exclude either of two words, we can make use of the conjunction operator `|`. With that, the filter regex becomes:
```regex
^(?!User:|Wikipedia:)
```
Let's enter this in Pivot:

![Regex filter with negative lookahead](/assets/2021-10-10-2-exclude-dynamic.jpeg)

You can experiment with more complex exclusion rules: for instance, you could include wildcards or character lists to match the same prefix in other languages too!

## Learnings

- Regular expressions are a powerful way of filtering string dimensions in Pivot.
- Regular expressions can be used for exclusion filtering using negative lookaheads.
