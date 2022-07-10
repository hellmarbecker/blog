---

---
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
