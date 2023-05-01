---
layout: post
title:  "Druid Sneak Peek: Schema Inference and Arrays"
categories: blog apache druid imply iot sql tutorial
---

One of the strong points of Druid has always been [built-in schema evolution](/2021/08/13/experiments-with-schema-evolution-in-apache-druid/). However, upon getting data of changing shape into Druid, you had two choices:

- either, specify each field with its type in the ingestion spec, which requires to know all the fields ahead of time
- or pick up whatever comes in using [schemaless ingestion](https://druid.apache.org/docs/latest/ingestion/schema-design.html#schema-less-dimensions), wich the downside that any dimension ingested that way would be interpreted as a string.

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

The `dimensionsSpec` has an empty dimension list, but there is a new flag `useSchemaDiscovery`:

```json
      "dimensionsSpec": {
        "useSchemaDiscovery": true,
        "includeAllDimensions": true,
        "dimensionExclusions": []
      }
```

