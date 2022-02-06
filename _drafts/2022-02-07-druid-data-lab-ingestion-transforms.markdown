---
layout: post
title:  "Druid Data Lab: Ingestion Transforms"
categories: blog druid imply ingestion tutorial
---

lorem ipsum 

what are ingestion transforms yada yada

documentation [here](https://druid.apache.org/docs/latest/ingestion/ingestion-spec.html#transforms)

note, transforms operate on dimensions as inputs, they can overshadow exiisting dimension names but you cannot use metrics nor other transforms as inputs

some simple examples in the [tutorial](https://druid.apache.org/docs/latest/tutorials/tutorial-transform-spec.html#load-data-with-transform-specs)

but we can actually do more!

## Simple math transformation

do a conversion from F to C or vice versa

## Compose a new dimension out of several fields

concatenate

## Composite timestamps

this is actually documented: what if you have to pull the timestamp out of more than one field in your data?

## Parse a MVD

refer to the blog I already wrote about that

## use a temp array to do map-filter-reduce processing

- n-grams, this can be done using wikipedia data
- parse headline and remove stopwords


