---
layout: post
title:  "Visualizing Geospatial Data in Imply Pivot"
categories: blog imply druid geospatial pivot
---
[Previously](/2021/11/07/fun-with-spatial-dimensions-in-apache-druid/), I looked at spatial dimensions in Apache Druid. Since [Imply](https://www.imply.io) created Pivot as a tailored visualization tool for Druid data, I would like to take advantage of the built-in map view to show my spatial dimensions on a map. How will we go about this?

Pivot cannot display latitude/longitude coordinates directly on a map but it can interpret geocoded data in [Geohash](https://en.wikipedia.org/wiki/Geohash) format. If we could transform our coordinates into a geohash, sure we would be able to make the map visualization work!

## Enabling Javascript transformations

Druid can work with arbitrary transformation expressions that are supplied as Javascript functions. Pivot supports this functionality, too. You have to use the legacy Plywood cube format though, it does not work with SQL.

Javascript is deactivated in Druid by default, but it can be enabled by setting
```
druid.javascript.enabled=true
```
in the common runtime properties for Druid. This is also a prerequisite for [working with Javascript transformations in Pivot](https://docs.imply.io/latest/dimensions/#custom-transformations). Here is how to do this in Imply Cloud:

![Common properties](/assets/2021-11-08-2-common-properties.jpeg)

Once Javascript is enabled, you can define custom transformations in the data cube options in Pivot. 

I took the example code from [Dave Troy's Github](https://github.com/davetroy/geohash-js), and modified it so as to accept the Druid geospatial format:

```javascript
function encodeGeoHash(arg) {
        var parts = arg.split(",");
        var latitude = parts[0];
        var longitude = parts[1];
        const BITS = [16, 8, 4, 2, 1];
        const BASE32 = "0123456789bcdefghjkmnpqrstuvwxyz";
        var is_even=1;
        var i=0;
        var lat = []; var lon = [];
        var bit=0;
        var ch=0;
        var precision = 12;
        geohash = "";

        lat[0] = -90.0;  lat[1] = 90.0;
        lon[0] = -180.0; lon[1] = 180.0;

        while (geohash.length < precision) {
          if (is_even) {
                        mid = (lon[0] + lon[1]) / 2;
            if (longitude > mid) {
                                ch |= BITS[bit];
                                lon[0] = mid;
            } else
                                lon[1] = mid;
          } else {
                        mid = (lat[0] + lat[1]) / 2;
            if (latitude > mid) {
                                ch |= BITS[bit];
                                lat[0] = mid;
            } else
                                lat[1] = mid;
          }

                is_even = !is_even;
          if (bit < 4)
                        bit++;
          else {
                        geohash += BASE32[ch];
                        bit = 0;
                        ch = 0;
          }
        }
        return geohash;
}
```


In order to make this into an one-liner, a number of free compressors/obfuscators are available, for instance [this one](https://javascriptcompressor.com/). Then escape all the double quotes with a backslash `\`, creating a code snippet ready to be inserted into the JSON configuration for the cube options.

```json
{
  "customTransforms": {
    "geohashFun": {
      "extractionFn": {
        "type": "javascript",
        "function": "function encodeGeoHash(arg){var parts=arg.split(\",\");var latitude=parts[0];var longitude=parts[1];const BITS=[16,8,4,2,1];const BASE32=\"0123456789bcdefghjkmnpqrstuvwxyz\";var is_even=1;var i=0;var lat=[];var lon=[];var bit=0;var ch=0;var precision=12;geohash=\"\";lat[0]=-90.0;lat[1]=90.0;lon[0]=-180.0;lon[1]=180.0;while(geohash.length<precision){if(is_even){mid=(lon[0]+lon[1])/2;if(longitude>mid){ch|=BITS[bit];lon[0]=mid}else lon[1]=mid}else{mid=(lat[0]+lat[1])/2;if(latitude>mid){ch|=BITS[bit];lat[0]=mid}else lat[1]=mid}is_even=!is_even;if(bit<4)bit++;else{geohash+=BASE32[ch];bit=0;ch=0}}return geohash}"
      }
    }
  }
}
```

Copy and paste this into the cube options:

![Cube Options](/assets/2021-11-08-3-cube-options.jpeg)
