---
layout: post
title:  "Visualizing Geospatial Data in Imply Pivot"
categories: blog imply druid geospatial pivot
---

![Small map](/assets/2022-02-21-0-banner.jpg)

[Previously](/2021/11/07/fun-with-spatial-dimensions-in-apache-druid/), I looked at spatial dimensions in Apache Druid. Since [Imply](https://www.imply.io) created Pivot as a tailored visualization tool for Druid data, I would like to take advantage of the built-in map view to show my spatial dimensions on a map. How will we go about this?

Pivot cannot display latitude/longitude coordinates directly on a map but it can interpret geocoded data in [Geohash](https://en.wikipedia.org/wiki/Geohash) format. If we could transform our coordinates into a geohash, sure we would be able to make the map visualization work!

Until recently, this was only possible with [a hack using Javascript](/2022/01/21/geospatial-data-in-apache-druid---generating-geohashes/). But with Imply's extensions to Druid, it becomes a breeze! This extension was added in Imply version 2022.02, so be sure to use the latest version if you run the test.

The `imply-utility-belt` extension has to be enabled in your common properties. If you are using the quickstart on unmanaged Imply Enterprise, look into the file `conf-quickstart/druid/_common/common.runtime.properties` and make sure it is included in the extension list:

```properties
#
# Extensions
#

druid.extensions.directory=dist/druid/extensions
druid.extensions.hadoopDependenciesDir=dist/druid/hadoop-dependencies
druid.extensions.loadList=["druid-histogram", "druid-datasketches", "druid-kafka-indexing-service", "imply-utility-belt"]
```

For the Cloud version of Imply Enterprise, this is just a checkbox in the Manager UI.

## Generating Data

I am using a simple Python script to generate my sample data. (You will need the [Faker](https://faker.readthedocs.io/en/master/index.html) module.)

```python
import json
from faker import Faker
from itertools import chain

fake = Faker()

def main():
    
    print("datetime,latitude,longitude,place_name,country_code,timezone,measure")

    for i in range(0, 10000):
        datetime = fake.date_time_this_month().strftime("%Y-%m-%d %H:%M:%S")
        place = fake.location_on_land()
        measure = str(fake.random_int(min=1, max=100))
        print(f"{datetime},{','.join(place)},{measure}")


if __name__ == "__main__":
    main()
```

The result looks like this:
```csv
datetime,latitude,longitude,place_name,country_code,timezone,measure
2022-02-09 13:37:51,14.37395,100.48528,Bang Ban,TH,Asia/Bangkok,13
2022-02-05 03:13:14,35.61452,-88.81395,Jackson,US,America/Chicago,8
2022-02-02 04:04:50,-19.65,47.31667,Antanifotsy,MG,Indian/Antananarivo,56
2022-02-01 17:11:39,49.13645,8.91229,Eppingen,DE,Europe/Berlin,60
2022-02-07 00:40:52,8.43333,99.96667,Nakhon Si Thammarat,TH,Asia/Bangkok,11
2022-02-03 13:33:20,35.74788,-95.36969,Muskogee,US,America/Chicago,76
2022-02-17 20:59:38,12.74482,4.52514,Argungu,NG,Africa/Lagos,17
2022-02-09 20:25:55,51.44889,5.51978,Tongelre,NL,Europe/Amsterdam,72
2022-02-08 12:04:56,6.84019,79.87116,Dehiwala-Mount Lavinia,LK,Asia/Colombo,27
```

Ingest these data without any changes or transformations, using monthly segments.

## Loading the Data into Pivot

Create a SQL cube from your data in Pivot.

In order to visualize the data on a map, use the new `ST_GEOHASH` function which is supplied by `imply_utility_belt`. It takes three parameters:
- longitude
- latitude
- an integer that describes the precision, that is the length of the Geohas string.

Make sure to tell Pivot that this is a Geo dimension:

<img src="/assets/2022-02-21-1-define-dim.jpg" width="60%" />

Then you can use the new dimension to show your data in a map view.

![Map view](/assets/2022-02-21-2-mapview.jpg)

## Learnings

- In order to use Imply's own visualization tool, there is now a function that creates Geohash strings out of geographical coordinates.
- This is new in Inply 2022.02
- It is found in the `imply-utility-belt` extension.
