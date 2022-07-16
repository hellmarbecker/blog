---
layout: post
title:  "Connecting Apache Druid to Kafka with TLS authentication"
categories: blog druid imply ingestion tutorial kafka tls
---

We've looked at connecting [Apache Druid](https://druid.apache.org/) to a secure [Kafka](https://kafka.apache.org/) cluster [before](/2021/10/19/reading-avro-streams-from-confluent-cloud-into-druid/). In that article, I used a [Confluent Cloud](https://confluent.cloud/) cluster with API key and secret, which is basically username + password authentication.

Kafka also supports mutual TLS (mTLS) authentication. In that case keys and certificates need to be stored on the Druid cluster that wants to connect to Kafka, and the ingestion spec needs to reference these data.

In this tutorial, I am going to show how to connect Druid to Kafka using mTLS. I am using an [Aiven](https://aiven.io/) Kafka cluster because Aiven supports mTLS secured Kafka out of the box as the default. [Aiven's documentation](https://developer.aiven.io/docs/products/kafka/getting-started.html) describes how to set up a Kafka cluster with Aiven. For this tutorial, the smallest plan (`Startup-2`) is enough.

## Setting up the Keystore and Truststore

An mTLS secured Kafka service will offer you three files to download:

- the access key `service.key`
- the access certificate `service.cert`
- the CA certificate `ca.pem`.

Download all three into a new directory `aiven-tls`:

![Aiven Cluster configuration with download buttons for certificates](/assets/2022-07-16-01.jpg)

We will have to convert those files to Java keystore format in order to use them with Druid. This is explained in detail in [this StackOverflow entry](https://stackoverflow.com/questions/45711911/add-certificates-to-key-store-and-trust-store).

Run the following commands to create a Java keystore and truststore file:

```bash
openssl pkcs12 -inkey service.key -in service.cert -export -out aiven_keystore.p12 -certfile ca.pem
keytool -importkeystore -destkeystore aiven_keystore.jks -srckeystore aiven_keystore.p12 -srcstoretype pkcs12
keytool -import -alias aivenCert -file ca.pem -keystore aiven_cacerts.jks 
```

Choose `changeit` as the password in both cases when `keytool` asks for a new password. 

This will leave you with two files `aiven_keystore.jks` and `aiven_cacerts.jks`.

## Generating Data

I am using Aiven's Pizza simulator to create a Kafka topic and fill it with some data. [This blog](https://aiven.io/blog/create-your-own-data-stream-for-kafka-with-python-and-faker) describes in detail how to set up the cluster, create a topic, and generate fake pizza order data to fill up the topic.

Enable `kafka.auto_create_topics_enable` in the advanced configuration settings of your cluster. Then, clone [the data simulator repository](https://github.com/aiven/python-fake-data-producer-for-apache-kafka) and make sure you have all necessary dependencies installed.

Run the command

```
python3 main.py --security-protocol SSL --cert-folder ~/aiven-tls --host <your Kafka service host + port> --port 17641 --topic-name pizza --nr-messages 100 --max-waiting-time 10
```

using the `cert-folder` where you downloaded the certificate files in the first step.

## Setting up Druid

Download the latest Druid release and follow the [quickstart instructions](https://druid.apache.org/docs/latest/tutorials/index.html). Go to the Druid console at `localhost:8888` and start the `Load data` wizard. Select `Kafka` and proceed to enter the connection details:

![Load data Wizard](/assets/2022-07-16-02.jpg)

Create a consumer properties snippet to reference the keystore and truststore files you created earlier. Here is a template, you need to fill in the correct path and `host:port`.

```json
{
  "bootstrap.servers": "<your Kafka service host + port>",
  "security.protocol": "SSL",
  "ssl.truststore.location": "<your home directory>/aiven-tls/aiven_cacerts.jks",
  "ssl.truststore.password": "changeit",
  "ssl.keystore.location": "<your home directory>/aiven-tls/aiven_keystore.jks",
  "ssl.keystore.password": "changeit"
}
```

Copy the snippet into the `Consumer properties` box and enter `pizza` as the topic name:

![Kafka connection details](/assets/2022-07-16-03.jpg)

As you continue from here, you should see the data coming in:

![Raw JSON Pizza data](/assets/2022-07-16-04.jpg)

Proceed all the way to the schema definition:

![Naive Pizza schema](/assets/2022-07-16-05.jpg)

But what is this?! All the order details seem to be missing.

Here is a sample of the data the simulator generates:

```
{'id': 91, 'shop': 'Its-a me! Mario Pizza!', 'name': 'Brandon Schwartz', 'phoneNumber': '305-351-2631', 'address': '746 Chelsea Plains Suite 656\nNew Richard, DC 42993', 'pizzas': [{'pizzaName': 'Diavola', 'additionalToppings': ['salami', 'green peppers', 'olives', 'garlic', 'strawberry']}, {'pizzaName': 'Salami', 'additionalToppings': ['tomato', 'olives']}, {'pizzaName': 'Salami', 'additionalToppings': ['banana', 'olives', 'artichokes', 'onion', 'banana']}, {'pizzaName': 'Mari & Monti', 'additionalToppings': ['olives', 'olives']}, {'pizzaName': 'Margherita', 'additionalToppings': ['mozzarella', 'tuna', 'olives']}, {'pizzaName': 'Mari & Monti', 'additionalToppings': ['pineapple', 'pineapple', 'ham', 'olives', 'salami']}], 'timestamp': 1657968296752}
```

There's a lot of details in nested structures: `pizzas` is a JSON array of objects that each have a `pizzaName` and a list of `additionalToppings`.

## Handling Nested Data

Since version 0.23, Druid supports ingestion of nested JSON data natively. (Previously you would have to extract individual fields using a [`flattenSpec`](https://druid.apache.org/docs/0.23.0/ingestion/data-formats.html#flattenspec). This works by specifying a dimension type of `json`.

Go to the ingestion spec editor, and add the below snippet

```json
          {
            "name": "pizzas",
            "type": "json"
          }
```

to the `dimensionsSpec` as shown:

![Pizza schema with nested data](/assets/2022-07-16-06.jpg)

Kick off the ingestion


```
{
  "type": "kafka",
  "spec": {
    "ioConfig": {
      "type": "kafka",
      "consumerProperties": {
        "bootstrap.servers": "<your Kafka service host + port>",
        "security.protocol": "SSL",
        "ssl.truststore.location": "<your home directory>/aiven-tls/aiven_cacerts.jks",
        "ssl.truststore.password": "changeit",
        "ssl.keystore.location": "<your home directory>/aiven-tls/aiven_keystore.jks",
        "ssl.keystore.password": "changeit"
      },
      "topic": "pizza",
      "inputFormat": {
        "type": "json"
      }
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
      "dimensionsSpec": {
        "dimensions": [
          {
            "type": "long",
            "name": "id"
          },
          "shop",
          "name",
          "phoneNumber",
          "address"
        ]
      },
      "granularitySpec": {
        "queryGranularity": "none",
        "rollup": false
      }
    }
  }
}
```
