---
layout: post
title: "Analyzing GitHub Stars with Imply Polaris"
categories: blog druid imply polaris sql tutorial
---

![Sterntaler drawing](/assets/2023-07-12-01-Ludwig_Richter-The_Star_Money-2-1862.jpg)

## Why all this?

A while ago, [Will](https://twitter.com/whycaniuse) asked if we could measure [community engagement](https://www.swyx.io/measuring-devrel) in the [Apache Druid](https://druid.apache.org/) community by analyzing the number of [GitHub stars](https://docs.github.com/en/rest/activity/starring) that the [Druid source repository](https://github.com/apache/druid) got over time. He wanted to compare that development with other repositories within the realtime analytics ecosystem, and possibly identify segments of GitHub users that had starred multiple repositories out of the list we are looking at.

This blog is _not_ about the results of that endeavor. Instead, I am going to look at an interesting data/query modeling problem I encountered on the way.

## The dataset

Let's get the stargazers for various repos that are either competitive or complementary with druid. This includes

- other realtime analytics datastores
- streaming platforms
- stream processors
- frontend (bi) tools.

For each stargazer record, we store

- the user
- the repository
- date and time when it was starred; this will be the primary timestamp for the Druid data model.

### How to get the data

The data we are going to analyze comes from the [GitHub stargazers API](https://docs.github.com/en/rest/activity/starring?apiVersion=2022-11-28#list-stargazers). [Vijay has written a great blog about this](https://dev.to/vnarayaj/analysing-github-stars-extracting-and-analyzing-data-from-github-using-apache-nifir-apache-kafkar-and-apache-druidr-280); I am using a simpler approach with a Python script that runs once and tries to retrieve all the data.

This probably warrants another blog about the quirks of the GitHub API, so for now let a few remarks suffice.

- Surprise: [Elon Musk](https://twitter.com/elonmusk) did not invent [API rate limiting](https://docs.github.com/en/rest/rate-limit/rate-limit?apiVersion=2022-11-28)! Our first idea was to get _all the repositories_ that Druid stargazers also starred. This approach is not viable.
- There are primary (hard) and secondary rate limits. Either way, if you hit a limit, GitHub throws a 403 error at you. The required action depends on the type of rate limit that you hit.
- The API imposes [pagination](https://docs.github.com/en/rest/guides/using-pagination-in-the-rest-api?apiVersion=2022-11-28) with a maximum page size of 100 records.
- The maximum page index you can retrieve is 399.
- As a consequence, _you will not get more than 40,000 stars for any one repository_, which will soon become important.

You can find the code that I used, as well as all the SQL samples from this post, in [my GitHub repository](https://github.com/hellmarbecker/druid-stargazers).

### loading the data into polaris

show a data sample

## first visualization

this shows only the new stars for every point in time

![Visualization: New Star Events over Time](/assets/2023-07-12-02-eventdata.jpg)

but what we really want: the growth of stars over time. let's do this with monthly resolution

## trying a self join

recur to the earlier blog about emulating window functions

show the query

![Visualization: Cumulative Sums with Self Join](/assets/2023-07-12-03-selfjoin.jpg)

highlight how the superset line drops to zero. why is that?
remember the 40k limit? so we don't get new entries after a certain date and the join has nothing to join against

## so let's build up a calendar dimension instead, shall we

this is a case for using the date_expand function with unnest. we want a list of all months from the min to max date in our data, let's try this:

 -> canvas-monthly-FAIL.sql

but this gives an error

 -> error message

so instead, let's use the next largest interval that works with date_expand, which is week, and dedup afterwards

 -> canvas-monthly.sql

## join up against the fact data

 -> cumulative-monthly-FAIL.sql

this fails

 -> error message

the message is clear, we need an equals join. let's do a workaround by adding the repo to the calendar canvas to use it as a join key. so this becomes a cross join:

 -> canvas-monthly-with-repo.sql

then do a join on repo and tuck the unbound preceding condition away into a [filtered metric](link to druid doc)

 -> cumulative-monthly-canvas-final.sql

use this query to define a cube in pivot and see the result

![Visualization: Cumulative Sums](/assets/2023-07-12-04-calendar-canvas.jpg)

now the superset stars max out at 40k but they don't drop to zero!

## Conclusion

- the [self join approach to cumulative sums](link to earlier blog) fails when there are "holes" in the data (aka factless facts)
- the best approach to counter this is building an explicit calendar dimension
- date_expand can be used to build a calendar canvas but has some limitations, we showed how to work around those
- also we learned how we can work around the join limitation in druid sql by adding a synthetic join key to the calendar dimension and using a filtered metric

"Ludwig_Richter-The_Star_Money-2-1862" (via [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Ludwig_Richter-The_Star_Money-2-1862.jpg)) is in the <b><a href="https://en.wikipedia.org/wiki/public_domain" class="extiw" title="en:public domain">public domain</a></b> in its country of origin and other countries and areas where the <a href="https://en.wikipedia.org/wiki/List_of_countries%27_copyright_lengths" class="extiw" title="w:List of countries&#39; copyright lengths">copyright term</a> is the author's <b>life plus 100 years or fewer</b>.

