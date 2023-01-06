---
layout: post
title:  "Druid Data Cookbook: Modeling Flag Lists"
categories: blog apache druid imply tutorial
---
![Druid Cookbook](/assets/2021-12-21-elf.jpg)

## Flags in a nested object

lorem ipsum

## With a completely flat structure

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
