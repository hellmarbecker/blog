---
layout: post
title:  "Overlaying Multiple Metrics in Imply Pivot"
categories: blog druid imply pivot tutorial
---

![Screenshot with 3 metrics overlayed]()

today we are going to look at a new enhancement for line chart graphs in pivot, such as timeseries curves
up until now, you could only do one measure in a graph, asterisk workaround with comparison
if you pulled in multiple metrics, you would get each in its own chart, like this

![Screenshot with 3 metrics in rows]()

but people wanted to have all curves in one, like in the screenshot above
this is possible now so how do you go about it?

## Two (or more) measures in one chart

here is how to show measures in one chart. we are looking at clickstream data and we want to show the total number of events, the number of clicks, and the number of sessions

pull all the measures you want to show into the show bar

-> events, clicks, unique sessions

![Screenshots with 3 measures in rows, highlight the drag and drop]()

select the paintbrush icon on the right sidebar
and "show measures in" "cell"

but what if the measures are to a vastly different scale?

## Two measures with separate axis scaling

say we have a conversion goal and we want to look at both the total traffic and the conversion rate

pull number of clicks, conversion rate

select measure in row again

![Screenshot with clicks and conversion rate, have a balloon on the curve to show the numbers at one point]()

but as you can see, the scales are so vastly different that the conversion rate all but disappears
well if you have only 2 measures you can show them on different axes so that both curves fill the canvas

![Screenshot with clicks and conversion rate, highlight dual axis menu]()

you can choose whether you want to show horizontal grid line for both axes or only for the first

![Highlight show horizontal grid menu and lines for both axes]()

## Learnings

yada yada
