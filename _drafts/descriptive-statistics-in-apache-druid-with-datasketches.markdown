---
layout: post
title:  "Descriptive Statistics in Apache Druid with Datasketches"
categories: blog druid imply ststistics datasketches tutorial
---

```python
import time
import random
import json

def main():

    distr = {
        'A': { 'fn': random.gauss,       'param': (10000, 2000) },
        'B': { 'fn': random.gauss,       'param': (10000, 10000) },
        'C': { 'fn': random.expovariate, 'param': (0.0001,) },
        'D': { 'fn': random.gauss,       'param': (20000, 2000) },
        'E': { 'fn': random.gauss,       'param': (25000, 2500) },
    }
    for i in range(0, 10000):
        grp = random.choices('ABCDE', cum_weights=(0.50, 0.75, 0.82, 0.91, 1.00), k=1)[0]
        rec = {
            'tn': time.time() - 10000 + i,
            'cn': grp,
            'rn': distr[grp]['fn'](*distr[grp]['param'])
        }
        print(json.dumps(rec))

if __name__ == "__main__":
    main()

```


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

