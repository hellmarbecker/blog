---
layout: post
title:  "Druid Data Cookbook - Multi-Value Dimensions Inside Lookups"
categories: blog apache druid imply jq tutorial
---

![Elf Chef](/assets/2021-12-21-elf.jpg)

We talked about joining a lookup against a multi-value dimension in Apache Druid [a while ago](/2021/10/14/druid-data-modeling-special-lookups-and-multi-value-dimensions/). Here's the opposite case: Imagine we have the multi-value entity *inside the lookup*!

Out of the box, this seems impossible. [Lookups](https://druid.apache.org/docs/latest/querying/lookups.html) are, after all, strictly single string to single string!

But let's see what we can find in our toolbox.

Like last time, I am going to use the [MPST movie data set](https://ritual.uh.edu/mpst-2018/). Conveniently, it already contains a file `movie_to_label_name.json`, which, as the name suggests, maps each movie ID to the tag list that is associated with it.

## Preparing the Data for the Lookup

Each label list in the data file is a JSON Array:

```
{
  ...
  "tt0077252": ["psychedelic", "violence"],
  "tt0388125": ["dramatic", "romantic", "flashback"],
  "tt0364725": ["comedy", "humor", "entertaining", "flashback"],
  ...
}
```

We cannot read this directly into a Druid lookup. Let's collapse each of these arrays into a comma separated string, for which task [`jq`](https://stedolan.github.io/jq/) comes in handy. After some trial and error (the [JQ Play](https://jqplay.org/) website is very helpful for trying out!), I found this neat little `jq` expression:

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
curl -H "Content-Type: application/json" http://localhost:8888/druid/coordinator/v1/lookups/config/__default/movie-labels -d@-
```

A few notes on using the API:
- The documentation says this is an update API, but it will also create new lookup tiers or lookups if these don't exist.
- Make sure to choose a new unique value for the `"version"` field if you update an existing lookup spec. Druid keeps track of the version numbers and will return an error if you reuse an existing version number. 
- The default lookup tier is `__default` with a double underscore. If you spell it wrong, this will fail silently.

## Pulling the Labels Apart

Look what we can do now!

![](/assets/2021-12-21-1.jpg)

The lookup table can be queried in SQL just like any Druid datasource, and we can use `STRING_TO_MV` to create a multi-value field from the `v` column.

## Joining Against Fact Data

Let's generate some fake viewing data that we can use to join against. This little script will get a list of all known movie IDs from the full dataset file. Then, in a loop it picks a random value and writes it out along with the current timestamp. It requires a sufficiently new version of either `bash` or `zsh`:

```bash
#!/bin/zsh

cut -d , -f 1 mpst_full_data.csv | grep ^tt | while read mv; do movies+=($mv); done
size=${#movies[@]}

echo 'ts,movie_id'
for i in {1..100}; do
    index=$(($RANDOM % $size))
    echo $(date "+%Y-%m-%d %T"),${movies[$index]}
    sleep 0.5
done
```

I've ingested these data as datasource `viewdata`.

Now I can write a query against the tag list using the `LOOKUP` function, *and I can pull the tag list apart and group by tag just like with a regular multi-value dimension!*

![](/assets/2021-12-21-2.jpg)

You can filter by specific movies even if you haven't listed the movie ID:

![](/assets/2021-12-21-3.jpg)

Or sometimes not:

![](/assets/2021-12-21-4.jpg)

This is a small bug that happens whenever you group by a multi-value value from a lookup, and filter by one single key. Luckily, until this (known) bug is fixed, there is a workaround:

![](/assets/2021-12-21-5.jpg)

Adding a *null* value to the filter list fixes the problem.

## Learnings

- Look Ma, no [Python](https://www.python.org/)! This little lab works entirely with basic shell commands and `jq`.
- You can sneak your multi-value dimension into a Druid lookup, if you pack it into a string and unpack it on the fly.
- This allows treating multi-value dimension data as a [slowly changing dimension](https://dwgeek.com/slowly-changing-dimensions-scd.html/).
- Uploading a lookup through the API endpoint is elegant, but has some caveats.
- You can use both the lookup key and (multi-value) value in the same query, but (as of now) there is a limitation that makes queries fail if you filter by a single key.

---

"This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che" by Medical Heritage Library, Inc. is licensed under [CC BY-NC-SA 2.0](https://creativecommons.org/licenses/by-nc-sa/2.0/?ref=openverse&atype=html)
