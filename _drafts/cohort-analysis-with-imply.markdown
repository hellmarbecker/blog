---
layout: post
title:  "Cohort Analysis with Imply"
categories: blog druid imply pivot analytics tutorial gaming
---

```sql
REPLACE INTO cohort OVERWRITE ALL
WITH source AS (SELECT * FROM TABLE(
  EXTERN(
    '{"type":"s3","uris":["s3://imply-cloud-sales-data/valsea/cohort.csv"]}',
    '{"type":"csv","findColumnsFromHeader":true,"skipHeaderRows":0}',
    '[{"name":"curdate","type":"string"},{"name":"playerid","type":"string"},{"name":"signupdate","type":"string"},{"name":"deposit","type":"double"},{"name":"signupdate_long","type":"long"}]'
  )
))
SELECT
  TIME_PARSE(curdate) AS __time,
  playerid,
  signupdate,
  deposit
FROM source
PARTITIONED BY MONTH
CLUSTERED BY playerid
```
