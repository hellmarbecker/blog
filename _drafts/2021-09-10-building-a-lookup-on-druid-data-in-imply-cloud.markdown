---
layout: post
title:  "Building a Lookup on Druid Data in Imply Cloud"
categories: blog imply druid
---

What you need:

Download the CA certificate 

Run
```
keytool -import -alias druid -file ~/Downloads/<CA CERTIFICATE>.crt -storetype JKS -keystore truststore.jks
```

Upload the truststore JKS to a S3 bucket that the cluster can access

Add it to the custom files for the cluster

```
{
  "type": "cachedNamespace",
  "extractionNamespace": {
    "type": "jdbc",
    "table": "druid.mini_news",
    "keyColumn": "id",
    "valueColumn": "v",
    "tsColumn": "__time",
    "connectorConfig": {
      "connectURI": "jdbc:avatica:remote:url=<INTERNAL LOAD BALANCER>/druid/v2/sql/avatica/;truststore=/opt/imply/user/truststore.jks;truststore_password=changeit;",
      "user": "admin",
      "password": "<PASSWORD>"
    },
    "pollPeriod": "PT1M"
  },
  "injective": false
}
```
