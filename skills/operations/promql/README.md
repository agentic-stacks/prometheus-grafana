# PromQL Query Language

## Selectors

### Instant Vector Selectors

An instant vector selector selects a set of time series and a single sample value for each at a given timestamp (the most recent sample before the query evaluation timestamp).

**Basic syntax** -- select by metric name:

```promql
http_requests_total
```

**With label matchers** -- curly braces filter by label values:

```promql
http_requests_total{job="prometheus",group="canary"}
```

#### Label Matching Operators

| Operator | Meaning |
|----------|---------|
| `=` | Select labels that are exactly equal to the provided string |
| `!=` | Select labels that are not equal to the provided string |
| `=~` | Select labels that regex-match the provided string |
| `!~` | Select labels that do not regex-match the provided string |

Regex matches are fully anchored. A match of `env=~"foo"` is treated as `env=~"^foo$"`.

Examples:

```promql
http_requests_total{environment=~"staging|testing|development",method!="GET"}
```

The metric name can be matched with the internal `__name__` label:

```promql
{__name__=~"job:.*"}
```

At least one matcher must not match the empty string. The selectors `{job=~".*"}` and `{job=""}` are not valid because they match the empty string.

### Range Vector Selectors

Range vector selectors return a set of time series containing a range of data points over time for each series. Append a duration in square brackets to an instant vector selector:

```promql
http_requests_total{job="prometheus"}[5m]
```

#### Time Duration Units

| Unit | Meaning |
|------|---------|
| `ms` | milliseconds |
| `s` | seconds (1s = 1000ms) |
| `m` | minutes (1m = 60s) |
| `h` | hours (1h = 60m) |
| `d` | days (1d = 24h, assuming exact days) |
| `w` | weeks (1w = 7d) |
| `y` | years (1y = 365d) |

Durations can be combined: `1h30m`, `12h34m56s`, `54s321ms`. Units must be ordered from longest to shortest, and each unit must appear only once.

### Offset Modifier

The `offset` modifier changes the time offset for individual instant and range vectors in a query.

```promql
http_requests_total offset 5m
```

Returns the value of `http_requests_total` 5 minutes in the past relative to the current evaluation time. Applied to range vectors:

```promql
rate(http_requests_total[5m] offset 1w)
```

Negative offsets look forward in time:

```promql
rate(http_requests_total[5m] offset -1w)
```

### @ Modifier

The `@` modifier sets the evaluation time for individual instant and range vectors to a specific Unix timestamp:

```promql
http_requests_total @ 1609746000
```

```promql
rate(http_requests_total[5m] @ 1609746000)
```

Special values `start()` and `end()` reference the boundaries of the query:

```promql
http_requests_total @ start()
http_requests_total @ end()
```

The `offset` and `@` modifiers can be combined in any order:

```promql
http_requests_total @ 1609746000 offset 5m
http_requests_total offset 5m @ 1609746000
```

## Operators

### Arithmetic Binary Operators

| Operator | Description |
|----------|-------------|
| `+` | addition |
| `-` | subtraction |
| `*` | multiplication |
| `/` | division |
| `%` | modulo |
| `^` | exponentiation |

Arithmetic operators are defined between scalar/scalar, vector/scalar, and vector/vector value pairs. Between two instant vectors, the operation applies to matching entries in the left and right vectors. The metric name is dropped in the result.

### Comparison Binary Operators

| Operator | Description |
|----------|-------------|
| `==` | equal |
| `!=` | not equal |
| `>` | greater than |
| `<` | less than |
| `>=` | greater or equal |
| `<=` | less or equal |

By default, comparison operators filter: they drop vector elements for which the comparison is false. Adding the `bool` modifier causes the operator to return `0` (false) or `1` (true) instead of filtering:

```promql
http_requests_total > bool 1000
```

### Logical/Set Binary Operators

These operators work only between instant vectors:

| Operator | Description |
|----------|-------------|
| `and` | Intersection: returns elements from the left vector that have matching label sets in the right vector. Values come from the left side. |
| `or` | Union: returns all elements from the left vector plus elements from the right vector that have no matching label set on the left. |
| `unless` | Complement: returns elements from the left vector that have no matching label sets in the right vector. |

### Aggregation Operators

Aggregation operators can be used to aggregate the elements of a single instant vector, resulting in a new vector with fewer elements.

**Syntax:**

```promql
<aggr-op> [without|by (<label list>)] ([parameter,] <vector expression>)
```

or equivalently:

