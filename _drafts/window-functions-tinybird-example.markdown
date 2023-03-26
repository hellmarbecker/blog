build druid 26 snapshot from master

disclaimer

this is still under development so it is hidden behind a secret query context option

This is a sneak peek into Druid 26 functionality. In order to use the new functions, you can (as of the time of writing) [build Druid](https://druid.apache.org/docs/latest/development/build.html) from the HEAD of the master branch:

```bash
git clone https://github.com/apache/druid.git
cd druid
mvn clean install -Pdist -DskipTests
```

Then follow the instructions to locate and install the tarball.

_**Disclaimer:** This tutorial uses undocumented functionality and unreleased code. This blog is neither endorsed by Imply nor by the Apache Druid PMC. It merely collects the results of personal experiments. The features described here might, in the final release, work differently, or not at all. In addition, the entire build, or execution, may fail. Your mileage may vary._

## now let's do it in practice

blog page: https://www.tinybird.co/blog-posts/coming-soon-on-clickhouse-window-functions

data is in: https://storage.googleapis.com/tinybird-assets/datasets/guides/events_10K.csv

let's see if we can do some stuff with this

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
