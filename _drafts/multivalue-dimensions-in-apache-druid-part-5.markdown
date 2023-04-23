---
layout: post
title:  "Multi-Value Dimensions in Apache Druid (Part 5)"
categories: blog apache druid imply
---

An interesting discussion that I had with a Druid user prompts me to continue the loose miniseries about multi-value dimensions in Apache Druid. The previous posts can be found here:

- [part 1](/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/)
- [part 2](/2021/08/29/multivalue-dimensions-in-apache-druid-part-2/)
- [part 3](/2021/09/25/multivalue-dimensions-in-apache-druid-part-3/)
- [part 4](/2021/10/03/multivalue-dimensions-in-apache-druid-part-4/)

In [part 1](/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/) I pointed out what multi-value dimensions (MVD) are, and how they behave with respect to `GROUP BY` (they do an implicit unnest or, if you will, a lateral join), and also with respect to filtering using a `WHERE` clause (you get all the rows that match the `WHERE` condition, but no unnesting happens.)

But what if you want to combine grouping and filtering? The behavior of Druid in these case could be a bit surprising. Let's have a look!

I am using Imply's version 2023.03.01 of Druid, because I am going to show a few things using Imply's graphical frontend. If you want to run the SQL examples only, Druid 25 quickstart works fine.

We are using the `ristorante` datasource from [part 3](/2021/09/25/multivalue-dimensions-in-apache-druid-part-3/); to create the datasource, follow the instructions given there. (You can make your life a bit easier because by now, Druid allows specifying the multi-value handling mode in the wizard.)

Start with a simple analysis, breaking down the count of items by item and customer:

```sql
SELECT customer, orders, COUNT(*) AS numOrders
FROM "ristorante"
GROUP BY 1,2
```

![](/assets/2023-04-23-01.jpg)

No surprises here. The MVD is unnested and the counts are broken down by item, as expected.

## Quirks in multi-value filtering

Now let's filter by one specific item.

```sql
SELECT customer, orders, COUNT(*) AS numOrders
FROM "ristorante"
WHERE orders = 'tiramisu'
GROUP BY 1,2
```

![](/assets/2023-04-23-02.jpg)

The result contains a lot of items that are definitely not Tiramisu! We got the filtering behavior from the plain query (without `GROUP BY`) and only after that the unnesting was applied!

Maybe if we try to filter after the grouping step, it would work?

```sql
SELECT customer, orders, COUNT(*) AS numOrders
FROM "ristorante"
GROUP BY 1,2
HAVING orders = 'tiramisu'
```

![](/assets/2023-04-23-03.jpg)

Alas, the result is the same. No matter how you write the filter, the query plan always selects whole rows of data as they are in the datasource. This is a common trap for the unwary, although the behavior is documented [here](https://docs.imply.io/latest/druid/querying/multi-value-dimensions/#filtering) for native queries, into which SQL queries are translated internally.

The same paragraph also mentions [SQL multi-value functions](https://docs.imply.io/latest/druid/querying/sql-multivalue-string-functions/). This is where the path to a solution lies.

## Filtering multi-value strings, properly

The core to the solution is the `MV_FILTER_ONLY` function, which is applied to a multi-value field in the _projection_ clause of the `SELECT` statement. Its first argument is the field that you want to filter on, the second argument is an _array literal_ of the values that you want to keep. 

Arrays are currently the redheaded stepchildren of Druid data modeling, although this is about to change soon and there will be a lot more support for them. For now, you cannot declare an `ARRAY` column (MVDs are of type string), but you can define an array literal with the `ARRAY` constructor.

(There is also a complementary function, `MV_FILTER_NONE`, that keeps only the values that are _not_ contained in the array that you pass as the second argument.)

Let's put together the query:

```sql
SELECT 
  customer, 
  MV_FILTER_ONLY(orders, ARRAY['tiramisu']) AS orderItem,
  COUNT(*) AS numOrders
FROM "ristorante"
WHERE orders = 'tiramisu'
GROUP BY 1,2
```

![](/assets/2023-04-23-04.jpg)

You might be thinking that we can do without the `WHERE` clause, now that the filter is applied in the projection. Let's try it out:

```sql
SELECT 
  customer, 
  MV_FILTER_ONLY(orders, ARRAY['tiramisu']) AS orderItem,
  COUNT(*) AS numOrders
FROM "ristorante"
GROUP BY 1,2
```

![](/assets/2023-04-23-05.jpg)

Unfortunately, now the result set has rows even for customers that didn't order Tiramisu, and what is worse, they get a `numOrders` value of 1. You have to apply both filters in order to get the correct result.

## More complex filters

This is how Druid has always behaved. However, the new thing (and please forgive me that I got a bit confused about this among all the other things yesterday), is that we have now an option to enable this strict filtering in our Pivot front-end as well, and I just tested it with release imply-2023.03.01 on Hybrid.
If you filter by an MVD, there is now an additional checkbox “Hide filtered-out values” that enables the behavior you would like to achieve (see attached picture.)

Queries in both cases:

With the checkbox unchecked:

```sql
SELECT
(t."statesVisited") AS "statesVisited",
(COUNT(*) FILTER (WHERE t.recordType = 'click' AND t."state" = 'subscribe')) AS "COUNTFI-5a3"
FROM "imply-news" AS t
WHERE (((TIMESTAMP '2023-04-21 12:02:00'<=(t."__time") AND (t."__time")<TIMESTAMP '2023-04-21 13:02:00') AND MV_OVERLAP((t."statesVisited"), ARRAY['content'])) AND MV_OVERLAP((t."statesVisited"), ARRAY['content','plusContent','home','subscribe','clickbait','affiliateLink','exitSession']))
GROUP BY 1
```

With the checkbox checked:

```sql
SELECT
MV_FILTER_ONLY((t."statesVisited"), ARRAY['content']) AS "statesVisited",
(COUNT(*) FILTER (WHERE t.recordType = 'click' AND t."state" = 'subscribe')) AS "COUNTFI-5a3"
FROM "imply-news" AS t
WHERE ((TIMESTAMP '2023-04-21 11:47:00'<=(t."__time") AND (t."__time")<TIMESTAMP '2023-04-21 12:47:00') AND MV_OVERLAP((t."statesVisited"), ARRAY['content']))
GROUP BY 1
ORDER BY "COUNTFI-5a3" DESC
LIMIT 1000
```

## Learnings

- lorem ipsum
