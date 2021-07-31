---
layout: post
title:  "Summary of Druid native query options"
categories: blog imply druid
---
Druid's native query language supports five distinct query types. Each of these query types requires that certain attributes be set in the query JSON object.

Here is a comprehensive overview of all the query types with their required and optional attributes.

In any column, a "yes" means that this option is required for that query type; "no" means it is optional, and "-" means it is unavailable.

|property |description |Timeseries |TopN |groupBy |scan |search |
|---|---|---|---|---|---|---|
|queryType |This String should always be "..."; this is the first thing Apache Druid looks at to figure out how to interpret the query |"timeseries"|"topN" |"groupBy"|"scan"|"search" |
|dataSource |A String or Object defining the data source to query, very similar to a table in a relational database. See DataSource for more information. |yes |yes |yes |yes |yes |
|intervals |A JSON Object representing ISO-8601 Intervals. This defines the time ranges to run the query over. |yes |yes |yes |yes |yes |
|granularity |Defines the granularity to bucket query results. See Granularities |yes |yes |yes |- |no (default to all) |
|filter |See Filters |no |no |no |no |no |
|aggregations |See Aggregations |no |for numeric metricSpec, aggregations or postAggregations should be specified. Otherwise no.|no |- |- |
|postAggregations|See Post Aggregations |no |for numeric metricSpec, aggregations or postAggregations should be specified. Otherwise no.|no |- |- |
|descending |Whether to make descending ordered result. Default is false(ascending). |no |- |- |- |- |
|limit |An integer that limits the number of results. The default is unlimited. Search query: Defines the maximum number per Historical process (parsed as int) of search results to return. |no |- |- |no |no (default to 1000)|
|offset |Skip this many rows when returning results. Skipped rows will still need to be generated internally and then discarded, meaning that raising offsets to high values can cause queries to use additional resources. Together, "limit" and "offset" can be used to implement pagination. However, note that if the underlying datasource is modified in between page fetches in ways that affect overall query results, then the different pages will not necessarily align with each other.|- |- |- |no |- |
|dimension |A String or JSON object defining the dimension that you want the top taken for. For more info, see DimensionSpecs |- |yes |- |- |- |
|threshold |An integer defining the N in the topN (i.e. how many results you want in the top list) |- |yes |- |- |- |
|metric |A String or JSON object specifying the metric to sort by for the top list. For more info, see TopNMetricSpec. |- |yes |- |- |- |
|dimensions |A JSON list of dimensions to do the groupBy over; or see DimensionSpec for ways to extract dimensions. | | |yes |- |- |
|limitSpec |See LimitSpec. |- |- |no |- |- |
|having |See Having. |- |- |no |- |- | 
|subtotalsSpec |A JSON array of arrays to return additional result sets for groupings of subsets of top level dimensions. It is described later in more detail. |- |- |no |- |- |
|resultFormat |How the results are represented: list, compactedList or valueVector. Currently only list and compactedList are supported. Default is list |- |- |- |no |- |
|columns |A String array of dimensions and metrics to scan. If left empty, all dimensions and metrics are returned. |- |- |- |no |- |
|batchSize |The maximum number of rows buffered before being returned to the client. Default is 20480 |- |- |- |no |- |
|order |The ordering of returned rows based on timestamp. "ascending", "descending", and "none" (default) are supported. Currently, "ascending" and "descending" are only supported for queries where the \_\_time column is included in the columns field and the requirements outlined in the time ordering section are met. |- |- |- |none |- |
|legacy |Return results consistent with the legacy "scan-query" contrib extension. Defaults to the value set by druid.query.scan.legacy, which in turn defaults to false. See Legacy mode for details. |- |- |- |no |- |
|searchDimensions|The dimensions to run the search over. Excluding this means the search is run over all dimensions. |- |- |- |- |no |
|query |See SearchQuerySpec. |- |- |- |- |yes |
|sort |An object specifying how the results of the search should be sorted. Possible types are "lexicographic" (the default sort), "alphanumeric", "strlen", and "numeric". See Sorting Orders for more details. |- |- |- |- |no |
|context |Can be used to modify query behavior, including grand totals and zero-filling. See also Context for parameters that apply to all query types. |no |no |no |no |no |

