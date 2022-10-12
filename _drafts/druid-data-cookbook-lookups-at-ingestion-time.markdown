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

I am using the [Druid 24.0 quickstart](https://druid.apache.org/docs/latest/tutorials/index.html) for this tutorial.

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

Follow the steps that the wizard suggests. You would expect that the lookup would be defined in the `Transform` step, but due to a bug in the console we will have to skip this and define the transformation manually later. Proceed all the way to `Schema Definition`. Make sure that the order of items in the baskets is preserved by selecting `ARRAY` as the multi-value handling mode:

![Preserve MV array order](/assets/2022-10-12-02-array-order.jpg)

This is discussed in [my article on multi-value dimensions](/2021/09/25/multivalue-dimensions-in-apache-druid-part-3/).

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
