---
layout: post
title:  "Druid Sneak Peek: Graphical Data Exploration"
categories: blog apache druid imply iot sql tutorial
---

Druid's unified console is a user interface mostly directed at data management. Among other things, you can control your ingestion tasks, manage segments and their compaction settings, monitor services, and there is also a query manager GUI that understands both SQL and Druid native queries.

For data visualization, up until now you had to use external tools such as Superset or Tableau, or Imply's own Pivot that comes bundled with the commercial distribution of the software.

But this is going to change. Druid 28 is going to add an exploration GUI that allows visual analysis of data!

![Screenshot of Data Explorer]()

This is a sneak peek into Druid 28 functionality. In order to use the new functions, you can (as of the time of writing) [build Druid](https://druid.apache.org/docs/latest/development/build.html) from the HEAD of the master branch:

```bash
git clone https://github.com/apache/druid.git
cd druid
mvn clean install -Pdist -DskipTests
```

Then follow the instructions to locate and install the tarball.

_**Disclaimer:** This tutorial uses undocumented functionality and unreleased code. This blog is neither endorsed by Imply nor by the Apache Druid PMC. It merely collects the results of personal experiments. The features described here might, in the final release, work differently, or not at all. In addition, the entire build, or execution, may fail. Your mileage may vary._

For this post, I ingested the wikipedia sample data, as described in [the quickstart tutorial](https://druid.apache.org/docs/latest/tutorials/tutorial-msq-extern.html). You are of course encouraged to try out different data sets with the new explorer.

## How to access the Explorer view

To access the data explorer, go to the three dots `...` right next to the Services tab, open the menu and click `Explore`:

![Screenshot of console with Explore menu selected]()

You will be greeted with a canvas in the middle, and surrounding GUI controls:

- In the top left field you select the datasource (table) that you wish to explore.
- As soon as a datasource is selected, the left panel shows a list of all fields as they occur in the datasource. This does not care whether the fields are dimensions or metrics.
- In the top bar you can set filters. Time filters come with an option of relative or absolute times. For character values, there is a regular expression filters as well as the ability to pick literal values.
- In the right panel you choose one of the supported visualization types. Depending on your selection, different configuration options appear below.

Let's go through the list of visualization types.

## Time chart

this is an area chart, optionally stacked area

one metric, it is possible to limit the stacked 

![Screenshot]()

## Bar chart

one bar column (dimension), one metric, possible another metric to sort

## Table

group by => aggregates. __time has builtin intelligence with bucketing

show, can show a column without aggregating

pivot (across instead of down). this uses filtered metrics

aggregates. these would be the derived metrics

compares. compare by time interval. compare and pivot are for now mutually exclusive

## Pie chart

one dimension, one metric. you can specify the # of slices, the rest goes into Other

## Multi-axis chart

time chart, many metrics - overlayed and each to its own scale

## Conclusion

- yada yada
