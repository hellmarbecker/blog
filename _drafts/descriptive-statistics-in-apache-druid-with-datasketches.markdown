---
layout: post
title:  "Descriptive Statistics in Apache Druid with Data Sketches"
categories: blog druid imply statistics datasketches tutorial
---

Let's try some fun with [data sketches](https://druid.apache.org/docs/latest/development/extensions-core/datasketches-extension.html) in Apache Druid.

Data sketches are a way to get a good estimate of measures that you would normally only obtain by going through every single row of your data, which could become prohibitive as the amount of data grows.

One common example is counting unique things, such as visitors to a web site. But today I am going to look at [quantiles sketches](https://druid.apache.org/docs/latest/development/extensions-core/datasketches-quantiles.html).

According to [Wikipedia](https://en.wikipedia.org/wiki/Quantile),
> quantiles are cut points dividing the range of a probability distribution into continuous intervals with equal probabilities, or dividing the observations in a sample in the same way.

So, the 0.5-quantile of a variable is the cut point such that 50% of rows have a value less than or equal to it, and 50% are above it. The 0.5-quantile is also known as the _median_. For a standard bell curve, the median and the mean (average) coincide:

![Bell curve with median](/assets/2022-03-20-01-gauss.png)

Likewise, the cut point such that 25% of values are less than or equal to it, is the _first quartile_.

Quantiles are handy when trying to describe properties of distributions that are skewed or have outliers. This is something we are going to look at today. Look at this distribution:

![Bimodal with median and mean](/assets/2022-03-20-02-bimodal.png)

## Generating Data

First, let's generate some data. I am logging down the incomes of a fictional population that is split into four segments:
- **Segment A**'s income is normal distributed (bell curve) with the peak at 10,000 and a standard deviation of 2,000.
- **Segment B** has a normal distribution too, but wider with a standard deviation of 10,000, so we can compare it to Segment C.
- **Segment C**'s income follows an exponential distribution with a mean of 10,000 (and a standard deviation of 10,000, that's how the exponential distribution works.)
- **Segment D** has a bimodal distribution, with the majority of members making around 10,000, and a small group that peaks around 50,000. This is modeled as the weighted sum of two bell curves.

Here's the code for it:

```python
import time
import random
import json


def bimodal(m1, m2, s, p):

    if random.random() <= p:
        m = m1
    else:
        m = m2
    return random.gauss(m, s)


def main():
 
    distr = {
        'A': { 'fn': random.gauss,       'param': (10000, 2000) },
        'B': { 'fn': random.gauss,       'param': (10000, 10000) },
        'C': { 'fn': random.expovariate, 'param': (0.0001,) },
        'D': { 'fn': bimodal,            'param': (10000, 50000, 2000, 0.8) },
    }
    for i in range(0, 10000):
        grp = random.choices('ABCD', cum_weights=(0.50, 0.75, 0.90, 1.00), k=1)[0]
        rec = {
            'tn': time.time() - 10000 + i,
            'cn': grp,
            'rn': distr[grp]['fn'](*distr[grp]['param'])
        }
        print(json.dumps(rec))

if __name__ == "__main__":
    main()
```
Run this code to generate an input file for Druid, and set up ingestion using the `Load data` wizard. We are going to roll up the data with a query granularity of 15 minutes, and we want to have three metrics:
- the standard `count` metric
- the equally standard sum of salaries `sum_rn` so we can actually compute the mean salary for each segment
- a quantiles sketch over `rn`.

This is configured in the `Configure schema` section of the wizard like so:
![Config Wizard showing data sketch](/assets/2022-03-20-03-config-sketch.jpg)

You can set the segment granularity to `HOUR`.

## Getting Quantiles

In order to work with data sketches, you would usually run a `GROUP BY` query, and run a [sketch aggregation](https://druid.apache.org/docs/latest/querying/sql.html#aggregation-functions) over the sketch field. In the case of quantiles sketches, the aggregator is `DS_QUANTILES_SKETCH`. On top of this, you can run [sketch functions](https://druid.apache.org/docs/latest/querying/sql.html#sketch-functions) that do something useful.

### Median vs. Mean

Let's compute the median income for each population segment:
```sql
SELECT 
  cn, 
  DS_GET_QUANTILE(DS_QUANTILES_SKETCH(qs_rn, 128), 0.5) AS median_s,
  SUM(sum_rn) / SUM("count") AS mean_s
FROM randstream_salary
GROUP BY 1
```

cn|	median_s|	mean_s
:---:|:---:|:---:
A	|9996.84943938557	|9989.123516983642
B	|9868.801625916114	|9856.779147538875
C	|6790.745679257308	|9771.002219613993
D	|10906.525946085518	|18933.47273254579

(Since these are random numbers, your result will be slightly different.)

As we expected, for segments A and B the median and mean values coincide (more or less.) Segment C, the one with the exponential distribution, has a difference - the mean value is `1 / λ`, the parameter of the distribution, while the median is `ln(2) / λ`, and `ln(2) ≃ 0.693` so we're good.

Segment D has the most pronounced discrepancy between mean and median (see the picture above) - the median is well within the big first bump of the distribution and reflects what "most" of the population members can expect to earn. The median shows a distorted picture because of the small subsegment of high earners.

Because the combination `DS_GET_QUANTILE(DS_QUANTILES_SKETCH(...)` is so frequently used, you can use the shorthand `APPROX_QUANTILE(expr, probability, [resolution])` instead.

### Quartiles and More

Let's beef up this query a bit. We want to know the salary thresholds for 25%, 50%, and 75% of the population of each segment, respectively:
```sql
SELECT 
  cn, 
  DS_GET_QUANTILE(DS_QUANTILES_SKETCH(qs_rn, 128), 0.25) AS q1_s,
  DS_GET_QUANTILE(DS_QUANTILES_SKETCH(qs_rn, 128), 0.5) AS q2_s,
  DS_GET_QUANTILE(DS_QUANTILES_SKETCH(qs_rn, 128), 0.75) AS q3_s,
  SUM(sum_rn) / SUM("count") AS mean_s
FROM randstream_salary
GROUP BY 1
```
cn	|q1_s|	q2_s	|q3_s	|mean_s
:---:|:---:|:---:|:---:|:---:
A	|8644.233235251919	|9996.84943938557	|11393.163108254672	|9989.123516983642
B	|2965.15230310896	|9969.383470761108	|16554.475078454096	|9856.779147538875
C	|2779.0521582375077	|6741.121631303921	|13769.232186289446	|9771.002219613993
D	|9201.740561816481	|10831.763371338158	|13466.267335825763	|18933.47273254579

We can now see the difference between A and B, B being much broader distributed. Also the skewedness of D becomes even clearer - more then 75% of that segment are concentrated within the lower peak!



### Outlier Detection

Finally, quantiles are handy to find boundaries of a value range if we want to exclude outliers. This query:
```sql
SELECT 
  cn,
  DS_GET_QUANTILE(DS_QUANTILES_SKETCH(qs_rn, 128), 0.98) AS most_s,
  SUM(sum_rn) / SUM("count") AS mean_s
FROM randstream_salary
GROUP BY 1
```
will give the maximum salary of each segment, excluding extreme outliers.





```sql
SELECT 
  cn, 
  count(*), 
  DS_QUANTILES_SKETCH(qs_rn, 128) AS q,
  DS_GET_QUANTILES(DS_QUANTILES_SKETCH(qs_rn, 128), 0.25, 0.5, 0.75) AS quartiles_s,
  DS_GET_QUANTILE(DS_QUANTILES_SKETCH(qs_rn, 128), 0.25) AS q1_s, 
  DS_GET_QUANTILE(DS_QUANTILES_SKETCH(qs_rn, 128), 0.5) AS median_s, 
  DS_GET_QUANTILE(DS_QUANTILES_SKETCH(qs_rn, 128), 0.75) AS q3_s, 
  DS_HISTOGRAM(DS_QUANTILES_SKETCH(qs_rn, 128), 5000, 10000, 20000) AS hi_s,
  DS_CDF(DS_QUANTILES_SKETCH(qs_rn, 128), 5000, 10000, 20000) AS ff_s,
  DS_RANK(DS_QUANTILES_SKETCH(qs_rn, 128), 5000) AS f5k_s,
  DS_RANK(DS_QUANTILES_SKETCH(qs_rn, 128), 10000) AS f10k_s,
  DS_RANK(DS_QUANTILES_SKETCH(qs_rn, 128), 20000) AS f20k_s,
  SUM(sum_rn) / SUM("count") AS avg_s
FROM randstream_salary
GROUP BY 1
```