```promql
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

- `by (<labels>)` keeps only the listed labels and drops all others.
- `without (<labels>)` removes the listed labels and preserves all others.

| Operator | Signature | Description |
|----------|-----------|-------------|
| `sum` | `sum(v instant-vector)` | Sum of sample values |
| `min` | `min(v instant-vector)` | Minimum sample value |
| `max` | `max(v instant-vector)` | Maximum sample value |
| `avg` | `avg(v instant-vector)` | Arithmetic average of sample values |
| `count` | `count(v instant-vector)` | Number of elements in the vector |
| `group` | `group(v instant-vector)` | All values in the resulting vector are 1 |
| `stddev` | `stddev(v instant-vector)` | Population standard deviation of sample values |
| `stdvar` | `stdvar(v instant-vector)` | Population standard variance of sample values |
| `topk` | `topk(k integer, v instant-vector)` | Largest k elements by sample value |
| `bottomk` | `bottomk(k integer, v instant-vector)` | Smallest k elements by sample value |
| `quantile` | `quantile(phi float, v instant-vector)` | phi-quantile (0 <= phi <= 1) of sample values |
| `count_values` | `count_values(label string, v instant-vector)` | Number of elements with the same value; the value is placed in the named label |

Examples:

```promql
sum by (job) (rate(http_requests_total[5m]))

count without (instance) (up)

