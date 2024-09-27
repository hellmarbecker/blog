---
layout: post
title:  "Druid Data Cookbook: Deconstructing Nested JSON Objects"
categories: blog apache druid imply sql tutorial
twitter:
  image: /assets/2021-12-21-elf.jpg
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)

In [the previous episode of the Druid Data Cookbook](/2024/09/22/druid-data-cookbook-flattening-arrays-of-complex-objects/), I showed how to extract and process all elements out of nested array of objects in Druid. But what if the elements you want to process are scattered over different levels of the object hierarchy? By extending the `UNNEST` paradigm, we can handle these cases too!

## A data sample

Our data looks like this:

```json
{
  "timestamp": "2024-09-01",
  "id": "1",
  "org": {
    "name": "Team 1",
    "members": [
      {
        "name": "Alice",
        "gender": "F"
      },
      {
        "name": "Bob",
        "gender": "M"
      },
      {
        "name": "Carol",
        "gender": "F"
      }
    ]
  }
}
```

Note how there are `name` fields at the first nesting leel of `org`, but also at the `members`level.

Here's the full dataset in [jsonl](https://www.atatus.com/glossary/jsonl/) format:

```json
{ "timestamp": "2024-09-01", "id": "1", "org": { "name": "Team 1", "members": [ { "name": "Alice", "gender": "F" }, { "name": "Bob", "gender": "M" }, { "name": "Carol", "gender": "F" } ] } }
{ "timestamp": "2024-09-01", "id": "2", "org": { "name": "Team 2", "members": [ { "name": "Dan", "gender": "M" }, { "name": "Eve", "gender": "F" }, { "name": "Frank", "gender": "M" } ] } }
```

Load this data sample into Druid (version 30 or higher.) It should be easy to ingest using the wizard, or you can submit this query:

```sql
REPLACE INTO "teams_nested" OVERWRITE ALL
WITH "ext" AS (
  SELECT *
  FROM TABLE(
    EXTERN(
      '{"type":"inline","data":"{ \"timestamp\": \"2024-09-01\", \"id\": \"1\", \"org\": { \"name\": \"Team 1\", \"members\": [ { \"name\": \"Alice\", \"gender\": \"F\" }, { \"name\": \"Bob\", \"gender\": \"M\" }, { \"name\": \"Carol\", \"gender\": \"F\" } ] } }\n{ \"timestamp\": \"2024-09-01\", \"id\": \"2\", \"org\": { \"name\": \"Team 2\", \"members\": [ { \"name\": \"Dan\", \"gender\": \"M\" }, { \"name\": \"Eve\", \"gender\": \"F\" }, { \"name\": \"Frank\", \"gender\": \"M\" } ] } }"}',
      '{"type":"json"}'
    )
  ) EXTEND ("timestamp" VARCHAR, "id" VARCHAR, "org" TYPE('COMPLEX<json>'))
)
SELECT
  TIME_PARSE(TRIM("timestamp")) AS "__time",
  "id",
  "org"
FROM "ext"
PARTITIONED BY DAY
```

## Extracting the leaf paths

Now, because the data does not neatly come in an array, we cannot extract elements at a specified level. But we can do another trick: we can obtain an array of the [JSONPath](https://www.baeldung.com/guide-to-jayway-jsonpath)s of all leaf elements in an object by calling `JSON_PATHS`. Run this query:

```sql
SELECT
  t.__time,
  t.id,
  t.org,
  x.leaf_path
FROM "teams_nested" t CROSS JOIN UNNEST(JSON_PATHS(org)) x("leaf_path")
```

It returns one row for each leaf element in the `org` object. 

![UNNEST query](/assets/2024-09-27-01.png)

In the next step, let's feed these values back into `JSON_VALUE`.

(This will really only work with Druid 30 or better. In earler versions, `JSON_PATH` required a string literal as its second element, limiting its flexibility.)

## Some query examples

We can use the above query as a CTE and do further processing in the main query.

Get all `name` fields, regardless of the hierarchy level, using a `LIKE` filter:

```sql
WITH cte AS (
  SELECT
    t.__time,
    t.id,
    t.org,
    x.leaf_path
  FROM "teams_nested" t CROSS JOIN UNNEST(JSON_PATHS(org)) x("leaf_path")
)
SELECT
  __time,
  id,
  leaf_path,
  JSON_VALUE(org, "leaf_path")
FROM cte
WHERE leaf_path LIKE '%.name'
```

![All names](/assets/2024-09-27-02.png)

Or put all those names back into an array per team with an array aggregator:

```sql
WITH cte AS (
  SELECT
    t.__time,
    t.id,
    t.org,
    x.leaf_path
  FROM "teams_nested" t CROSS JOIN UNNEST(JSON_PATHS(org)) x("leaf_path")
)
SELECT
  __time,
  id,
  ARRAY_AGG(JSON_VALUE(org, "leaf_path"))
FROM cte
WHERE leaf_path LIKE '%.name'
GROUP BY __time, id
```

![All names, array](/assets/2024-09-27-03.png)

You can do arbitrary gymnastics on the JSONPath expressions using regular expression filters. Here I use this technique to emulate a JSONPath expression like `"$.members[*].gender"`:

```sql
WITH cte AS (
  SELECT
    t.__time,
    t.id,
    t.org,
    x.leaf_path
  FROM "teams_nested" t CROSS JOIN UNNEST(JSON_PATHS(org)) x("leaf_path")
)
SELECT
  __time,
  id,
  leaf_path,
  JSON_VALUE(org, "leaf_path")
FROM cte
WHERE REGEXP_LIKE(leaf_path, 'members\[\d+\]\.gender$')
```

![JSON wildcard emulation with regex](/assets/2024-09-27-04.png)

## Conclusion

- By unnesting the result of `JSON_PATHS`, you get access to the entire structure of a nested (JSON) object in Druid.
- In conjunction with regular expression filters, you get processing capabilities that are almost as powerful as `jq`.

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
