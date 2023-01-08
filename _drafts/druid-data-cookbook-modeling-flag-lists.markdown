---
layout: post
title:  "Druid Data Cookbook: Modeling Flag Lists"
categories: blog apache druid imply tutorial
---
![Druid Cookbook](/assets/2021-12-21-elf.jpg)

In timeseries data, you may have lists of boolean flags that come with each data record. Running a Druid ingestion on that, you would create a large number of dimensions. If you have followed the [Druid Data Modeling class](https://learn.imply.io/apache-druid-ingestion-and-data-modeling), you know that reducing the number of dimensions is one goal of optimizing the data model in Druid.

This is one of the prime use cases for [multi-value dimensions](https://blog.hellmar-becker.de/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/).

Rather than having one dimension per flag, create one multi-value dimension for all flags. So

## Generating Sample Data

Let's generate ourselves some fake timeseries data. The data format is simple:

- a timestamp
- a quantity and value, where the quantity is a single random letter
- various boolean flags that may or may not conform to the naming convention "f" plis any number of digits.

Here's a small python script for generating the data, you will need the [Faker](https://pypi.org/project/Faker/) module installed:

```python
import json
import time
from faker import Faker

maxRecords = 20000
fake = Faker()
flagsOrNot = [ "f1", "f2", "f3", "f10", "f1f", "ff1", "abc" ];

def main():

    ts = time.time() - 10 * 86400 # 10 days back
    for id in range(maxRecords):

        emitRecord = {
            "timestamp" : int(round(ts)),
            "quantity" : fake.random_element(elements=('a', 'b', 'c', 'd')),
            "value" : fake.random_number(digits=3) / 100.0,
        }
        emitRecord.update( {
            k : fake.boolean() for k in flagsOrNot
        } )

        print(json.dumps(emitRecord))
        ts += fake.random_int(min=1, max=60)

if __name__ == "__main__":
    main()
```

Run the script and save the output in a file named `flags.json`.

## Extracting the flag data with `jq`

We want to select only those fields from the JSON object that have names that start with "f" and have only digits after that. Let's break this down into steps.

- First, I use the `to_entries` function to transform the JSON object into an array of objects that have fields `key` and `value`.
 
  So, `{ "a" : "1" }` becomes `[ { "key" : "a", "value" : "1" } ]`. 
  
- The first `map` statement selects only those keys that match the regular expression for the flag name ("f" + any number of digits, but nothing else).
- The result is filtered by a `select` statement that keeps only the entries where the `value` evaluates to `true`.
- The final `map` statement keeps only the keys for the matching entries and returns them in an array.

This is the filter phrase as you would use it with `jq`:

```
to_entries | map(select(.key | match("^f\\d+$")) | select(.value)) | map(.key)
```

## Approach 1: Preprocessing

One way of ingesting the flags as a multi-value dimension is preprocessing. Let's just add a `flags` field to each JSON object:

```bash
jq -c '. += { "flags": to_entries | map(select(.key | match("^f\\d+$")) | select(.value)) | map(.key) }' < flags.json
```

This does the trick and it may be good enough. But let's look at some more elegant solutions.

## Approach 2: Using `flattenSpec`

data sample:

```
{ "timestamp" : "2023-01-05",  "lorem" : "ipsum",  "f1" : true,  "f2" : false,  "f3" : true, "sf3" : true,  "ffb" : true }
```

```json
{
  "type": "index_parallel",
  "spec": {
    "ioConfig": {
      "type": "index_parallel",
      "inputSource": {
        "type": "inline",
        "data": "{   \"timestamp\" : \"2023-01-05\",  \"lorem\" : \"ipsum\",  \"f1\" : true,  \"f2\" : false,  \"f3\" : true, \"sf3\" : true,  \"ffb\" : true }"
      },
      "inputFormat": {
        "type": "json",
        "flattenSpec": {
          "useFieldDiscovery": true,
          "fields": [
            {
              "type": "path",
              "name": "alldata",
              "expr": "$"
            }
          ]
        }
      }
    },
    "tuningConfig": {
      "type": "index_parallel",
      "partitionsSpec": {
        "type": "dynamic"
      }
    },
    "dataSchema": {
      "dataSource": "inline_data",
      "timestampSpec": {
        "column": "!!!_no_such_column_!!!",
        "missingValue": "2010-01-01T00:00:00Z"
      },
      "transformSpec": {
        "transforms": [
          {
            "type": "expression",
            "expression": "json_value(root, '$.lorem')",
            "name": "tsl"
          }
        ]
      },
      "granularitySpec": {
        "queryGranularity": "none",
        "rollup": false
      },
      "dimensionsSpec": {
        "dimensions": [
          {
            "name": "alldata",
            "type": "json"
          },
          "timestamp",
          "lorem",
          "f1",
          "f2",
          "f3",
          "sf3",
          "ffb",
          "tsl"
        ]
      }
    }
  }
}
```

## Conclusion

lorem ipsum

- ...
- ...

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
