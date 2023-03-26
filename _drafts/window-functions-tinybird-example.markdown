[Great changes have been announced for the upcoming Druid 26.0 release.](https://www.linkedin.com/feed/update/urn:li:activity:7043593237915148288/) The one that excites me the most is the introduction of [window functions](https://github.com/paul-rogers/druid/wiki/Window-Functions).

Window functions allow a query to interrelate and aggregate rows beyond a simple `GROUP BY`. [Previously](/2022/11/05/druid-data-cookbook-cumulative-sums-in-druid-sql/), I have looked at ways to emulate such processing patterns using self joins or grouping sets in Druid. But now, we are close to getting window functions as first class citizens.

This is a sneak peek into Druid 26 functionality. In order to use the new functions, you can (as of the time of writing) [build Druid](https://druid.apache.org/docs/latest/development/build.html) from the HEAD of the master branch:

```bash
git clone https://github.com/apache/druid.git
cd druid
mvn clean install -Pdist -DskipTests
```

Then follow the instructions to locate and install the tarball.

All this is still under development so it is undocumented, and hidden behind a secret query context option. I will show that in a moment. Also notice, that window functions only work within `GROUP BY` queries, and there are still some other limitations. But this is fast progressing.

_**Disclaimer:** This tutorial uses undocumented functionality and unreleased code. This blog is neither endorsed by Imply nor by the Apache Druid PMC. It merely collects the results of personal experiments. The features described here might, in the final release, work differently, or not at all. In addition, the entire build, or execution, may fail. Your mileage may vary._

## Let's do it in practice

I am taking a data sample from [the Tinybird blog](https://www.tinybird.co/blog-posts/coming-soon-on-clickhouse-window-functions) which is simulated data from an ecommerce store. The data is downloadable from https://storage.googleapis.com/tinybird-assets/datasets/guides/events_10K.csv and has a straightforward format:

- a _timestamp_
- string fields for _product id, user_id,_ and _event type_
- an _extra data_ field: this is a variable JSON object whose schema depends on the event type.

Let's see if we can do some interesting things with this!

ingest:

```sql
REPLACE INTO "events" OVERWRITE ALL
WITH "ext" AS (SELECT *
FROM TABLE(
  EXTERN(
    '{"type":"http","uris":["https://storage.googleapis.com/tinybird-assets/datasets/guides/events_10K.csv"]}',
    '{"type":"csv","findColumnsFromHeader":false,"columns":["date","product_id","user_id","event","extra_data"]}',
    '[{"name":"date","type":"string"},{"name":"product_id","type":"string"},{"name":"user_id","type":"long"},{"name":"event","type":"string"},{"name":"extra_data","type":"string"}]'
  )
))
SELECT
  TIME_PARSE("date") AS "__time",
  "product_id",
  "user_id",
  "event",
  PARSE_JSON("extra_data") AS "extra_data"
FROM "ext"
PARTITIONED BY MONTH
```

let's get an idea of the amount of data in there, this is one of the neat things in druid console, it has these things available in a menu

![](/assets/2023-03-26-01-selectminmaxtime.jpg)

this gives us a quick query

```sql
SELECT
  MIN("__time") AS "min_time",
  MAX("__time") AS "max_time"
FROM "events"
GROUP BY ()
```

which shows data is from more than 3 years. 

look at the data with a select *:

![]()



query

```

```
