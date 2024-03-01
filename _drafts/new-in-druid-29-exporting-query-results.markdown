---
layout: post
title:  "New in Druid 29: Exporting Query Results"
categories: blog apache druid imply sql tutorial
twitter:
  image: /assets/2021-12-21-elf.jpg
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)


## the problem

if you want to extract detailed data / large data from druid, you need to issue an asynchronous query (link)

this is a 2 step process:
- first start the query
- then poll the task endpoint until it is done, and retrieve the result

meanwhile the data is written inside druid already. you can define a path and instruct druid to use durable storage (link) for this result, but: this is still in a druid specific format

what if we could skip that step (persisting the result) completely and send the result directly to a file in a format of our choice?

turns out druid 29 can do this. this is for now somewhat limited - it only supports csv, and only local filesystem or s3. but other formats, such as parquet, are coming

let's try this out with a quickstart (link) installation

in this tutorial, you will
- yada yada

## preparation

we are going to export to local storage. to limit the attack surface for malicious or inexperiences users, you have to define a specific filesystem path where druid is allowed to store export files.

on your local machine, install druid 29 from the tarball

create the directory `/tmp/druid-export`

edit the file `conf/druid/auto/_common/common.runtime.properties` and add the line

```
druid.export.storage.baseDir=/tmp/druid-export
```

at the end of the file

then start druid like so, from within your druid install directory

```
bin/start-druid -m5g
```

ingest the wikipedia sample data following the instructions (link)

then go to the query tab

## exporting data

run this query

```sql
INSERT INTO 
EXTERN(local(exportPath => '/tmp/druid-export/wikipedia-export'))
AS CSV
SELECT * FROM wikipedia
```

![Screenshot of running query](/assets/2024-03-01-01.jpg)

when the query finishes, check the export directory and voilà - the csv file is there

![Preview of result file in a shell window](/assets/2024-03-01-02.jpg)

note: the target directory has to be empty, else you get an error message

this also works for export to [S3](https://druid.apache.org/docs/latest/multi-stage-query/reference/#s3)

## Learnings

- yada yada

---

"[This image is taken from Page 500 of Praktisches Kochbuch f&uuml;r die gew&ouml;hnliche und feinere K&uuml;che](https://www.flickr.com/photos/mhlimages/48051262646/)" by [Medical Heritage Library, Inc.](https://www.flickr.com/photos/mhlimages/) is licensed under <a target="_blank" rel="noopener noreferrer" href="https://creativecommons.org/licenses/by-nc-sa/2.0/">CC BY-NC-SA 2.0 <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/nc.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" style="height: 1em; margin-right: 0.125em; display: inline;"/></a>.
