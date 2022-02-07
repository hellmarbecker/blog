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

Consider this data sample from an ADS-B data collector:
```
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
The possible timestamp candidates are each spread across two columns for date and time.

## Parse a MVD

refer to the blog I already wrote about that

## use a temp array to do map-filter-reduce processing

- n-grams, this can be done using wikipedia data
- parse headline and remove stopwords


