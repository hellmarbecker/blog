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

## Recap: Parameters in the query API

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

Let's make the query a bit more complex. We want to count the rows for more than one channel with a simple `GROUP BY` and an `IN` clause: 

```json
{
    "query": "SELECT COUNT(*) FROM wikipedia WHERE channel IN ?",
    "parameters": [
        {
            "type": "VARCHAR",
            "value": "(#en.wikipedia, #de.wikipedia, #fr.wikipedia)"
        }
    ]
}
```

Alas, this fails.

![Failing Postman query](/assets/2024-06-24-02-postman2.jpg)

And until Druid 29, you would have to work around this problem because `ARRAY`s as parameters weren't really supported.

## Two new features in Druid

Druid 30 brings two new features that help us here:

- Druid 30 supports passing `ARRAY`s as parameters. (https://github.com/apache/druid/pull/16274) The way to do this is to specify a type of `"ARRAY"` and to use a JSON array as the value.
- There is a new `SCALAR_IN_ARRAY` function that checks for presence of a particular value in an array. In fact, a conventional `IN` filter would internally use this functionality if the number of elements in the list is large enough. (https://github.com/apache/druid/pull/16306)

With that, we have everything to make the query work.

## Now, let's make it work

We'll pass the list of channels as an `ARRAY` type parameter and use `SCALAR_IN_ARRAY` in place of `IN`:

```json
{
    "query": "SELECT channel, COUNT(*) FROM wikipedia WHERE SCALAR_IN_ARRAY(channel, ?) GROUP BY 1",
    "parameters": [
        {
            "type": "ARRAY",
            "value": ["#en.wikipedia", "#de.wikipedia", "#fr.wikipedia"]
        }
    ]
}
```

The query works and returns the expected result:

![Successful Postman query](/assets/2024-06-24-03-postman-final.jpg)

## Conclusion

We've learned that a simple `IN` filter cannot be parameterized through the Druid REST API. However, Druid 30 introduces an alternative because it supports array parameters. With array parameters and the new `SCALAR_IN_ARRAY` function, you can efficiently parameterize filter lists through the SQL REST API.

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
