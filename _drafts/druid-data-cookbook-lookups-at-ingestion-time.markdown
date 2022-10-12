---
layout: post
title:  "Druid Data Cookbook: Lookups at Ingestion Time"
categories: blog imply druid tutorial
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)

A while ago I explained how to [apply a lookup to a multi-value dimension in Apache Druid](/2021/10/14/druid-data-modeling-special-lookups-and-multi-value-dimensions/). In that post, I also mentioned that you can apply lookups at ingestion time. Documentation on this feature is a bit terse, so let's find out how to make it work!

In this tutorial, you will:
- Define the product catalog - a lookup (SKU to product name) as a JSON map 
- Ingest a snippet of transactions that each have a customer and a basket, where the basket is a list of SKUs
- Replace the SKU numbers by product names during ingestion.

## Preparing Druid

I am using the [Druid 24.0 quickstart](https://druid.apache.org/docs/latest/tutorials/index.html) for this tutorial.

Untar the Druid tarball as per the instructions. We have to make a configuration change in order to make the tutorial work. Edit the file `conf/druid/single-server/micro-quickstart/_common/common.runtime.properties` and set

```properties
druid.lookup.enableLookupSyncOnStartup=true
```

Or, for any other form of deployment, make sure this option is set.

Then, from within the Druid directory, start Druid with the command:

```bash
bin/start-micro-quickstart
```

## Defining the Lookup

The product catalog is the same as in my earlier post and looks like this:

```json
{
  "0001": "Mug O'Grog",
  "0002": "Red Herring",
  "0003": "Root Beer",
  "0004": "Staple Remover",
  "0005": "Breath Mints",
  "0006": "Fabulous Idol"
}
```

Use the lookup creation wizard to define a lokup `eshop_sku`:

<img src="/assets/2021-10-14-1-create-lookup.jpg" width="50%" />

The exact steps are described [here](/2021/10/14/druid-data-modeling-special-lookups-and-multi-value-dimensions).

## Building the Data Model

In Druid 24.0 there are two ways to do batch ingestion: Classic and SQL. For now, use `Classic` mode:

<img src="/assets/2022-10-12-01-classic.jpg" width="25%" />

Use the sample transactions from the same earlier article as an inline input:

```json
{ "ts": "2021-10-14 10:00:00", "customer": "Gian", "basket": [ "0001", "0001", "0002", "0004" ] }
{ "ts": "2021-10-14 10:10:00", "customer": "Rachel", "basket": [ "0002", "0004", "0005" ] }
{ "ts": "2021-10-14 10:20:00", "customer": "Peter", "basket": [ "0005", "0004", "0002" ] }
{ "ts": "2021-10-14 10:30:00", "customer": "Gian", "basket": [ "0002" ] }
{ "ts": "2021-10-14 10:40:00", "customer": "Jessy", "basket": [ "0003", "0005", "0006" ] }
{ "ts": "2021-10-14 10:50:00", "customer": "Gian", "basket": [ "0005", "0006" ] }
```

Follow the steps that the wizard suggests. You would expect that the lookup would be defined in the `Transform` step, but due to a bug in the console we will have to skip this and define the transformation manually later. Proceed all the way to `Schema Definition`. Make sure that the order of items in the baskets is preserved by selecting `ARRAY` as the multi-value handling mode:

![Preserve MV array order](/assets/2022-10-12-02-array-order.jpg)

The different handling modes, and when to use them, are discussed in [my article on multi-value dimensions](/2021/09/25/multivalue-dimensions-in-apache-druid-part-3/).

On the `Partition` screen, set the segment granularity to `day` and proceed to the `Publish` screen. Here, set the datasource name to `eshop` and proceed again to the JSON editor.

## Adding the Lookup, First Attempt

We are going to add a new dimension which is supposed to hold the names of the basket items. Thus, it has to be a multi-value dimension too. So, add the following snippet to the `dimensionsSpec`:

```json
          {
            "type": "string",
            "name": "basket_item",
            "multiValueHandling": "ARRAY",
            "createBitmapIndex": true
          }
```

This new dimension will be populated by a transform. Add a `transformSpec` like so:

```json
      "transformSpec": {
        "transforms": [
          {
            "type": "expression",
            "expression": "lookup(basket, 'eshop_sku')",
            "name": "basket_item"
          }
        ]
      }
```

Here's a screenshot of the ingestion spec as it should look now:

![Ingestion spec with lookup, naive](/assets/2022-10-12-03-ingest1.jpg)

Submit the ingestion spec and wait for the job to finish. Let's look at the result:

![Query, naive model](/assets/2022-10-12-04-query1.jpg)

Unfortunately we are not quite there yet. The basket_item column has been populated only for one row of data, all the rest is _null_. This is because _basket_ is a multi-value dimension. The lookup has only worked for the one case where there is only one value.

## Making the Lookup Work With a Multi-Value Dimension

We will have to find a way to apply the lookup transform to _all_ values in the multi-value dimension. This is done using [the `map` function with a lambda expression](https://druid.apache.org/docs/latest/misc/math-expr.html#lambda-expressions-syntax).

Go back to your ingestion spec and change the transform definition to this:

```json
      "transformSpec": {
        "transforms": [
          {
            "type": "expression",
            "expression": "map((x)->lookup(x, 'eshop_sku'), basket)",
            "name": "basket_item"
          }
        ]
      }
```

And with that change,the query will give the expected result:

![Query, final](/assets/2022-10-12-05-query2.jpg)

Note how not only all the values are there, we have also preserved the order of items by specifying the `ARRAY` handling mode!

## Learnings

- ...

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
