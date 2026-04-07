---
title: Debugging Tips
weight: 2
---

Strategies and techniques for debugging RCA algorithms.

## General Debugging Approach

### 1. Start Small

Test with minimal data before scaling up:

```python
# Test with single datapack first
./main.py eval single my-rca trainticket-pandora-v1 0

# Then small batch
./main.py eval batch my-rca trainticket-pandora-v1 0 9

# Finally full evaluation
./main.py eval batch my-rca trainticket-pandora-v1 0 99
```

### 2. Add Logging

Insert print statements at key points:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    print(f"[DEBUG] Processing {args.dataset}/{args.datapack}")

    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    print(f"[DEBUG] Loaded {len(traces)} traces")

    error_spans = traces.filter(pl.col("status_code") == "ERROR")
    print(f"[DEBUG] Found {len(error_spans)} error spans")

    if len(error_spans) == 0:
        print("[WARNING] No error spans found!")

    # Continue processing...
    results = [
        AlgorithmAnswer(level="service", name=service_name, rank=rank)
        for rank, service_name in enumerate(ranked_services, start=1)
    ]
    print(f"[DEBUG] Ranked {len(results)} services")

    return results
```

### 3. Inspect Data

Examine input data structure:

```python
# Check schema
traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
print(traces.schema)

# View sample data
print(traces.head(10))

# Check for nulls
print(traces.null_count())

# Examine specific columns
print(traces.select(["service_name", "status_code"]).unique())
```

## Debugging Techniques

### Interactive Debugging with pdb

```python
import pdb

def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

    # Set breakpoint
    pdb.set_trace()

    # Now you can:
    # - Inspect variables: print(traces)
    # - Step through code: n (next), s (step into)
    # - Continue execution: c
    # - Evaluate expressions: p len(traces)

    error_spans = traces.filter(pl.col("status_code") == "ERROR")

    results = [
        AlgorithmAnswer(level="service", name=service_name, rank=rank)
        for rank, service_name in enumerate(ranked_services, start=1)
    ]
    return results
```

### Using IPython for Exploration

```python
# Create test script: test_algorithm.py
from rcabench_platform.v3.sdk.algorithms.spec import AlgorithmArgs
from my_algorithm import MyRCA
import polars as pl

args = AlgorithmArgs(
    dataset="trainticket-pandora-v1",
    datapack="ts0-ts-auth-service-stress-jv8m9r",
    input_folder=Path("data/rcabench-platform-v2/data/trainticket-pandora-v1/ts0-ts-auth-service-stress-jv8m9r"),
    output_folder=Path("output")
)

# Load data
traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

# Now explore interactively
# IPython: ipython -i test_algorithm.py
```

### Logging to File

```python
import logging

