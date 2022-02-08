---
layout: post
title:  "Druid Data Lab: Ingestion Transforms"
categories: blog druid imply ingestion tutorial
---

While [Apache Druid](https://druid.apache.org/) is not an ETL tool, it is possible to transform incoming data to do elementary cleaning or to derive dimension values from sets of input fields. Druid Transforms work on single rows of data. They operate on input fields and they create additional fields in the data. A Transform can overshadow existing field (dimension) names but you cannot use metrics nor other transforms as inputs.

The Druid documentation explais Transforms [here](https://druid.apache.org/docs/latest/ingestion/ingestion-spec.html#transforms). There's also a [tutorial](https://druid.apache.org/docs/latest/tutorials/tutorial-transform-spec.html#load-data-with-transform-specs), which is a good place to start if you haven't used transforms before.

## How to code Transforms

The simplest way to employ Transforms is via the ingestion wizard in the Druid console. Right after the `Parse time` step, you are greeted with the option to roll your own Transform:

![transform tab](/assets/2022-02-08-1-t.jpg)

As you hit `Add column transform`, you can define your transfrom using the [Druid expression syntax](https://druid.apache.org/docs/latest/misc/math-expr.html):

![define transform](/assets/2022-02-08-2-detail.jpg)

In the ingestion spec, Transforms are defined within the `transforms` section of the `dataSchema`. Here is how this might look like:

![transform spec](/assets/2022-02-08-3-spec.jpg)

One remark about quotes in expression syntax: It works much like in SQL. This means
- string literals are enclosed in single quotes: `'string literal'`
- identifiers such as field names, if required, are enclosed in double quotes: `"field name"`.

<mark>Since double quotes are also used as string delimiters in JSON, double quotes inside an expression have to be escaped like so: `\"field name\"`.</mark> This does not apply when you use the wizard - it will add the escape characters for you.

but we can actually do more!

## Simple math transformation

Imagine we have weather data from America that specifies the temperature in Fahrenheit degrees, `tempF`. As a European, I'd prefer to read the temperature in Celsius degrees.

Here is the formula

```json
{
  "type": "expression",
  "name": "tempC",
  "expression": "5.0 / 9.0 * (tempF - 32.0)"
}
```

Note how I have been careful to write all the number literals with a decimal point? Druid does make a difference between integer and floating point numbers, and it will perform arithmetic according to the types involved. If you write `5 / 9 * (tempF - 32)`, you will get all zeroes because an integer division of 5 by 9 yields (integer) 0!

## Compose a new dimension out of several fields

concatenate

## Composite timestamps

Druid is very good at parsing timestamps of various formats. But what if you have to pull the timestamp out of more than one field in your data?

The [documentation](https://druid.apache.org/docs/latest/ingestion/ingestion-spec.html#transforms) says:

> Transforms can refer to the timestamp of an input row by referring to `__time` as part of the expression. They can also replace the timestamp if you set their "name" to `__time`.

What does this mean? It means that the `__time` column value can be _overridden_ by anything you specify in an expression. 


Consider this data sample from an ADS-B data collector:

```csv
MT,TT,SID,AID,Hex,FID,DMG,TMG,DML,TML,CS,Alt,GS,Trk,Lat,Lng,VR,Sq,Alrt,Emer,SPI,Gnd
MSG,1,111,11111,3C70A8,111111,2016/02/05,04:51:22.709,2016/02/05,04:51:22.693,BCS850  ,,,,,,,,,,,0
MSG,8,111,11111,484411,111111,2016/02/05,04:51:22.746,2016/02/05,04:51:22.698,,,,,,,,,,,,0
MSG,8,111,11111,484411,111111,2016/02/05,04:51:22.755,2016/02/05,04:51:22.756,,,,,,,,,,,,0
MSG,5,111,11111,44D1CF,111111,2016/02/05,04:51:22.765,2016/02/05,04:51:22.758,,27025,,,,,,,0,,0,0
MSG,5,111,11111,3C70A8,111111,2016/02/05,04:51:22.837,2016/02/05,04:51:22.824,,36000,,,,,,,0,,0,0
MSG,5,111,11111,3C70A8,111111,2016/02/05,04:51:22.862,2016/02/05,04:51:22.827,,36000,,,,,,,0,,0,0
MSG,5,111,11111,44D1CF,111111,2016/02/05,04:51:22.893,2016/02/05,04:51:22.888,,27025,,,,,,,0,,0,0
MSG,5,111,11111,3C70A8,111111,2016/02/05,04:51:22.894,2016/02/05,04:51:22.888,,36000,,,,,,,0,,0,0
MSG,3,111,11111,484411,111111,2016/02/05,04:51:22.903,2016/02/05,04:51:22.889,,12525,,,52.58052,5.29031,,,,,,0
MSG,5,111,11111,44D1CF,111111,2016/02/05,04:51:22.916,2016/02/05,04:51:22.891,,27025,,,,,,,0,,0,0
MSG,5,111,11111,3C70A8,111111,2016/02/05,04:51:22.917,2016/02/05,04:51:22.891,,36000,,,,,,,0,,0,0
MSG,5,111,11111,44D1CF,111111,2016/02/05,04:51:22.932,2016/02/05,04:51:22.893,,27025,,,,,,,0,,0,0
MSG,5,111,11111,44D1CF,111111,2016/02/05,04:51:22.946,2016/02/05,04:51:22.895,,27025,,,,,,,0,,0,0
MSG,8,111,11111,A51EDE,111111,2016/02/05,04:51:22.967,2016/02/05,04:51:22.954,,,,,,,,,,,,0
MSG,7,111,11111,44D1CF,111111,2016/02/05,04:51:22.972,2016/02/05,04:51:22.955,,27025,,,,,,,,,,0
MSG,8,111,11111,44D1CF,111111,2016/02/05,04:51:23.040,2016/02/05,04:51:23.021,,,,,,,,,,,,0
MSG,6,111,11111,3C70A8,111111,2016/02/05,04:51:23.072,2016/02/05,04:51:23.025,,,,,,,,4131,0,0,0,0
MSG,3,111,11111,44D1CF,111111,2016/02/05,04:51:23.075,2016/02/05,04:51:23.026,,27025,,,51.32419,6.70715,,,,,,0
MSG,8,111,11111,44D1CF,111111,2016/02/05,04:51:23.220,2016/02/05,04:51:23.215,,,,,,,,,,,,0
MSG,8,111,11111,A51EDE,111111,2016/02/05,04:51:23.306,2016/02/05,04:51:23.284,,,,,,,,,,,,0
MSG,8,111,11111,44D1CF,111111,2016/02/05,04:51:23.356,2016/02/05,04:51:23.347,,,,,,,,,,,,0
MSG,8,111,11111,44D1CF,111111,2016/02/05,04:51:23.367,2016/02/05,04:51:23.348,,,,,,,,,,,,0
MSG,8,111,11111,484411,111111,2016/02/05,04:51:23.387,2016/02/05,04:51:23.351,,,,,,,,,,,,0
MSG,8,111,11111,44D1CF,111111,2016/02/05,04:51:23.402,2016/02/05,04:51:23.353,,,,,,,,,,,,0
MSG,5,111,11111,484411,111111,2016/02/05,04:51:23.406,2016/02/05,04:51:23.354,,12525,,,,,,,0,,0,0
```

The possible timestamp candidates are each spread across two columns for date and time. A transform definition of

```json
{
  "type": "expression",
  "name": "__time",
  "expression": "timestamp_parse(concat(\"DMG\", ' ', \"TMG\"), 'yyyy/M/d HH:mm:ss.SSS')"
}
```

can concatenate the date and time fields and parse them according to the custom format.

## Use a case expression inside a transform

...

## Parse a multi-value dimension

refer to the blog I already wrote about that

## use a temp array to do map-filter-reduce processing

- n-grams, this can be done using wikipedia data
- parse headline and remove stopwords


