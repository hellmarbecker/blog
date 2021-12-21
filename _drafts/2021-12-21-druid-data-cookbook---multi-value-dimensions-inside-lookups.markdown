---
layout: post
title:  "Druid Data Cookbook - Multi-Value Dimensions Inside Lookups"
categories: blog apache druid imply jq tutorial
---

![Elf Chef](/assets/2021-12-21-elf.jpg)

We talked about joining a lookup against a multi-value dimension in Apache Druid [a while ago](/2021/10/14/druid-data-modeling-special-lookups-and-multi-value-dimensions/). Here's the opposite case: Imagine we have the multi-value entity inside the lookup!

Out of the box, this seems impossible. [Lookups](https://druid.apache.org/docs/latest/querying/lookups.html) are, after all, strictly single string to single string!

But let's see what we can find in our toolbox.

Like last time, I am going to use the [MPST movie database](https://ritual.uh.edu/mpst-2018/). Conveniently, it already contains a file `movie_to_label_name.json`, which, as the name suggests, maps each movie ID to the tag list that is associated with it.

## Preparing the Data for the Lookup

Alas, each label list in the data file is a JSON Array:

```
{
  ...
  "tt0077252": ["psychedelic", "violence"],
  "tt0388125": ["dramatic", "romantic", "flashback"],
  "tt0364725": ["comedy", "humor", "entertaining", "flashback"],
  ...
}
```

We cannot read this directly into Druid. Let's collapse each of these arrays into a comma separated string, for which task [`jq`](https://stedolan.github.io/jq/) comes in handy. After some trial and error (the [JQ Play](https://jqplay.org/) website is very helpful for trying out!), I found this neat little `jq` expression:

```bash
jq '.[] |= join(",")' movie_to_label_name.json >movie_to_label_str.json
```

With this preprocessing step, the data looks much better for our purpose:

```
{
  ...
  "tt0077252": "psychedelic,violence",
  "tt0388125": "dramatic,romantic,flashback",
  "tt0364725": "comedy,humor,entertaining,flashback",
  ...
}
```

## Populating the Lookup

Let's use the [Lookup API](https://druid.apache.org/docs/latest/querying/lookups.html#update-lookup) to upload these data into a static map:

```bash
echo '{ "version": "v1", "lookupExtractorFactory": { "type": "map", "map":'$(cat movie_to_label_str.json)' } }' | \
curl -H "Content-Type: application/json" http://localhost:8888/druid/coordinator/v1/lookups/config/_default/movie-tags -d@-
```

---

"This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che" by Medical Heritage Library, Inc. is licensed under [CC BY-NC-SA 2.0](https://creativecommons.org/licenses/by-nc-sa/2.0/?ref=openverse&atype=html)
