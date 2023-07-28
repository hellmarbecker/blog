---
layout: post
title:  "Druid Sneak Peek: Graphical Data Exploration"
categories: blog apache druid imply iot sql tutorial
---

Druid's unified console is a user interface mostly directed at data management. Among other things, you can control your ingestion tasks, manage segments and their compaction settings, monitor services, and there is also a query manager GUI that understands both SQL and Druid native queries.

For data visualization, up until now you had to use external tools such as Superset or Tableau, or Imply's own Pivot that comes bundled with the commercial distribution of the software.

But this is going to change. Druid 28 is going to add an exploration GUI that allows visual analysis of data!

- disclaimer and build instructions -

## How to access the Explorer view

go to the three dots `...` right next to the Services tab, open the menu and click `Explore`

![Screenshot of console with Explor menu selected]()

explain the GUI elements

left:

- datasource - table
- list of fields as they occur in the table, this does not care whether the fields are dimensions or metrics

middle:

- canvas
- filter

right: select visualization, the options available depend on the visualization type

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
