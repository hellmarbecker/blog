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

```sql
SELECT 
  cn, 
  count(*), 
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

