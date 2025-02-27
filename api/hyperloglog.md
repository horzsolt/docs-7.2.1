---
api_name: hyperloglog()
excerpt: Aggregate data into a hyperloglog for approximate counting
license: community
toolkit: true
topic: hyperfunctions
tags: [hyperfunctions, approximate count distinct, distinct count, hyperloglog]
api_category: hyperfunction
api_experimental: false
hyperfunction_toolkit: true
hyperfunction_family: 'approximate count distinct'
hyperfunction_subfamily: hyperloglog
hyperfunction_type: aggregate
---

# hyperloglog()  <tag type="toolkit">Toolkit</tag>
The `hyperloglog` function constructs and returns a hyperloglog with at least
the specified number of buckets over the given values.

For more information about approximate count distinct functions, see the
[hyperfunctions documentation][hyperfunctions-approx-count-distincts].

## Required arguments

|Name|Type|Description|
|-|-|-|
|buckets|integer|Number of buckets in the digest. Rounded up to the next power of 2. Must be between 16 and 2^18, values less than 1024 are not recommended. If unsure, start experimenting with 32768 (2^15).|
|value|AnyElement| Column to count distinct elements. The type must have an extended, 64-bit, hash function.|

Increasing the `buckets` argument usually provides more accuracy at the expense
of more memory usage. Because hyperloglog is a probabilistic algorithm, it works
best on datasets that have many distinct values: at least tens of thousands. 

See [stderror][stderror] for how estimated error rate is related to `buckets`.

## Returns

|Column|Type|Description|
|-|-|-|
|hyperloglog|hyperloglog|A hyperloglog object which can be passed to other hyperloglog APIs.|

<!---Any special notes about the returns-->

## Sample usage
This examples assumes you have a table called `samples`, that contains a column
called `weights` that holds DOUBLE PRECISION values. This command returns a
digest over that column:

``` sql
SELECT hyperloglog(32768, weights) FROM samples;
```

Alternatively, you can build a view from the aggregate that you can pass to
other `hyperloglog` functions:

``` sql
CREATE VIEW hll AS SELECT hyperloglog(32768, data) FROM samples;
```


[hyperfunctions-approx-count-distincts]: /timescaledb/:currentVersion:/how-to-guides/hyperfunctions/approx-count-distincts/
[stderror]: /api/:currentVersion:/hyperfunctions/approx_count_distincts/stderror/
