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


We are using the `ristorante` datasource from [part 3](/2021/09/25/multivalue-dimensions-in-apache-druid-part-3/), and start with the analysis how many customers bought each item:

![](/assets/2021-10-03-1-groupby-orders_set.jpeg)

- If you apply a WHERE filter to a multi-value dimension (MVD), Druid selects all rows that contain any value that matches the expression, like I stated yesterday. For these rows, also all other MV items are included in the query result. This has always been the case and is documented in https://docs.imply.io/latest/druid/querying/multi-value-dimensions/#filtering.
- If you want the result to only contain the specific MV items that are in the filter clause, you can use the MV_FILTER_ONLY function: https://docs.imply.io/latest/druid/querying/sql-multivalue-string-functions/.

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
