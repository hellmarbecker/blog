---
layout: post
title:  "Street Level Maps in Imply Pivot"
categories: blog druid imply pivot tutorial
---

![Street Level Map: Pivot Screenshot](/assets/xxxx.jpg)

[Imply Pivot](https://docs.imply.io/latest/pivot-overview/) has had map visualizations for a while. But the builtin map outline was limited to mostly by-country resolution (with ISO codes), or to the grid points defined by geohashes. Consequently, the map overlay was static and the maximum resoluton limited.

With Imply Private and Hybrid, you can use fine grained maps now the display data points by their exact longitude/latitude coordinates. The maps are provided by [MapBox](https://www.mapbox.com/) and are zoomable to the individual street level. Let's see how street level maps work with Imply Pivot in the quickstart version.

In this tutorial, you will:

- set up Pivot for street level maps with your own MapBox token to use MapBox overlays
- ingest a data set with ADS-B flight data
- configure your Pivot cube to use longitude and latitude data
- and configure a visualization, exploring some of the options.

The tutorial works with the [Imply 2023.01 quickstart](https://docs.imply.io/latest/quickstart/#use-unmanaged-imply-enterprise).

## Prerequisites

### Obtaining a Mapbox API Token

First of all, you will need to create a free account on the [MapBox](https://www.mapbox.com/) website. MapBox has a free tier that is sufficient for this tutorial. Navigate to your account page and click the `Create a token` button:

![Mapbox API token page](/assets/2023-02-01-01-mapbox1.jpg)

This takes you to the token creation page. Enter a name for your key and do not change any of the default options. In particular, _do not select any of the options under_ `Secret scopes`. Selecting any of those will create a secret key with a different prefix that Pivot cannot use.

![Mapbox API token page](/assets/2023-02-01-02-mapbox2.jpg)

Hit the `Create token` button 

![Mapbox API token page](/assets/2023-02-01-03-mapbox3.jpg)

and you will be taken to the `Tokens` page.

Find your token and click the clipboard icon to copy the token string:

![Mapbox API token page](/assets/2023-02-01-04-mapbox4.jpg)

### Configuring the Pivot Instance

Before you start up Imply, you have to enter the MapBox token into the Pivot configuration file. After you untar the Imply distribution, edit the file `conf-quickstart/pivot/config.yaml` and add the following lines to the end:

```yaml
mapboxConfig:
  token: <your mapbox token>
```

replacing the token placeholder with the token string you just copied. Save the configuration file.

### Enabling the Feature Flag

Start Imply:

```
bin/supervise -c conf/supervise/quickstart.conf
```

and navigate your browser to `localhost:9095` to access Pivot. Open the `Settings` page

![Pivot Settings](/assets/2023-02-01-05-pivotcfg1.jpg)

and select `Feature flags`. You will find the flag for street level maps at the very bottom. Toggle the flag and save the settings:

![Pivot Settings for flag](/assets/2023-02-01-06-pivotcfg2.jpg)


## The Data

### Data Generation

describe quickly the adsb format, paste a sample

### Data Ingestion

paste the ingestion spec here, note either kafka or static sample

### Logical Data Model

IN PIVOT, yada yada lon lat dimensions

![Lon/Lat Geo Dimensions in Pivot](/assets/xxxx.jpg)

## Creating a Visualization

### Selecting the Street Level Map

yada yada screenshot

![Street Level Map: Pivot Screenshot (same as above)](/assets/xxxx.jpg)

### Configuring the Street Level Map

heatmap vs blur

![Street Level Map: Pivot Screenshot with heatmap](/assets/xxxx.jpg)

## Conclusion

how cool is this
