---
layout: post
title:  "Processing Nested JSON Data and Kafka Metadata in Apache Druid"
categories: blog imply druid kafka json tutorial
---

[Druid 24.0](https://github.com/apache/druid/releases/tag/druid-24.0.0) brought the ability to [ingest and store nested (JSON) columns directly](https://druid.apache.org/docs/latest/querying/nested-columns.html). In simple cases, this would eliminate the need to [flatten JSON data during ingestion](https://druid.apache.org/docs/latest/ingestion/data-formats.html#flattenspec).

Inspired by real world experience, I am showing how column flattening and nested columns complement each other to enable some more sophisticated use cases of nested columns.

This tutorial uses [Aiven's pizza simulator](https://github.com/aiven/python-fake-data-producer-for-apache-kafka). I assume that you are running Kafka locally in a single broker configuration, and that you are producing data to a topic named `pizza`.

The tutorial uses [Druid 24.0 quickstart](https://druid.apache.org/docs/latest/tutorials/index.html).

ingestion spec

```json
{
  "type": "kafka",
  "spec": {
    "ioConfig": {
      "type": "kafka",
      "consumerProperties": {
        "bootstrap.servers": "localhost:9092"
      },
      "topic": "pizza",
      "inputFormat": {
        "type": "json",
        "flattenSpec": {
          "fields": [
            {
              "type": "root",
              "name": "pizzas"
            },
            {
              "type": "jq",
              "expr": "[.pizzas[].pizzaName]",
              "name": "pizzaNames"
            },
            {
              "type": "jq",
              "expr": "[.pizzas[].additionalToppings]",
              "name": "additionalToppings"
            },
            {
              "type": "jq",
              "name": "pizzasMV",
              "expr": "[.pizzas[]|tostring]"
            }
          ]
        }
      },
      "useEarliestOffset": true,
      "taskDuration": "PT1H",
      "appendToExisting": false
    },
    "tuningConfig": {
      "type": "kafka"
    },
    "dataSchema": {
      "dataSource": "pizza",
      "timestampSpec": {
        "column": "timestamp",
        "format": "millis"
      },
      "granularitySpec": {
        "queryGranularity": "none",
        "rollup": false,
        "segmentGranularity": "day"
      },
      "dimensionsSpec": {
        "dimensions": [
          "shop",
          {
            "type": "json",
            "name": "pizzas"
          },
          {
            "type": "long",
            "name": "id"
          },
          "name",
          "phoneNumber",
          "address",
          {
            "type": "string",
            "name": "pizzaNames",
            "multiValueHandling": "ARRAY"
          },
          {
            "type": "string",
            "multiValueHandling": "ARRAY",
            "name": "pizzasMV"
          }
        ]
      }
    }
  }
}
```


get out all the individual pizzas

```sql
WITH pizza_lines AS (
  SELECT id, shop, TO_JSON_STRING(pizzas) AS order_total, name, pizzasMV AS order_item
  FROM pizza
  GROUP BY 1, 2, 3, 4, 5
)
SELECT
  id,
  shop, 
  order_total, 
  name, 
  JSON_VALUE(PARSE_JSON(order_item), '$.pizzaName') AS order_item_pizza,
  JSON_QUERY(PARSE_JSON(order_item), '$.additionalToppings') AS order_item_additionalToppings
FROM pizza_lines
```