# Configure logging
logging.basicConfig(
    filename='algorithm_debug.log',
    level=logging.DEBUG,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    logging.info(f"Processing {args.dataset}/{args.datapack}")

    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    logging.debug(f"Loaded {len(traces)} traces")

    # Algorithm logic

    logging.info(f"Completed with {len(ranked_services)} ranked services")
    return [
        AlgorithmAnswer(level="service", name=service_name, rank=rank)
        for rank, service_name in enumerate(ranked_services, start=1)
    ]
```

## Common Debugging Scenarios

### Empty Results

**Problem**: Algorithm returns empty ranked_services.

**Debug steps**:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    print(f"Total traces: {len(traces)}")

    # Check for errors
    error_spans = traces.filter(pl.col("status_code") == "ERROR")
    print(f"Error spans: {len(error_spans)}")

    if len(error_spans) == 0:
        print("No errors found - checking status codes:")
        print(traces.select("status_code").unique())
        # Maybe status codes are different? (e.g., "FAILED" instead of "ERROR")

    # Check services
    services = traces.select("service_name").unique()
    print(f"Unique services: {len(services)}")
    print(services)

    # Continue debugging...
    return []
```

### Incorrect Rankings

**Problem**: Algorithm produces rankings but they're wrong.

**Debug steps**:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    # Load ground truth
    with open(args.input_folder / "injection.json") as f:
        injection = json.load(f)
    print(f"Ground truth: {injection['target_service']}")

    # Check if ground truth service appears in traces
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    gt_service = injection['target_service']

    gt_spans = traces.filter(pl.col("service_name") == gt_service)
    print(f"Ground truth service spans: {len(gt_spans)}")
    print(f"Ground truth service errors: {len(gt_spans.filter(pl.col('status_code') == 'ERROR'))}")

    # Compare with your ranking logic
    # ...
    return []
```

### Performance Issues

**Problem**: Algorithm is too slow.

**Debug steps**:

```python
import time

def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    start = time.time()

    # Time each operation
    t1 = time.time()
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    print(f"Load time: {time.time() - t1:.2f}s")

    t2 = time.time()
    error_spans = traces.filter(pl.col("status_code") == "ERROR")
    print(f"Filter time: {time.time() - t2:.2f}s")

    t3 = time.time()
    service_stats = error_spans.group_by("service_name").agg([...])
    print(f"Aggregation time: {time.time() - t3:.2f}s")

    print(f"Total time: {time.time() - start:.2f}s")

    return [
        AlgorithmAnswer(level="service", name=service_name, rank=rank)
        for rank, service_name in enumerate(ranked_services, start=1)
    ]
```

### Memory Issues

**Problem**: Algorithm uses too much memory.

**Debug steps**:

```python
import tracemalloc

def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    tracemalloc.start()

    # Check memory at each step
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    current, peak = tracemalloc.get_traced_memory()
    print(f"After load: {current / 1024**2:.1f} MB (peak: {peak / 1024**2:.1f} MB)")

    error_spans = traces.filter(pl.col("status_code") == "ERROR")
    current, peak = tracemalloc.get_traced_memory()
    print(f"After filter: {current / 1024**2:.1f} MB (peak: {peak / 1024**2:.1f} MB)")

    # If memory is high, try lazy evaluation
    # traces = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")

    tracemalloc.stop()
    return [
        AlgorithmAnswer(level="service", name=service_name, rank=rank)
        for rank, service_name in enumerate(ranked_services, start=1)
    ]
```

## Data Validation

### Check Data Quality

```python
def validate_data(traces):
    """Validate trace data quality."""

    # Check for required columns
    required_cols = ["trace_id", "span_id", "service_name", "status_code"]
    missing = [col for col in required_cols if col not in traces.columns]
    if missing:
        print(f"WARNING: Missing columns: {missing}")

    # Check for nulls
    null_counts = traces.null_count()
    for col in null_counts.columns:
        count = null_counts[col][0]
        if count > 0:
            print(f"WARNING: {count} nulls in {col}")

    # Check for invalid timestamps
    invalid_times = traces.filter(pl.col("end_time") < pl.col("start_time"))
    if len(invalid_times) > 0:
        print(f"WARNING: {len(invalid_times)} spans with invalid timestamps")

    # Check for duplicate span IDs
    duplicates = traces.group_by("span_id").agg(pl.count()).filter(pl.col("count") > 1)
    if len(duplicates) > 0:
        print(f"WARNING: {len(duplicates)} duplicate span IDs")

    return True
```

### Visualize Data

```python
def visualize_traces(traces):
    """Create simple visualizations for debugging."""

    # Service distribution
    service_counts = traces.group_by("service_name").agg(pl.count().alias("count"))
    print("\nService distribution:")
    for row in service_counts.sort("count", descending=True).iter_rows(named=True):
        print(f"  {row['service_name']}: {row['count']}")

    # Status code distribution
    status_counts = traces.group_by("status_code").agg(pl.count().alias("count"))
    print("\nStatus code distribution:")
    for row in status_counts.iter_rows(named=True):
        print(f"  {row['status_code']}: {row['count']}")

    # Latency distribution
    latencies = traces.with_columns(
        (pl.col("end_time") - pl.col("start_time")).alias("latency")
    )
    print("\nLatency statistics:")
    print(f"  Mean: {latencies['latency'].mean():.2f}ms")
    print(f"  Median: {latencies['latency'].median():.2f}ms")
    print(f"  P95: {latencies['latency'].quantile(0.95):.2f}ms")
    print(f"  Max: {latencies['latency'].max():.2f}ms")
```

## Testing Strategies

### Unit Tests

```python
# tests/test_my_rca.py
import pytest
import polars as pl
from my_rca import MyRCA
from rcabench_platform.v3.sdk.algorithms.spec import AlgorithmArgs

def test_basic_functionality():
    """Test algorithm with minimal data."""

    # Create minimal test data
    traces = pl.DataFrame({
        "trace_id": ["t1", "t1", "t2"],
        "span_id": ["s1", "s2", "s3"],
        "service_name": ["svc-a", "svc-b", "svc-a"],
        "status_code": ["OK", "ERROR", "OK"],
        "start_time": [1000, 1100, 2000],
        "end_time": [1050, 1150, 2050]
    })

    traces.write_parquet("test_trace.parquet")

    # Create ground truth
    gt = pl.DataFrame({"root_cause_service": ["svc-b"]})
    gt.write_parquet("test_gt.parquet")

    # Test algorithm
    args = AlgorithmArgs(
        trace_path="test_trace.parquet",
        ground_truth_path="test_gt.parquet",
        output_path="output/",
        dataset_name="test",
        datapack_id="0",
        benchmark="test"
    )

    algo = MyRCA()
    result = algo(args)

    # Assertions
    assert len(result.ranked_services) > 0
    assert result.ranked_services[0][0] == "svc-b"  # Top prediction should be svc-b

def test_empty_data():
    """Test algorithm with no errors."""

    traces = pl.DataFrame({
        "trace_id": ["t1"],
        "span_id": ["s1"],
        "service_name": ["svc-a"],
        "status_code": ["OK"],
        "start_time": [1000],
        "end_time": [1050]
    })

    traces.write_parquet("test_traces_empty.parquet")

    args = AlgorithmArgs(
        trace_path="test_traces_empty.parquet",
        ground_truth_path="test_gt.parquet",
        output_path="output/",
        dataset_name="test",
        datapack_id="0",
        benchmark="test"
    )

    algo = MyRCA()
    result = algo(args)

    # Should handle gracefully
    assert isinstance(result.ranked_services, list)
```

### Integration Tests

```python
def test_full_pipeline():
    """Test algorithm on real dataset."""

    algo = MyRCA()

    # Test on multiple datapacks
    for datapack_id in range(5):
        args = AlgorithmArgs(
            trace_path=f"data/trainticket-pandora-v1/{datapack_id}/trace.parquet",
            ground_truth_path=f"data/trainticket-pandora-v1/{datapack_id}/ground_truth.parquet",
            output_path="output/",
            dataset_name="trainticket-pandora-v1",
            datapack_id=str(datapack_id),
            benchmark="trainticket"
        )

        result = algo(args)

        # Basic validation
        assert len(result.ranked_services) > 0
        assert all(isinstance(svc, str) and isinstance(score, (int, float))
                   for svc, score in result.ranked_services)
```

## Remote Debugging

### Debug Remote Execution

When debugging remote execution failures:

```bash
# Get logs
./main.py trace task-abc123 --follow

# Download logs for specific datapack
kubectl logs -n aegislab job/task-abc123-dp-0

# Check pod status
kubectl get pods -n aegislab -l task-id=task-abc123

# Describe pod for events
kubectl describe pod -n aegislab task-abc123-dp-0
```

### Test Container Locally

```bash
# Build image
docker build -t my-rca:debug .

# Run locally with test data
docker run --rm \
    -v $(pwd)/data:/data \
    -v $(pwd)/output:/output \
    -e TRACE_PATH=/data/trainticket-pandora-v1/0/trace.parquet \
    -e GROUND_TRUTH_PATH=/data/trainticket-pandora-v1/0/ground_truth.parquet \
    -e OUTPUT_PATH=/output \
    my-rca:debug

# Check output
cat output/output.json
```

## Best Practices

1. **Start simple**: Test with minimal data first
2. **Add logging**: Print statements are your friend
3. **Validate data**: Check input data quality
4. **Profile performance**: Identify bottlenecks early
5. **Write tests**: Unit tests catch regressions
6. **Use version control**: Track changes and revert if needed
7. **Document issues**: Keep notes on problems and solutions

## Tools and Resources

### Polars Debugging

```python
# Enable query plan visualization
traces = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")
print(traces.explain())  # Show query plan

# Check lazy frame
print(traces.describe_plan())  # Detailed plan
```

### Python Profiling

```bash
# Profile with cProfile
python -m cProfile -o profile.stats main.py eval single my-rca trainticket-pandora-v1 0

# Analyze results
python -m pstats profile.stats
# Then: sort cumulative, stats 20
```

### Memory Profiling

```bash
# Install memory_profiler
pip install memory_profiler

# Profile memory usage
python -m memory_profiler main.py eval single my-rca trainticket-pandora-v1 0
```

## See Also

- [Common Errors](../common-errors): Frequently encountered errors
- [Examples](../../examples): Study working algorithms
- [Local Evaluation](../../local-evaluation): Testing guide
