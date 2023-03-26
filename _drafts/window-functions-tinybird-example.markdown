build druid 26 snapshot from master

disclaimer

this is still under development so it is hidden behind a secret query context option

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