topk(3, sum by (app, proc) (rate(instance_cpu_time_ns[5m])))
```

### Operator Precedence

From highest to lowest:

1. `^` (right-associative)
2. `*`, `/`, `%`, `atan2`
3. `+`, `-`
4. `==`, `!=`, `<=`, `<`, `>=`, `>`
5. `and`, `unless`
6. `or`

Operators at the same precedence level are left-associative, except for `^` which is right-associative: `2 ^ 3 ^ 2` is equivalent to `2 ^ (3 ^ 2)`.

## Vector Matching

### One-to-One Vector Matching

By default, binary operations match entries where the complete set of labels is identical on both sides (excluding the metric name). The `on` and `ignoring` keywords control which labels are used for matching:

```promql
<vector expr> <bin-op> ignoring(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) <vector expr>
```

- `on(<label list>)` -- match only on the specified labels.
- `ignoring(<label list>)` -- match on all labels except those listed.

Example -- matching HTTP requests to failures ignoring the `method` label:

```promql
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```

### Many-to-One and One-to-Many Vector Matching

When one side of the operation can match multiple elements on the other side, use `group_left` or `group_right` to specify which side has higher cardinality:

```promql
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
```

- `group_left` -- many-to-one: the left vector has higher cardinality. Each element on the right can match multiple elements on the left.
- `group_right` -- one-to-many: the right vector has higher cardinality. Each element on the left can match multiple elements on the right.

The label list in the group modifier specifies additional labels from the "one" side to copy into the result.

Example:

```promql
method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
```

Here, each `method` value on the right matches multiple `(method, code)` pairs on the left. `group_left` declares the left side has higher cardinality.

## Functions Reference

### Instant Vector Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `abs` | `abs(v instant-vector)` | Returns absolute value of all sample values |
| `absent` | `absent(v instant-vector)` | Returns a 1-element vector with value 1 if the input vector is empty |
| `ceil` | `ceil(v instant-vector)` | Rounds sample values up to the nearest integer |
| `clamp` | `clamp(v instant-vector, min scalar, max scalar)` | Clamps sample values between min and max bounds |
| `clamp_max` | `clamp_max(v instant-vector, max scalar)` | Clamps the sample values to an upper bound |
| `clamp_min` | `clamp_min(v instant-vector, min scalar)` | Clamps the sample values to a lower bound |
| `exp` | `exp(v instant-vector)` | Calculates the exponential function for all sample values |
| `floor` | `floor(v instant-vector)` | Rounds sample values down to the nearest integer |
| `ln` | `ln(v instant-vector)` | Calculates the natural logarithm for all sample values |
| `log2` | `log2(v instant-vector)` | Calculates the binary logarithm for all sample values |
| `log10` | `log10(v instant-vector)` | Calculates the decimal logarithm for all sample values |
| `round` | `round(v instant-vector, to_nearest=1 scalar)` | Rounds sample values to the nearest integer or to_nearest multiple |
| `scalar` | `scalar(v instant-vector)` | Returns the sample value of a single-element vector as a scalar |
| `sgn` | `sgn(v instant-vector)` | Returns the sign of sample values: 1 if positive, -1 if negative, 0 if zero |
| `sort` | `sort(v instant-vector)` | Returns elements sorted by sample value in ascending order |
| `sort_desc` | `sort_desc(v instant-vector)` | Returns elements sorted by sample value in descending order |
| `sort_by_label` | `sort_by_label(v instant-vector, label string, ...)` | Returns elements sorted by label values in ascending order (experimental) |
| `sort_by_label_desc` | `sort_by_label_desc(v instant-vector, label string, ...)` | Returns elements sorted by label values in descending order (experimental) |
| `sqrt` | `sqrt(v instant-vector)` | Calculates the square root of all sample values |
| `vector` | `vector(s scalar)` | Returns the scalar as a vector with no labels |

### Range Vector Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `absent_over_time` | `absent_over_time(v range-vector)` | Returns a 1-element vector with value 1 if the range vector has no elements |
| `changes` | `changes(v range-vector)` | Returns the number of times the value has changed within the time range |
| `delta` | `delta(v range-vector)` | Calculates the difference between the first and last value of a range vector (for gauges) |
| `deriv` | `deriv(v range-vector)` | Calculates the per-second derivative using simple linear regression (for gauges) |
| `double_exponential_smoothing` | `double_exponential_smoothing(v range-vector, sf scalar, tf scalar)` | Produces a smoothed value for time series (experimental, requires feature flag) |
| `idelta` | `idelta(v range-vector)` | Calculates the difference between the last two samples of a range vector (for gauges) |
| `increase` | `increase(v range-vector)` | Calculates the increase in value over the time range (for counters) |
| `irate` | `irate(v range-vector)` | Calculates the per-second instant rate using the last two data points (for counters) |
| `predict_linear` | `predict_linear(v range-vector, t scalar)` | Predicts the value t seconds from now using simple linear regression (for gauges) |
| `rate` | `rate(v range-vector)` | Calculates the per-second average rate of increase over the time range (for counters) |
| `resets` | `resets(v range-vector)` | Returns the number of counter resets within the time range |

### Histogram Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `histogram_avg` | `histogram_avg(v instant-vector)` | Returns the arithmetic average of observed values in a native histogram |
| `histogram_count` | `histogram_count(v instant-vector)` | Returns the count of observations in a native histogram |
| `histogram_fraction` | `histogram_fraction(lower scalar, upper scalar, v instant-vector)` | Returns the estimated fraction of observations between lower and upper bounds |
| `histogram_quantile` | `histogram_quantile(phi scalar, b instant-vector)` | Calculates the phi-quantile (0 <= phi <= 1) from a histogram |
| `histogram_stddev` | `histogram_stddev(v instant-vector)` | Returns the estimated standard deviation of observed values in a native histogram |
| `histogram_stdvar` | `histogram_stdvar(v instant-vector)` | Returns the estimated standard variance of observed values in a native histogram |
| `histogram_sum` | `histogram_sum(v instant-vector)` | Returns the sum of observed values in a native histogram |

### Time Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `day_of_month` | `day_of_month(v=vector(time()) instant-vector)` | Returns the day of the month in UTC (1--31) |
| `day_of_week` | `day_of_week(v=vector(time()) instant-vector)` | Returns the day of the week in UTC (0=Sunday, 6=Saturday) |
| `day_of_year` | `day_of_year(v=vector(time()) instant-vector)` | Returns the day of the year in UTC (1--366) |
| `days_in_month` | `days_in_month(v=vector(time()) instant-vector)` | Returns the number of days in the month in UTC (28--31) |
| `hour` | `hour(v=vector(time()) instant-vector)` | Returns the hour of the day in UTC (0--23) |
| `minute` | `minute(v=vector(time()) instant-vector)` | Returns the minute of the hour in UTC (0--59) |
| `month` | `month(v=vector(time()) instant-vector)` | Returns the month of the year in UTC (1--12) |
| `time` | `time()` | Returns the number of seconds since January 1, 1970 UTC |
| `timestamp` | `timestamp(v instant-vector)` | Returns the timestamp of each sample as the number of seconds since epoch |
| `year` | `year(v=vector(time()) instant-vector)` | Returns the year in UTC |

### Label Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `label_join` | `label_join(v instant-vector, dst_label string, separator string, src_label_1 string, src_label_2 string, ...)` | Joins the values of source labels with the separator and stores the result in the destination label |
| `label_replace` | `label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)` | Matches regex against the source label and writes the replacement into the destination label (with `$1`, `$2`, etc. for capture groups) |
| `info` | `info(v instant-vector, data-label-selector instant-vector)` | Adds informational labels from info metrics to series (experimental) |

### Aggregation Over Time Functions (for Range Vectors)

| Function | Signature | Description |
|----------|-----------|-------------|
| `avg_over_time` | `avg_over_time(range-vector)` | Average value of all samples in the time range |
| `min_over_time` | `min_over_time(range-vector)` | Minimum value of all float samples in the time range |
| `max_over_time` | `max_over_time(range-vector)` | Maximum value of all float samples in the time range |
| `sum_over_time` | `sum_over_time(range-vector)` | Sum of all sample values in the time range |
| `count_over_time` | `count_over_time(range-vector)` | Count of all samples in the time range |
| `quantile_over_time` | `quantile_over_time(scalar, range-vector)` | phi-quantile of all float samples in the time range |
| `stddev_over_time` | `stddev_over_time(range-vector)` | Population standard deviation of all float samples in the time range |
| `stdvar_over_time` | `stdvar_over_time(range-vector)` | Population standard variance of all float samples in the time range |
| `mad_over_time` | `mad_over_time(range-vector)` | Median absolute deviation of all float samples in the time range (experimental) |
| `last_over_time` | `last_over_time(range-vector)` | Most recent sample value in the time range |
| `present_over_time` | `present_over_time(range-vector)` | Returns value 1 for any series that has samples in the time range |

### Trigonometric Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `acos` | `acos(v instant-vector)` | Calculates the arccosine of all sample values (in radians) |
| `acosh` | `acosh(v instant-vector)` | Calculates the inverse hyperbolic cosine of all sample values |
| `asin` | `asin(v instant-vector)` | Calculates the arcsine of all sample values (in radians) |
| `asinh` | `asinh(v instant-vector)` | Calculates the inverse hyperbolic sine of all sample values |
| `atan` | `atan(v instant-vector)` | Calculates the arctangent of all sample values (in radians) |
| `atanh` | `atanh(v instant-vector)` | Calculates the inverse hyperbolic tangent of all sample values |
| `cos` | `cos(v instant-vector)` | Calculates the cosine of all sample values (in radians) |
| `cosh` | `cosh(v instant-vector)` | Calculates the hyperbolic cosine of all sample values |
| `sin` | `sin(v instant-vector)` | Calculates the sine of all sample values (in radians) |
| `sinh` | `sinh(v instant-vector)` | Calculates the hyperbolic sine of all sample values |
| `tan` | `tan(v instant-vector)` | Calculates the tangent of all sample values (in radians) |
| `tanh` | `tanh(v instant-vector)` | Calculates the hyperbolic tangent of all sample values |
| `deg` | `deg(v instant-vector)` | Converts radians to degrees for all sample values |
| `rad` | `rad(v instant-vector)` | Converts degrees to radians for all sample values |
| `pi` | `pi()` | Returns the value of pi |

## Commonly Used Functions

### rate()

Calculates the per-second average rate of increase of a counter over the specified time range. It automatically handles counter resets (when a counter restarts from zero after a process restart).

```promql
rate(http_requests_total[5m])
```

- Should only be used with counters and native histograms where the samples are counters.
- The range window should be at least four times the scrape interval to account for brief outages.
- When combined with aggregation, always apply `rate()` first, then aggregate:

```promql
sum by (job) (rate(http_requests_total[5m]))
```

#### rate() vs. irate()

- `rate()` uses the first and last data points in the range, computing an average rate. Best for alerting and slow-moving graphs.
- `irate()` uses the last two data points only, providing an instant rate. Best for volatile, fast-moving counters and dashboards.

```promql
irate(http_requests_total[5m])
```

With `irate()`, the range window is only used to determine how far back to look for two data points. A wider window makes the query more resilient to scrape failures but does not affect the rate calculation itself.

### histogram_quantile()

Calculates the phi-quantile from the buckets of a classic or native histogram. The value of phi must be between 0 and 1 (inclusive).

For classic histograms, the metric must have `le` (less-than-or-equal) bucket labels:

```promql
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

