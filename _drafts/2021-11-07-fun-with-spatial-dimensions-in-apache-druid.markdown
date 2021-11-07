---
layout: post
title:  "Fun with Spatial Dimensions in Apache Druid"
categories: blog apache druid imply
---

I took the example code from [](), and made it to accept the Druid geospatial format:

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


In order to make this into an one-liner, a number of free compressors/obfuscators are available, for instance [this one](https://javascriptcompressor.com/). Then escape all the double quotes with a backslash `\`, cresting a code snippet ready to be inserted into the JSON configuration for the cube options.

