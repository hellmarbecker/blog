---
layout: post
title:  "Processing Nested JSON Data and Kafka Metadata in Apache Druid"
categories: blog imply druid kafka json tutorial
---

[Druid 24.0](https://github.com/apache/druid/releases/tag/druid-24.0.0) brought the ability to [ingest and store nested (JSON) columns directly](https://druid.apache.org/docs/latest/querying/nested-columns.html). In most cases, this would eliminate the need to [flatten JSON data during ingestion](https://druid.apache.org/docs/latest/ingestion/data-formats.html#flattenspec).

Inspired by real world experience, I am showing how to work with JSON columns and Kafka timestamps in the same ingestion spec, and how column flattening and nested columns complement each other to enable this use case.

This tutorial uses [Aiven's pizza simulator](https://github.com/aiven/python-fake-data-producer-for-apache-kafka). I assume that you are running Kafka locally in a single broker configuration, and that you are producing data to a topic named `pizza`.

The tutorial uses [Druid 24.0 quickstart](https://druid.apache.org/docs/latest/tutorials/index.html).

## The Data

The data generator simulates pizza orders. Here's a sample record:

```json
{
  "id": 14492,
  "shop": "Mammamia Pizza",
  "name": "Juan Gonzalez",
  "phoneNumber": "5474461494",
  "address": "462 Williams Tunnel Apt. 424\nNicoletown, WY 02258",
  "pizzas": [
    {
      "pizzaName": "Mari & Monti",
      "additionalToppings": [
        "🍅 tomato",
        "🧅 onion"
      ]
    },
    {
      "pizzaName": "Diavola",
      "additionalToppings": []
    },
    {
      "pizzaName": "Salami",
      "additionalToppings": [
        "🧄 garlic",
        "🍅 tomato",
        "🧅 onion",
        "🥚 egg"
      ]
    },
    {
      "pizzaName": "Margherita",
      "additionalToppings": [
        "🍅 tomato",
        "🍓 strawberry",
        "🍍 pineapple",
        "🐟 tuna"
      ]
    }
  ],
  "timestamp": 1669202722766
}
```

So each order record has an `id` field, some information about the customer, and an array of pizzas ordered. Each pizza has a `pizzaName` and an array of `additionalToppings`.

Let's first do an ordinary ingestion using the ingest wizard. Note how the `pizzas` field does not appear in the list in `Parse data`:

![Parse Kafka fields](/assets/2022-11-23-01-parse.jpg)

Don't worry, we will fix that in a moment. Proceed to `Configure schema`, accepting the defaults.

Add a new dimension:

![Add dimension](/assets/2022-11-23-02-add-dim.jpg)

and choose the `pizzas` field and type `json`:

![Configure JSON dimension](/assets/2022-11-23-03-json-dim.jpg)

In the following steps, choose daily segments and name the datasource `pizza-01`.

Here is the ingestion spec:

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
        "type": "json"
      },
      "useEarliestOffset": true
    },
    "tuningConfig": {
      "type": "kafka"
    },
    "dataSchema": {
      "dataSource": "pizza-01",
      "timestampSpec": {
        "column": "timestamp",
        "format": "millis"
      },
      "dimensionsSpec": {
        "dimensions": [
          {
            "type": "long",
            "name": "id"
          },
          "shop",
          "name",
          "phoneNumber",
          "address",
          {
            "type": "json",
            "name": "pizzas"
          }
        ]
      },
      "granularitySpec": {
        "queryGranularity": "none",
        "rollup": false,
        "segmentGranularity": "day"
      }
    }
  }
}
```

Let's query the data:

![Query the simple data](/assets/2022-11-23-04-query-simple.jpg)

The `pizzas` column has been ingested correctly. The tree symbol next to the column shows that it is a nested JSON column and you can query it further with [JSON functions](https://druid.apache.org/docs/latest/querying/sql-json-functions.html).


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

```
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
            },
            {
              "type": "jq",
              "name": "pizzasToppingsMV",
              "expr": "[.pizzas[]|{pizzaName, additionalToppings: .additionalToppings|tostring}]"
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
          },
          {
            "type": "string",
            "name": "pizzasToppingsMV",
            "multiValueHandling": "ARRAY"
          }
        ]
      }
    }
  }
}
```