Aggregate across labels while preserving `le`:

```promql
histogram_quantile(0.99, sum by (job, le) (rate(http_request_duration_seconds_bucket[5m])))
```

The function interpolates quantile values linearly between bucket boundaries. If the quantile falls into the highest bucket (`+Inf`), it returns the lower bound of that bucket. If the quantile falls into a bucket with an upper bound of 0 (for histograms covering negative values), the upper bound of the lowest bucket is returned.

### increase()

Returns the increase in value of a counter over the specified time range. It is syntactically equivalent to `rate()` multiplied by the number of seconds in the range window:

```promql
increase(http_requests_total[5m])
```

This is equivalent to:

```promql
rate(http_requests_total[5m]) * 300
```

Like `rate()`, `increase()` handles counter resets. The result is not necessarily an integer even for integer counters, because the boundary values are estimated based on the actual data points in the range.

### label_replace() and label_join()

`label_replace()` matches a regex against a source label and writes the replacement (with `$1` capture groups) into a destination label:

```promql
label_replace(up{job="api-server", instance="0:8080"}, "foo", "$1", "instance", "(.*):.*")
```

This produces a new label `foo` with value `0` by capturing everything before the colon.

`label_join()` concatenates the values of multiple source labels using a separator:

```promql
label_join(up{job="api-server",instance="0:8080"}, "address", ":", "instance", "job")
```

