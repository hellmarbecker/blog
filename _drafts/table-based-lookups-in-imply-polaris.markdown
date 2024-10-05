---
layout: post
title:  "Table Based Lookups in Imply Polaris"
categories: blog imply sql datamodel warehouse tutorial
twitter:
  image: /assets/2021-12-21-elf.jpg
---

Imply's [Polaris](https://imply.io/imply-fully-managed-dbaas-polaris/) service has a new feature: [Lookups](https://docs.imply.io/polaris/lookups/). Lookups in Polaris provide a convenient way to model dimension tables in a star schema, and they are more flexible and less resource hungry than the lookups you know from open source Druid. Let's take a closer look!

## Recap: Lookups in Apache Druid

yada yada

I wrote about this in [an earlier blog](/2021/10/14/druid-data-modeling-special-lookups-and-multi-value-dimensions/)

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
