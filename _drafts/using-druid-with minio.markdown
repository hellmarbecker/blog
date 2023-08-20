---
layout: post
title:  "Rebuilding my mail server using OpenSMTPD on Debian"
categories: druid minio tutorial blog
---

## Configuring MinIO as deep storage and log target

lorem ipsum

## Ingesting data from MinIO

we take advantage of the same settings as for deep storage

## Changing the endpoint settings in the ingestion command

now let's go back to local deep storage. so we cannot take advantage of endpoint settings that are baked into the service properties file.

hence we need to establish those settings right in the ingestion spec

### JSON version

yada yada

```json
      "inputSource": {
        "type": "s3",
        "prefixes": [
          "s3://indata/"
        ],
        "properties": {
          "accessKeyId": {
            "type": "default",
            "password": "admin"
          },
          "secretAccessKey": {
            "type": "default",
            "password": "password"
          }
        },
        "endpointConfig": {
          "url": "http://localhost:9000",
          "signingRegion": "us-east-1"
        },
        "clientConfig": {
          "disableChunkedEncoding": true,
          "enablePathStyleAccess": true,
          "forceGlobalBucketAccessEnabled": false
        }
      }
```

note: you need to include the `http://` in the endpoint URL, if you put it in the `clientConfig.protocol`, it is not recognized

### SQL version

in the SQL version, we copy the same settings into the EXTERN statement, like so ...

## Conclusion

yada yada
