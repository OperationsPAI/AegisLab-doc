---
title: Development Guide
weight: 2
---


This guide covers the essential concepts and patterns for developing RCA algorithms with rcabench-platform.

## Topics

{{< cards >}}
  {{< card link="algorithm-interface" title="Algorithm Interface" icon="code" >}}
  {{< card link="data-formats" title="Data Formats" icon="table" >}}
  {{< card link="containerization" title="Containerization" icon="cube" >}}
  {{< card link="best-practices" title="Best Practices" icon="light-bulb" >}}
{{< /cards >}}

## Overview

Developing an RCA algorithm involves:

1. **Implementing the interface**: Inherit from `Algorithm` base class
2. **Processing input data**: Parse traces, metrics, and logs from parquet files
3. **Generating predictions**: Return ranked list of root cause candidates
4. **Containerizing**: Package as Docker image with metadata
5. **Testing**: Evaluate locally before remote submission

## Quick Example

Here's a minimal RCA algorithm:

```python
from rcabench_platform.v3.sdk.algorithms.spec import Algorithm, AlgorithmArgs, AlgorithmAnswer
import polars as pl

class SimpleRCA(Algorithm):
    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        # Read trace data
        traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

        # Analyze error spans
        error_spans = traces.filter(pl.col("status_code") == "ERROR")

        # Count errors by service
        service_errors = (
            error_spans
            .group_by("service_name")
            .agg(pl.count().alias("error_count"))
            .sort("error_count", descending=True)
        )

        # Return ranked services
        return [
            AlgorithmAnswer(level="service", name=service_name, rank=rank)
            for rank, service_name in enumerate(service_errors["service_name"].to_list(), start=1)
        ]
```

## Key Concepts

### Algorithm Lifecycle

```
Input Data → Algorithm.__call__() → Ranked Predictions → Evaluation Metrics
```

1. **Input**: `AlgorithmArgs` with paths to parquet files
2. **Processing**: Your algorithm logic
3. **Output**: `AlgorithmAnswer` with ranked root cause candidates
4. **Evaluation**: Platform compares predictions with ground truth

### Data Processing Patterns

Use Polars for efficient data processing:

```python
import polars as pl

# Lazy evaluation for large datasets
traces = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")

# Filter and aggregate
result = (
    traces
    .filter(pl.col("timestamp") > start_time)
    .group_by("service_name")
    .agg(pl.mean("duration").alias("avg_duration"))
    .collect()  # Execute lazy query
)
```

### Error Handling

Always handle missing or malformed data:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    try:
        traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

        if traces.is_empty():
            return []

        # Your algorithm logic

    except Exception as e:
        # Log error and return empty result
        print(f"Error processing traces: {e}")
        return []
```

## Next Steps

- [Algorithm Interface](algorithm-interface): Detailed API reference
- [Data Formats](data-formats): Understanding input parquet schemas
- [Containerization](containerization): Packaging for remote execution
- [Best Practices](best-practices): Performance and debugging tips
