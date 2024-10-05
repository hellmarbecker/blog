---
layout: post
title:  "Table Based Lookups in Imply Polaris"
categories: blog imply sql datamodel warehouse tutorial
---

Imply's [Polaris](https://imply.io/imply-fully-managed-dbaas-polaris/) service has a new feature: [Lookups](https://docs.imply.io/polaris/lookups/). Lookups in Polaris provide a convenient way to model dimension tables in a star schema, and they are more flexible and less resource hungry than the lookups you know from open source Druid. Let's take a closer look!

## Recap: Lookups in Apache Druid

[Druid lookups](https://druid.apache.org/docs/latest/querying/lookups) are cached key-value maps that are kept in memory on all historical and peon processes. I wrote about them in [an earlier blog](/2021/10/14/druid-data-modeling-special-lookups-and-multi-value-dimensions/). A lookup can be queried like a table, but it has only two columns - key _k_ and value _v_. If you want to lookup multiple values for a key - say, a customer's name, stret address, city, and country - then you need to create a separate lookup for each.

Lookups can be populated in various ways: 

- you can upload the map data directly via the API or the Druid console
- or you can populate a lookup from [a file described by a URL](https://druid.apache.org/docs/latest/querying/lookups-cached-global#uri-lookup)
- or from [a Kafka topic](https://druid.apache.org/docs/latest/querying/kafka-extraction-namespace) 
- or from [a database table](https://druid.apache.org/docs/latest/querying/lookups-cached-global#jdbc-lookup).

The last option is particularly interesting because the source of the lookup can also be a regular Druid table, if you specify Druid's Avatica driver in the JDBC URL. This works but is kind of a hack, and has limited flexibility. But it is how we used to set up lookups in Polaris in the backend, before they were officially supported.

## Lookups based on Druid Segments

this is available only in Polaris

big advantage: any (string) column can be either key or value. also more memory efficient

## How to do it in practice

model the dimension table
- has to be partition by all
- has to be all string columns
- has to fit in a single segment

model the fact table
- make sure the (foreign) key column is a string too or else you will need a cast in the lookup

create the lookup in Polaris GUI

## How to visualize lookup data in Pivot

- use the `LOOKUP(..., 'lookup_name[key_column][value_column]')` in dimension definitions
- unlike a regular Kimball model, you can also model measures by aggregating over a LOOKUP expression
