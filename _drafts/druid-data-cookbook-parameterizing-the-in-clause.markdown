---
layout: post
title:  "Druid Data Cookbook: Parameterizing the IN clause"
categories: blog apache druid imply sql tutorial
twitter:
  image: /assets/2021-12-21-elf.jpg
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)

Let's look at a neat new feature in Druid 30 that makes using the API more flexible.

For this tutorial, download a fresh copy of [Druid 30](https://druid.apache.org/downloads/). Run a local [quickstart](https://druid.apache.org/docs/latest/tutorials/) instance and ingest the _Wikipedia_ sample data as per [this tutorial](https://druid.apache.org/docs/latest/tutorials/tutorial-msq-extern).


## The problem



### Recap: Parameters in the query API

Druid is typically queried through a REST API; the payload is documented [here](https://druid.apache.org/docs/latest/api-reference/sql-api#request-body). The API supports parameterizing queries, which is frequently used by programming language specific clients that create a wrapper layer around the API calls.

For instance, a simple query with a filter could be written like this: 

```json
{
    "query": "SELECT COUNT(*) FROM wikipedia WHERE channel = ?",
    "parameters": [
        {
            "type": "VARCHAR",
            "value": "#en.wikipedia"
        }
    ]
}
```

You can submit this query to the SQL endpoint using _curl_ like so:

```bash
curl --location 'http://localhost:8888/druid/v2/sql' \
--header 'Content-Type: application/json' \
--data '{
    "query": "SELECT COUNT(*) FROM wikipedia WHERE channel = ?",
    "parameters": [
        {
            "type": "VARCHAR",
            "value": "#en.wikipedia"
        }
    ]
}'
```

or you can use Postman:

![Postman query](/assets/2024-06-24-01-postman.jpg)


ingest the data: https://druid.apache.org/docs/latest/tutorials/tutorial-msq-extern

naive attempt, show how it fails (fabrice example)

## two new features in druid

- array as parameter
- planning in clauses as array selection
- scalar_in_array

## now let's make it work

proper attempt

## Conclusion

- yada yada


---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
