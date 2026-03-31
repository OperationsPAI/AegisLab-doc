---
title: Simple RCA
weight: 1
---

A basic heuristic-based root cause analysis algorithm that demonstrates the core concepts.

## Overview

The `simplerca` algorithm uses simple heuristics to identify root causes:
1. Find spans with errors
2. Calculate error rates per service
3. Rank services by error rate and span count

## Implementation

### Algorithm Class

```python
from rcabench_platform.v3.sdk.algorithms.spec import Algorithm, AlgorithmArgs, AlgorithmAnswer
import polars as pl

class SimpleRCA(Algorithm):
    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        # Load trace data
        traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

        # Filter error spans
        error_spans = traces.filter(
            pl.col("status_code") == "ERROR"
        )

        # Calculate error rate per service
        service_stats = error_spans.group_by("service_name").agg([
            pl.count().alias("error_count"),
            pl.col("span_id").n_unique().alias("unique_spans")
        ])

        # Calculate score (error_count * unique_spans)
        service_stats = service_stats.with_columns(
            (pl.col("error_count") * pl.col("unique_spans")).alias("score")
        )

        # Sort by score descending
        ranked = service_stats.sort("score", descending=True)

        # Convert to list of AlgorithmAnswer objects
        return [
            AlgorithmAnswer(level="service", name=row["service_name"], rank=rank)
            for rank, row in enumerate(ranked.iter_rows(named=True), start=1)
        ]
```

### Key Concepts

**1. Loading Data**

```python
traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
```

Use Polars to read parquet files efficiently. The trace data contains:
- `trace_id`: Unique identifier for each trace
- `span_id`: Unique identifier for each span
- `service_name`: Name of the service
- `operation_name`: Name of the operation
- `status_code`: Status (OK, ERROR, UNSET)
- `start_time`: Span start timestamp
- `end_time`: Span end timestamp

**2. Filtering Errors**

```python
error_spans = traces.filter(pl.col("status_code") == "ERROR")
```

Focus on spans that have errors, as these are likely related to the root cause.

**3. Aggregating by Service**

```python
service_stats = error_spans.group_by("service_name").agg([
    pl.count().alias("error_count"),
    pl.col("span_id").n_unique().alias("unique_spans")
])
```

Group error spans by service and calculate statistics.

**4. Scoring and Ranking**

```python
service_stats = service_stats.with_columns(
    (pl.col("error_count") * pl.col("unique_spans")).alias("score")
)
ranked = service_stats.sort("score", descending=True)
```

Calculate a score for each service and rank them.

**5. Returning Results**

```python
return [
    AlgorithmAnswer(level="service", name=service_name, rank=rank)
    for rank, service_name in enumerate(ranked_services, start=1)
]
```

Return a list of `AlgorithmAnswer` objects, each with `level`, `name`, and `rank` fields, sorted by likelihood.

## Running the Example

### Local Evaluation

```bash
cd rcabench-platform
./main.py eval single simplerca trainticket-pandora-v1 0
```

Expected output:

```
Loading dataset: trainticket-pandora-v1, datapack: 0
Running algorithm: simplerca
Processing traces...
Algorithm completed in 0.85s

Results:
  MRR: 0.850
  Avg@1: 0.750
  Top-1 Accuracy: 0.750

Predicted root causes:
  1. ts-order-service (score: 45.0)
  2. ts-payment-service (score: 23.0)
  3. ts-user-service (score: 12.0)

Ground truth: ts-order-service
```

### Batch Evaluation

```bash
./main.py eval batch simplerca trainticket-pandora-v1 0 9
```

## Extending the Algorithm

### Add Latency Analysis

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

    # Calculate latency
    traces = traces.with_columns(
        (pl.col("end_time") - pl.col("start_time")).alias("latency")
    )

    # Find high-latency spans
    high_latency = traces.filter(pl.col("latency") > 1000)  # > 1 second

    # Combine with error analysis
    error_spans = traces.filter(pl.col("status_code") == "ERROR")

    # Aggregate both metrics
    service_stats = traces.group_by("service_name").agg([
        pl.col("status_code").filter(pl.col("status_code") == "ERROR").count().alias("error_count"),
        pl.col("latency").mean().alias("avg_latency"),
        pl.col("latency").max().alias("max_latency")
    ])

    # Calculate combined score
    service_stats = service_stats.with_columns(
        (pl.col("error_count") * 10 + pl.col("avg_latency") / 100).alias("score")
    )

    ranked = service_stats.sort("score", descending=True)

    return [
        AlgorithmAnswer(level="service", name=row["service_name"], rank=rank)
        for rank, row in enumerate(ranked.iter_rows(named=True), start=1)
    ]
```

### Add Dependency Analysis

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

    # Build service dependency graph
    dependencies = traces.group_by(["parent_service", "service_name"]).agg(
        pl.count().alias("call_count")
    )

    # Find services with many downstream errors
    error_spans = traces.filter(pl.col("status_code") == "ERROR")

    # Calculate propagation score for each service
    service_scores = {}
    for service in error_spans["service_name"].unique():
        # Count how many downstream services have errors
        downstream_errors = dependencies.filter(
            pl.col("parent_service") == service
        ).join(
            error_spans.select("service_name").unique(),
            on="service_name"
        ).height

        # Services with more downstream errors are more likely root causes
        service_scores[service] = downstream_errors

    # Sort services by score
    ranked_services = sorted(service_scores.items(), key=lambda x: x[1], reverse=True)

    # Return ranked services
    return [
        AlgorithmAnswer(level="service", name=service_name, rank=rank)
        for rank, (service_name, score) in enumerate(ranked_services, start=1)
    ]
```

## Common Patterns

### Lazy Evaluation for Large Datasets

```python
# Use scan_parquet instead of read_parquet
traces = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")

# Build query
result = traces.filter(
    pl.col("status_code") == "ERROR"
).group_by("service_name").agg(
    pl.count().alias("error_count")
).sort("error_count", descending=True).collect()  # Execute query
```

### Handling Missing Data

```python
# Fill missing values
traces = traces.with_columns(
    pl.col("status_code").fill_null("UNSET")
)

# Filter out invalid spans
traces = traces.filter(
    pl.col("end_time") > pl.col("start_time")
)
```

### Debugging Output

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    print(f"Processing {args.dataset}/{args.datapack}")

    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    print(f"Loaded {len(traces)} spans")

    error_spans = traces.filter(pl.col("status_code") == "ERROR")
    print(f"Found {len(error_spans)} error spans")

    # ... rest of algorithm

    results = [
        AlgorithmAnswer(level="service", name=service_name, rank=rank)
        for rank, service_name in enumerate(ranked_services, start=1)
    ]
    print(f"Ranked {len(results)} services")
    return results
```

## Performance Tips

1. **Use lazy evaluation**: `scan_parquet` instead of `read_parquet`
2. **Filter early**: Reduce data size before aggregations
3. **Avoid collect()**: Keep data in lazy frames as long as possible
4. **Use efficient aggregations**: Polars is optimized for group_by operations
5. **Profile your code**: Use `cProfile` to identify bottlenecks

## Next Steps

- [ML-Based RCA](../ml-based-rca): Learn about machine learning approaches
- [Contributing](../contributing): Share your algorithm with the community
- [Data Formats](../../development-guide/data-formats): Understand the data schema
