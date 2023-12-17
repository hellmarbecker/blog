---
layout: post
title:  "Druid 29 Preview: Handling Nested Arrays"
categories: blog apache druid imply sql tutorial
twitter:
  image: /assets/2022-11-23-00-pizza.jpg
---

![Pizza](/assets/2022-11-23-00-pizza.jpg)

Imagine you have a data sample like this:

```json-nd
{'id': 93, 'shop': 'Circular Pi Pizzeria', 'name': 'David Murillo', 'phoneNumber': '305-351-2631', 'address': '746 Chelsea Plains Suite 656\nNew Richard, MA 16940', 'pizzas': [{'pizzaName': 'Salami', 'additionalToppings': ['🥓 bacon']}], 'timestamp': 1702815411410}
{'id': 94, 'shop': 'Marios Pizza', 'name': 'Darius Roach', 'phoneNumber': '344.571.9608x0590', 'address': '58235 Robert Cliffs\nAguilarland, PR 76249', 'pizzas': [{'pizzaName': 'Diavola', 'additionalToppings': []}, {'pizzaName': 'Salami', 'additionalToppings': ['🧄 garlic']}, {'pizzaName': 'Peperoni', 'additionalToppings': ['🫒 olives', '🧅 onion', '🍅 tomato', '🍓 strawberry']}, {'pizzaName': 'Diavola', 'additionalToppings': ['🫒 olives', '🍌 banana', '🍍 pineapple']}, {'pizzaName': 'Margherita', 'additionalToppings': ['🍓 strawberry', '🍍 pineapple', '🥚 egg', '🐟 tuna', '🐟 tuna']}, {'pizzaName': 'Margherita', 'additionalToppings': ['🥚 egg']}, {'pizzaName': 'Margherita', 'additionalToppings': ['🫑 green peppers', '🥚 egg', '🥚 egg']}, {'pizzaName': 'Peperoni', 'additionalToppings': []}, {'pizzaName': 'Salami', 'additionalToppings': []}], 'timestamp': 1702815415518}
{'id': 95, 'shop': 'Mammamia Pizza', 'name': 'Ryan Juarez', 'phoneNumber': '(041)278-5690', 'address': '934 Melissa Lights\nPaulland, UT 40700', 'pizzas': [{'pizzaName': 'Marinara', 'additionalToppings': ['🫑 green peppers', '🧅 onion']}, {'pizzaName': 'Marinara', 'additionalToppings': ['🍅 tomato', '🥓 bacon', '🍌 banana', '🌶️ hot pepper']}, {'pizzaName': 'Peperoni', 'additionalToppings': ['🍓 strawberry', '🍌 banana', '🐟 tuna', '🧀 blue cheese']}, {'pizzaName': 'Marinara', 'additionalToppings': ['🐟 tuna', '🧅 onion', '🍍 pineapple', '🍓 strawberry']}, {'pizzaName': 'Mari & Monti', 'additionalToppings': ['🫒 olives', '🐟 tuna']}, {'pizzaName': 'Marinara', 'additionalToppings': ['🍍 pineapple', '🍅 tomato', '🍌 banana', '🧀 blue cheese', '🫒 olives']}, {'pizzaName': 'Marinara', 'additionalToppings': ['🍌 banana', '🫑 green peppers', '🧄 garlic', '🍅 tomato']}], 'timestamp': 1702815418643}
```

I created the data sample using [Francesco's pizza simulator](https://github.com/Aiven-Labs/python-fake-data-producer-for-apache-kafka). The structure of these simulated pizza orders is quite deeply nested:

- Each order has a field `pizzas`, which is an array of JSON objects.
- Each individual pizza item has
  - a `pizzaName` field, which is a string
  - `additionalToppings`, an array of strings that may be empty.

Arrays of objects are a bit obtuse, and I would like to create a data model that breaks down the orders so that each row in Druid represents a line item (a single pizza.)
To that end, it would be nice to use some combination of JSON functions and [`UNNEST`](/2023/04/08/druid-sneak-peek-timeseries-interpolation/) during ingestion. But how exactly? Let's find out!
 
## Getting set up

This is a sneak peek into Druid 29 functionality. In order to use the new functions, you can (as of the time of writing) [build Druid](https://druid.apache.org/docs/latest/development/build.html) from the HEAD of the master branch:

```bash
git clone https://github.com/apache/druid.git
cd druid
mvn clean install -Pdist -DskipTests
```

Then follow the instructions to locate and install the tarball.

In this tutorial, you will 

- break down a nested JSON array in 
- do a quick cumulative report using window functions.

_**Disclaimer:** This tutorial uses undocumented functionality and unreleased code. This blog is neither endorsed by Imply nor by the Apache Druid PMC. It merely collects the results of personal experiments. The features described here might, in the final release, work differently, or not at all. In addition, the entire build, or execution, may fail. Your mileage may vary._

## The data

just run a local kafka and then

check out the pizza simulator

run it like so:

```bash
python3 main.py --security-protocol PLAINTEXT --host localhost --port 9092 --topic-name pizza-orders --nr-messages 0 --max-waiting-time 5
```

capture the output using `kcat` and redirect to a file:

```bash
kcat -b localhost:9092 -t pizza-orders >>./pizza-orders.json
```

right now this technique is limited to batch ingestion

## Basic ingestion: the `pizza-orders` table

this is done using the wizard. note how in the SQL view, the type of the `pizzas` field is somewhat correctly recognized as a `COMPLEX<json>` but it does not know about the array structure

![Ingestion view for pizza-orders](/assets/2023-12-17-01-ingest-orders.jpg)

here is the ingestion query using MSQ:

```sql
REPLACE INTO "pizza-orders" OVERWRITE ALL
WITH "ext" AS (
  SELECT *
  FROM TABLE(
    EXTERN(
      '{"type":"local","baseDir":"/Users/hellmarbecker/meetup-talks/jsonarray","filter":"*json"}',
      '{"type":"json"}'
    )
  ) EXTEND ("id" BIGINT, "shop" VARCHAR, "name" VARCHAR, "phoneNumber" VARCHAR, "address" VARCHAR, "pizzas" TYPE('COMPLEX<json>'), "timestamp" BIGINT)
)
SELECT
  MILLIS_TO_TIMESTAMP("timestamp") AS "__time",
  "id",
  "shop",
  "name",
  "phoneNumber",
  "address",
  "pizzas"
FROM "ext"
PARTITIONED BY DAY
```

when we query this table, we see that indeed we have a general nested column here

![Sample of a query over orders](/assets/2023-12-17-02-select-orders.jpg)

we can look at the detailed values in the column

![Detail view of a pizzas object](/assets/2023-12-17-03-orders-detail.jpg)

## First attempt at breaking down the line items



----
 <p class="attribution">"<a target="_blank" rel="noopener noreferrer" href="https://www.flickr.com/photos/26242865@N04/5919366429">Pizza</a>" by <a target="_blank" rel="noopener noreferrer" href="https://www.flickr.com/photos/26242865@N04">Katrin Gilger</a> is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-sa/2.0/?ref=openverse">CC BY-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>. </p> 

