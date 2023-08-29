---
layout: post
title:  "Using Druid with MinIO"
categories: druid minio tutorial blog
---

With on premise setups, compute/storage separation is often implemented using a NAS or similar storage unit that exposes an S3 API endpoint.

I want to emulate S3 related behavior in a self contained demo that I can run on my laptop without an internet connection. This is conveniently done using MinIO as my S3 compatible storage.

Let's deploy MinIO using this docker compose file:

```yaml
version: "3"

services:
  minio:
    image: minio/minio
    container_name: minio
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=password
      - MINIO_DOMAIN=minio
    networks:
      minio_net:
        aliases:
          - druid.minio
    ports:
      - 9001:9001
      - 9000:9000
    command: ["server", "/data", "--console-address", ":9001"]
  mc:
    depends_on:
      - minio
    image: minio/mc
    container_name: mc
    networks:
      minio_net:
    environment:
      - AWS_ACCESS_KEY_ID=admin
      - AWS_SECRET_ACCESS_KEY=password
      - AWS_REGION=us-east-1
    entrypoint: >
      /bin/sh -c "
      until (/usr/bin/mc config host add minio http://minio:9000 admin password) do echo '...waiting...' && sleep 1; done;
      /usr/bin/mc rm -r --force minio/indata;
      /usr/bin/mc mb minio/indata;
      /usr/bin/mc policy set public minio/indata;
      /usr/bin/mc rm -r --force minio/deepstorage;
      /usr/bin/mc mb minio/deepstorage;
      /usr/bin/mc policy set public minio/deepstorage;
      tail -f /dev/null
      "
networks:
  minio_net:
```

Save this file as `docker-compose.yaml` to your work directory and run the command

```bash
docker compose up -d
```

This gives you a MinIO instance and the `mc` client. It will also automatically create two buckets in MinIO, named `indata` and `deepstorage`, that we will need for this tutorial. If you point your browser to localhost:9000, you can verify that the buckets have been created:

![MinIO Bucket Explorer screenshot]()

(Kudos to [Tabular](https://github.com/tabular-io/docker-spark-iceberg) from whose GitHub repository I adapted the docker compose file.)

## Configuring MinIO as deep storage and log target

I am using the standard Druid 27.0 quickstart. If you want to start Druid using the new `start-druid` script, you find the relevant configuration settings in `conf/druid/auto/_common/common.runtime.properties` under your Druid installation directory.

First of all, you need to load the S3 extension by adding it to the load list - it should look similar to this:

```
druid.extensions.loadList=["druid-s3-extensions", "druid-hdfs-storage", "druid-kafka-indexing-service", "druid-datasketches", "druid-multi-stage-query"]
```

Also configure the S3 default settings (endpoint, authentication):

```
druid.s3.accessKey=admin
druid.s3.secretKey=password
druid.s3.protocol=http
druid.s3.enablePathStyleAccess=true
druid.s3.endpoint.signingRegion=us-east-1
druid.s3.endpoint.url=http://localhost:9000/
```

For using MinIO as deep storage, comment out the default settings for `druid.storage.*`, and insert this section instead:

```
druid.storage.type=s3
druid.storage.bucket=deepstorage
druid.storage.baseKey=segments
```

Likewise, change the default configuration for the indexer logs to:

```
druid.indexer.logs.type=s3
druid.indexer.logs.s3Bucket=deepstorage
druid.indexer.logs.s3Prefix=indexing-logs
```

Then start Druid like this:

```bash
bin/start-druid -m5g
```

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
