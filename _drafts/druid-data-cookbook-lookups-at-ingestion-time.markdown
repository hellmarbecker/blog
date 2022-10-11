---
layout: post
title:  "Druid Data Cookbook: Lookups at Ingestion Time"
categories: blog imply druid tutorial
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)

A while ago I explained how to [apply a lookup to a multi-value dimension in Apache Druid](/2021/10/14/druid-data-modeling-special-lookups-and-multi-value-dimensions/). In that post, I also mentioned that you can apply lookups at ingestion time. Documentation on this feature is a bit terse, so let's find out how to make it work!


---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
