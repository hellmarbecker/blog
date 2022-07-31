---
layout: post
title:  "Using Imply Pivot with Druid to Deduplicate Timeseries Data"
categories: blog druid imply analytics tutorial
---

For instance, imagine you are running a solar or wind power plant. You are measuring the power output, among other parameters, on a regular basis, but the measurements are not evenly distributed along the time axis. Sometimes you get several measurements for the same unit of time, sometimes there is only one.



```csv
ts,var,val
2022-07-31T10:01:00,p,31
2022-07-31T10:02:00,p,33
2022-07-31T10:18:00,p,48
```
