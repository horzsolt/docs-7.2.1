# Create a continuous aggregate
Creating a continuous aggregate is a two-step process. You need to create the
view first, then enable a policy to keep the view refreshed. You can have more
than one continuous aggregate on a single hypertable.

Continuous aggregates require a `time_bucket` on the time partitioning column of
the hypertable.

By default, views are automatically refreshed. You can adjust this by setting
the [WITH NO DATA](#using-the-with-no-data-option) option. Additionally, the
view can not be a [security barrier view][postgres-security-barrier].

## Create a continuous aggregate
In this example, we are using a hypertable called `conditions`, and creating a
continuous aggregate view for daily weather data. The `GROUP BY` clause must
include a `time_bucket` expression which uses time dimension column of the
hypertable. Additionally, all functions and their arguments included in
`SELECT`, `GROUP BY`, and `HAVING` clauses must be
[immutable][postgres-immutable].

<procedure>

### Creating a continuous aggregate
1.  At the `psql`prompt, create the materialized view:
    ```sql
    CREATE MATERIALIZED VIEW conditions_summary_daily
    WITH (timescaledb.continuous) AS
    SELECT device,
       time_bucket(INTERVAL '1 day', time) AS bucket,
       AVG(temperature),
       MAX(temperature),
       MIN(temperature)
    FROM conditions
    GROUP BY device, bucket;
    ```
1.  Create a policy to refresh the view every hour:
    ```sql
    SELECT add_continuous_aggregate_policy('conditions_summary_daily',
	     start_offset => INTERVAL '1 month',
	     end_offset => INTERVAL '1 day',
	     schedule_interval => INTERVAL '1 hour');
    ```

</procedure>

You can use most PostgreSQL aggregate functions in continuous aggregations. To
see what PostgreSQL features are supported, check the
[function support table][cagg-function-support].

## Choosing an appropriate bucket interval
Continuous aggregates require a `time_bucket` on the time partitioning column of
the hypertable. The time bucket allows you to define a time interval, instead of
having to use specific timestamps. For example, you can define a time bucket as
five minutes, or one day.

When the continuous aggregate is materialized, the materialization table stores
partials, which are then used to calculate the result of the query. This means a
certain amount of processing capacity is required for any query, and the amount
required becomes greater as the interval gets smaller. Because of this, if you
have very small intervals, it can be more efficient to run the aggregate query
on the raw data in the hypertable. We recommend that you test both methods to
determine what is best for your data set and desired bucket interval.

You can read more about the time bucket function in
our [API Guide][api-time-bucket]. If you want to use
[time_bucket_gapfill][api-time-bucket-gapfill], you need to run it in the
`SELECT` statement on the continuous aggregate view, you can not run it in the
continuous aggregate directly.

## Using the WITH NO DATA option
By default, when you create a view for the first time, it is populated with
data. This is so that the aggregates can be computed across the entire
hypertable. If you don't want this to happen, for example if the table is very
large, or if new data is being continuously added, you can control the order in
which the data is refreshed. You can do this by adding a manual refresh with
your continuous aggregate policy using the `WITH NO DATA` option.

The `WITH NO DATA` option allows the continuous aggregate to be created
instantly, so you don't have to wait for the data to be aggregated. Data begins
to populate only when the policy begins to run. This means that only data newer
than the `start_offset` time begins to populate the continuous aggregate. If you
have historical data that is older than the `start_offset` interval, you need to
manually refresh the history up to the current `start_offset` to allow real-time
queries to run efficiently.

<procedure>

### Creating a continuous aggregate with the WITH NO DATA option
1.  At the `psql` prompt, create the view:
    ```sql
    CREATE MATERIALIZED VIEW cagg_rides_view
    WITH (timescaledb.continuous) AS
    SELECT vendor_id,
    time_bucket('1h', pickup_datetime) AS day,
      count(*) total_rides,
      avg(fare_amount) avg_fare,
      max(trip_distance) as max_trip_distance,
      min(trip_distance) as min_trip_distance
    FROM rides
    GROUP BY vendor_id, time_bucket('1h', pickup_datetime)
    WITH NO DATA;
    ```
1.  Manually refresh the view:
    ```sql
    CALL refresh_continuous_aggregate('cagg_rides_view', NULL, localtimestamp - INTERVAL '1 week');
    ```
1.  Add the policy:
    ```sql
    SELECT add_continuous_aggregate_policy('cagg_rides_view',
      start_offset => INTERVAL '1 week',
      end_offset   => INTERVAL '1 hour',
      schedule_interval => INTERVAL '30 minutes');
    ```
  
</procedure>

## Query continuous aggregates
When you have created a continuous aggregate and set a refresh policy, you can
query the view with a `SELECT` query. You can only specify a single hypertable
in the `FROM` clause. Including more hypertables, joins, tables, views, or
subqueries in your `SELECT` query is not supported. Additionally, make sure that
the hypertable you are querying does not have [row-level-security
policies][postgres-rls] enabled.

<procedure>

### Querying a continuous aggregate
1.  At the `psql` prompt, query the continuous aggregate view called
    `conditions_summary_hourly` for the average, minimum, and maximum
    temperatures for the first quarter of 2021 recorded by device 5:
    ```sql
    SELECT *
      FROM conditions_summary_hourly
      WHERE device = 5
      AND bucket >= '2020-01-01'
      AND bucket < '2020-04-01';
    ```
1.  Alternatively, query the continuous aggregate view called
    `conditions_summary_hourly` for the top 20 largest metric spreads in that
    quarter:
    ```sql
    SELECT *
      FROM conditions_summary_hourly
      WHERE max - min > 1800
      AND bucket >= '2020-01-01' AND bucket < '2020-04-01'
      ORDER BY bucket DESC, device DESC LIMIT 20;
    ```

</procedure>

## Use continuous aggregates with window functions
Continuous aggregates don't currently support window functions. You can work
around this by:
1.  Creating a continuous aggregate for the other parts of your query, then
1.  Using the window function on your continuous aggregate at query time

For example, say you have a hypertable named `example` with a `time` column and
a `value` column. You bucket your data by `time` and calculate the delta between
time buckets using the `lag` window function:
```sql
WITH t AS (
  SELECT
    time_bucket('10 minutes', time) as bucket,
    first(value, time) as value
  FROM example GROUP BY bucket
)
SELECT
  bucket,
  value - lag(value, 1) OVER (ORDER BY bucket) delta
  FROM t;
```

You can't create a continuous aggregate using this query, because it contains
the `lag` function. But you can create a continuous aggregate by excluding the
`lag` function:

```sql
CREATE MATERIALIZED VIEW example_aggregate
  WITH (timescaledb.continuous) AS
    SELECT
      time_bucket('10 minutes', time) AS bucket,
      first(value, time) AS value
    FROM example GROUP BY bucket;
```

Then, at query time, calculate the delta by using `lag` on your continuous
aggregate:

```sql
SELECT
  bucket,
  value - lag(value, 1) OVER (ORDER BY bucket) AS delta
FROM example_aggregate;
```

This speeds up your query by calculating the aggregation ahead of time. The
delta still needs to be calculated at query time.

[api-time-bucket]: /api/:currentVersion:/hyperfunctions/time_bucket/
[api-time-bucket-gapfill]: /api/:currentVersion:/hyperfunctions/gapfilling-interpolation/time_bucket_gapfill/
[cagg-function-support]: /timescaledb/:currentVersion:/how-to-guides/continuous-aggregates/about-continuous-aggregates/#function-support
[postgres-security-barrier]: https://www.postgresql.org/docs/current/rules-privileges.html
[postgres-immutable]: https://www.postgresql.org/docs/current/xfunc-volatility.html
[postgres-parallel-agg]: https://www.postgresql.org/docs/current/parallel-plans.html#PARALLEL-AGGREGATION
[postgres-rls]: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
