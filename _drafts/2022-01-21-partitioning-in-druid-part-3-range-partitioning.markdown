---
layout: post
title:  "Partitioning in Druid - Part 3: Range Partitioning"
categories: blog apache druid imply tutorial
---
![Test Tubes](/assets/2022-01-21-0-test-tubes.jpg)

This is part 3 of the miniseries about data partitioning in [Apache Druid](https://druid.apache.org/). Previous articles in this series:
- [part 1](/2022/01/06/partitioning-in-druid-part-1-dynamic-and-hash-partitioning/)
- [part 2](/2022/01/14/partitioning-in-druid-part-2-single-dimension-partitioning/)

## Recap

In the previous articles, we learned that _hash_ partitioning is great at ensuring uniform segment size. _Single dim_ partitioning can group data according to frequent query patterns - however, if the partition key has a skewed distribution you may end up with very uneven segment sizes.

_Range Partitioning_ fixes this problem. At the time of this writing it isn't part of the standard Druid release yet - it is slated to be released with Druid 0.23. But you can get a sneak peek when you [build the current snapshot from the source](https://github.com/apache/druid/blob/master/docs/development/build.md).

## Range (or Multi Dimension) Partitioning

Range partitioning works by allowing a list of dimensions to partition by, effectively creating a composite partition key. Note that this is _not_ a multi-level partitioning scheme as some databases know it. Rather, if a value in the first listed dimension is so frequent that it would create a partition that is too big, the second dimension is used to split the data, then the third, and so on.

Also, data is sorted by this composite key. This has the effect that a query that groups or filters by all (or the first _n_) of the partition key columns will run faster. In this respect it works a bit like composite indexes in databases.

As mentioned above, you will have to build your own snapshot Druid version to try this out.

Range partitioning is not supported by the web console wizard, so we have to resort to a little trick. As before, I'll show this with the quickstart wikipedia dataset.

Configure your ingestion just like in part 1, pretending that you want to do do a hash partitioning. Set the segment size to 14,000 rows, however this time, enter both `channel` and `user` as the partitioning column:

![First step in configuring](/assets/2022-01-21-1-params.jpg)

Then, continue in the wizard until you get to edit the JSON Spec. On this screen, look up the partitioning configuration and replace the word `hashed` by `range`:

![Editing the JSON spec](/assets/2022-01-21-2-jsonspec.jpg)



