---
layout: post
title:  "Schema Evolution in Apache Druid"
categories: blog apache druid imply data-governance
---
[Apache Druid](https://druid.apache.org/) is known for its ability to handle schema changes during data ingestion and query in an extraordinarily flexible way. Normally, you don't even need to worry because everything is handled automatically for you. However, there are a few edge cases that data engineers and data analysts should be aware of. Let's have a look.

We are going to create a very simplistic [datasource (that's what we call the tables inside Druid](https://docs.imply.io/latest/druid/ingestion/#datasources)). It is going to have just two rows of data and two columns: a timestamp and an integer.

Create an inline datasource in the Druid Wizard and paste the following bit of data:
```
ts,value
2021-01-01,1
2021-01-02,2
```
Go through all the steps of schema creation, accepting the defaults. Choose weekly segments, because we want to add more data in the next steps. Again, continue through all the steps, accepting the defaults.

Pick a meaningful name for your datasource, for instance `schema_evolution`. By default, Druid suggests `inline_data` as the data source name for pasted data, which makes it too easy to mix up data from different experiments, creating a lot more schema evolution than you probably want to handle.

![Data Source Name](/assets/2021-08-13-ds-name.jpg)

Submit the ingestion task and wait until it is finished.

Repeat these steps with a similar data set:
```
ts,value
2021-02-01,1
2021-02-02,2
```
and you will be left with a datasource that has 4 rows of data ans is arguably not particularly interesting.

Now, let's try something different. The column names will be the same but the _value_ field now contains a string rather than a number.
```
ts,value
2021-03-01,aaa1
2021-03-02,aaa2
```
Data set number four is going to have floating point numbers:
```
ts,value
2021-04-01,1.4
2021-04-02,2.5
```
Finally, let's ingest another set of data with string values. This one, however, has [multi-value dimensions](https://blog.hellmar-becker.de/2021/08/07/multivalue-dimensions-in-apache-druid-part-1/).
```
ts,value
2021-05-01,a|b
2021-05-02,c|d
```
Here's how to configure the parser.
![CSV Parser configuration](/assets/2021-08-13-configure-parser.jpg)

## Learnings

Druid is amazingly flexible with regards to schema evolution. Often times, this is handled quietly without problems. In case of type conflicts:

- When segments have columns of the same name but a different type, the data dictionary will record the last column type that was ingested.
- If you have to run a SQL query that spans across such segments, in many cases you can get away with implicit type conversions.
- You can always get at the original data that is stored in a segment, using a [CAST](https://druid.apache.org/docs/latest/querying/sql.html#other-scalar-functions) function.
- If you have difficulty extracting values from a multivalue dimension, [array functions](https://druid.apache.org/docs/latest/misc/math-expr.html#array-functions) are your friend.
