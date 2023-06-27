---
layout: post
title: "New in Druid 26: Data Provenance Tracking with Kafka Headers, Automatically"
categories: blog imply druid kafka json tutorial
---

![Lufthansa Airbus A350 XWB D-AIXP arrives SFO L1060413, by wbaiv (Bill Abbott)](/assets/2023-06-26-00-airplane.jpg)

## Generating the data

let's generate ourselves some data with kafka headers, this can be dome with `kcat`

recur to <PREVIOUS BLOGs>

but because we have multiple raspis now, we want to do some data provenance tracking. you can do that with kafka headers

the script

```bash
#!/bin/bash

CC_BOOTSTRAP="<confluent cloud bootstrap server>"
CC_APIKEY="<api key>"
CC_SECRET="<secret>"
CC_SECURE="-X security.protocol=SASL_SSL -X sasl.mechanism=PLAIN -X sasl.username=${CC_APIKEY} -X sasl.password=${CC_SECRET}"
CLIENT_ID="<client id>"
LON="0.0"
LAT="0.0"
TOPIC_NAME="adsb-raw"

nc localhost 30003 \
    | awk -F "," '{ print $5 "|" $0 }' \
    | kafkacat -P \
        -t ${TOPIC_NAME} \
        -b ${CC_BOOTSTRAP} \
        -H "ClientID=${CLIENT_ID}" \
        -H "ReceiverLon=${LON}" \
        -H "ReceiverLat=${LAT}" \
        -K "|" \
        ${CC_SECURE}
```

so this adds a kafka key (the aircraft hex ID), and a unique ID for the radar receiver, and also the receiver coordinates

## Ingesting the data

use Druid 26 quickstart

create a kafka connection, we are using confluent cloud so we have to encode the credentials in the consumer properties as described in <PREVIOUS BLOG>

note how the preview looks different from previous druid versions:

![Kafka topic preview with metadata](/assets/2023-06-27-01-preview.jpg)

it now lists the kafka metadata

- timestamp
- key
- headers

along with the payload





the column headers

```csv
MT,TT,SID,AID,Hex,FID,DMG,TMG,DML,TML,CS,Alt,GS,Trk,Lat,Lng,VR,Sq,Alrt,Emer,SPI,Gnd
```

## Conclusion

- lorem ipsum

----

 <p class="attribution">"<a target="_blank" rel="noopener noreferrer" href="https://www.flickr.com/photos/wbaiv/52202356360/">Lufthansa Airbus A350 XWB D-AIXP arrives SFO L1060413</a>" by <a target="_blank" rel="noopener noreferrer" href="https://www.flickr.com/photos/wbaiv">wbaiv</a> is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-sa/2.0/">CC BY-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>. </p> 