This produces a label `address` with value `0:8080:api-server`.

## Practical Examples

### CPU Usage Percentage

Per-instance CPU usage percentage (across all CPU modes except idle):

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Memory Usage

Unused memory per instance in MiB:

```promql
(instance_memory_limit_bytes - instance_memory_usage_bytes) / 1024 / 1024
```

Grouped by application and process type:

```promql
sum by (app, proc) (instance_memory_limit_bytes - instance_memory_usage_bytes) / 1024 / 1024
```

### Request Rate

Per-second HTTP request rate over the last 5 minutes:

```promql
rate(http_requests_total[5m])
```

Summed by job:

```promql
sum by (job) (rate(http_requests_total[5m]))
```

### Error Rate Percentage

Percentage of HTTP requests that are 500 errors, using vector matching:

```promql
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```

Percentage of requests returning 5xx status codes:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

### 99th Percentile Latency

99th percentile request duration from a classic histogram, grouped by job:

```promql
histogram_quantile(0.99, sum by (job, le) (rate(http_request_duration_seconds_bucket[5m])))
```

### Top 5 by Resource Usage

Top 5 CPU-consuming applications by app and process type:

```promql
topk(5, sum by (app, proc) (rate(instance_cpu_time_ns[5m])))
```

Count of running instances per application:

```promql
count by (app) (instance_cpu_time_ns)
```

### Subquery Examples

5-minute rate computed over 30 minutes at 1-minute resolution:

```promql
rate(http_requests_total[5m])[30m:1m]
```

Nested subquery -- maximum acceleration from distance data:

```promql
max_over_time(deriv(rate(distance_covered_total[5s])[30s:5s])[10m:])
```

## Version Notes

### Prometheus v3.x

Prometheus 3.0 was released in November 2024 as a major version update. Key PromQL-related changes and additions:

- **Native histograms** (experimental since 2.40, maturing in 3.x): `histogram_avg()`, `histogram_count()`, `histogram_sum()`, `histogram_fraction()`, `histogram_stddev()`, `histogram_stdvar()` functions operate on native histogram samples. Native histograms store bucket boundaries and counts more efficiently than classic histograms with explicit `le` buckets.
- **UTF-8 metric and label names**: Prometheus 3.x supports full UTF-8 in metric and label names, with backward-compatible escaping for older consumers.
- **`info()` function** (experimental): Joins informational metric labels onto time series, replacing manual label-matching workarounds.
- **`double_exponential_smoothing()`** (experimental): Replaces the deprecated `holt_winters()` function for double exponential smoothing of time series.
- **Experimental sorting functions**: `sort_by_label()` and `sort_by_label_desc()` allow sorting results by label values rather than sample values.
- **Experimental `*_over_time` timestamp functions**: `ts_of_min_over_time()`, `ts_of_max_over_time()`, `ts_of_first_over_time()`, `ts_of_last_over_time()` return the timestamp of the selected sample. Requires `--enable-feature=promql-experimental-functions`.
- **Experimental aggregation operators**: `limitk(k, v)` returns a pseudo-random sample of k elements; `limit_ratio(r, v)` returns approximately ratio r of elements.
- **Experimental fill modifiers** for binary operators: `fill(<value>)`, `fill_left(<value>)`, `fill_right(<value>)` replace missing matches. Requires `--enable-feature=promql-binop-fill-modifiers`.
- **`mad_over_time()`** (experimental): Median absolute deviation of float samples over time.
- **`first_over_time()`** (experimental): Returns the oldest sample value in the specified time range.
