blog page: https://www.tinybird.co/blog-posts/coming-soon-on-clickhouse-window-functions

data is in: https://storage.googleapis.com/tinybird-assets/datasets/guides/events_10K.csv

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
PARTITIONED BY DAY
```


query

```

```
