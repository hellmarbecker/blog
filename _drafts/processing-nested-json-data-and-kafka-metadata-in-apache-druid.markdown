---
layout: post
title:  "Processing Nested JSON Data and Kafka Metadata in Apache Druid"
categories: blog imply druid kafka json tutorial
---

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
