---
layout: post
title:  "Druid Sneak Peek: Schema Inference and Arrays"
categories: blog apache druid imply iot sql tutorial
---

One of the strong points of Druid has always been [built-in schema evolution](/2021/08/13/experiments-with-schema-evolution-in-apache-druid/). However, upon getting data of changing shape into Druid, you had two choices:

- either, specify each field with its type in the ingestion spec, which requires to know all the fields ahead of time
- or pick up whatever comes in using [schemaless ingestion](https://druid.apache.org/docs/latest/ingestion/schema-design.html#schema-less-dimensions), with the downside that any dimension ingested that way would be interpreted as a string.

The good news is that this is going to change. Druid 26 is going to come with the ability to infer its schema completely from the input data, and even ingest structured data automatically.

_**Disclaimer:** This tutorial uses undocumented functionality and unreleased code. This blog is neither endorsed by Imply nor by the Apache Druid PMC. It merely collects the results of personal experiments. The features described here might, in the final release, work differently, or not at all. In addition, the entire build, or execution, may fail. Your mileage may vary._

Druid 26 hasn't been released yet, but you can [build Druid](https://druid.apache.org/docs/latest/development/build.html) from the master branch of the repository and try out the new features.

I am going to pick up the [multi-value dimensions example from last week](/2023/04/23/multivalue-dimensions-in-apache-druid-part-5/), but this time I want you to get an idea how these types of scenarios are going to be handled in the future. We are going to:

- ingest data using the new schema discovery feature
- ingest structured data into an SQL ARRAY
- show how `GROUP BY` and lateral joins work with that array. 

## Ingestion: Schema Inference

We are using the `ristorante` dataset that you can find [here](/2021/09/25/multivalue-dimensions-in-apache-druid-part-3/), but with a little twist: On the `Configure schema` tab, uncheck `Explicitly specify dimension list`.

![Set autodetect](/assets/2023-05-01-01-autodetect.jpg)

Confirm the warning dialog that pops up, and continue modeling the data. When you proceed to the `Edit spec` stage, you can see a new setting that slipped in:

![Autodetect](/assets/2023-05-01-02-useSchemaDiscovery.jpg)

The `dimensionsSpec` has no dimension list now, but there is a new flag `useSchemaDiscovery`:

```json
      "dimensionsSpec": {
        "useSchemaDiscovery": true,
        "includeAllDimensions": true,
        "dimensionExclusions": []
      }
```

## Querying the data

Let's look at the resulting data with a simple `SELECT *` query:

![Select all](/assets/2023-05-01-03-select-trueArray.jpg)

Notice how Druid has automatically detected that `orders` is an array of primitives (strings, in this case.) In older versions, this would have been either a multi-value string. But now, Druid has true `ARRAY` columns!

(In the more general case of nested objects, Druid would have generated a nested JSON column.)

In order to take the arrays apart, we can once again make use of the `UNNEST` function. This has to be enabled using a query context flag. In the console, use the `Edit context` function inside the query engine menu

<img src="/assets/2023-05-01-04-editcontext.jpg" width="40%" />

and enter the context:

```json
{
  "enableUnnest": true
}
```

In the REST API, you can pass the context directly.

Then, unnest and group the items:

```sql
SELECT 
  order_item, 
  COUNT(*) AS order_count
FROM "ristorante_auto", UNNEST(orders) AS t(order_item)
GROUP BY 1
```

![Select groupby](/assets/2023-05-01-05-groupby.jpg)

Once you have done this, you can filter by individual order items and you don't have all the quirks that we talked about when doing multi-value dimensions:

```sql
SELECT
  customer,
  order_item, 
  COUNT(*) AS order_count
FROM "ristorante_auto", UNNEST(orders) AS t(order_item)
WHERE order_item = 'tiramisu'
GROUP BY 1, 2
```

![Filtered groupby](/assets/2023-05-01-06-filter.jpg)

## Conclusion

- Druid can now do schema inference.
- It can automatically detect primitive types, but also nested objects and arrays of primitives.
- Typical Druid queries that would use multi-value dimensions in the past can now be done in a more standard way using array columns and `UNNEST`.
