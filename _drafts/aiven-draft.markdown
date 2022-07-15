---
layout: post
title:  "Connecting Apache Druid to Kafka with TLS authentication"
categories: blog druid imply ingestion tutorial kafka tls
---

We've looked at connecting [Apache Druid](https://druid.apache.org/) to a secure [Kafka](https://kafka.apache.org/) cluster [before](/2021/10/19/reading-avro-streams-from-confluent-cloud-into-druid/). In that article, I used a [Confluent Cloud](https://confluent.cloud/) cluster with API key and secret, which is basically username + password authentication.

Kafka also supports mutual TLS (mTLS) authentication. If this is used, keys and certificates need to be stored on the Druid cluster that wants to connect to Kafka, and the ingestion spec needs to reference these data.

In this tutorial, I am going to show how to connect Druid to Kafka using mTLS. I am using an [Aiven](https://aiven.io/) Kafka cluster because Aiven supports mTLS secured Kafka out of the box as the default.


https://stackoverflow.com/questions/45711911/add-certificates-to-key-store-and-trust-store

```
 1027  openssl pkcs12 -inkey service.key -in service.cert -export -out keystored.p12 -certfile ca.pem
 1028  keytool -importkeystore -destkeystore mykeystore.jks -srckeystore keystore.p12 -srcstoretype pkcs12
 1029  keytool -importkeystore -destkeystore mykeystore.jks -srckeystore keystored.p12 -srcstoretype pkcs12
 1030  keytool -import -alias aivenCert -file ca.pem -keystore cacerts –storepass changeit
 1031  keytool -import -alias aivenCert -file ca.pem -keystore cacerts 
 1032  ls -l
 1033  file mykeystore.jks
 1034  file keystored.p12
 1035  file cacerts
 1036  aws s3 cp cacerts s3://imply-cloud-sales-data/6d568a27-373c-4cc9-bb10-94647374e2ed/aiven_cacerts.jks
 1037  aws s3 cp mykeystore.jks s3://imply-cloud-sales-data/6d568a27-373c-4cc9-bb10-94647374e2ed/aiven_keystore.jks
```

```
{
        "bootstrap.servers": "kafka-imply-imply-f31e.aivencloud.com:17641",
        "security.protocol": "SSL",
        "ssl.truststore.location": "/opt/imply/user/aiven_cacerts.jks",
        "ssl.truststore.password": "changeit",
        "ssl.keystore.location": "/opt/imply/user/aiven_keystore.jks",
        "ssl.keystore.password": "changeit"
}
```

```
{
  "type": "kafka",
  "spec": {
    "ioConfig": {
      "type": "kafka",
      "consumerProperties": {
        "bootstrap.servers": "kafka-imply-imply-f31e.aivencloud.com:17641",
        "security.protocol": "SSL",
        "ssl.truststore.location": "/opt/imply/user/aiven_cacerts.jks",
        "ssl.truststore.password": "changeit",
        "ssl.keystore.location": "/opt/imply/user/aiven_keystore.jks",
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
          "address",
          {
            "name": "pizzas",
            "type": "json"
          }
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
