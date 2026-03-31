---
title: Local Evaluation
weight: 3
---

Test and evaluate your RCA algorithms locally before submitting for remote execution.

## Overview

Local evaluation allows you to:
- Test algorithms on real datasets
- Iterate quickly during development
- Debug issues before remote submission
- Compare multiple algorithms
- Validate performance metrics

## Prerequisites

- rcabench-platform installed (`uv sync --all-extras`)
- Access to datasets (via JuiceFS or local files)
- Your algorithm registered in the platform

## Dataset Setup

### Using JuiceFS (Recommended)

Mount the shared dataset storage:

```bash
# Mount JuiceFS (use your JUICEFS_REDIS_URL from .env)
sudo juicefs mount ${JUICEFS_REDIS_URL} /mnt/jfs -d
# Default: redis://10.10.10.119:6379/1

# Create symlink in your project
cd rcabench-platform
mkdir -p data
cd data
ln -s /mnt/jfs/rcabench-platform-v2 ./
```

### Using Local Datasets

Download or copy datasets to your local directory:

```bash
cd rcabench-platform/data
mkdir -p trainticket-pandora-v1
# Copy parquet files to this directory
```

## Single Evaluation

Evaluate one algorithm on one datapack:

```bash
./main.py eval single <algorithm> <dataset> <datapack>
```

### Example

```bash
# Evaluate simplerca on trainticket-pandora-v1 dataset, datapack 0
./main.py eval single simplerca trainticket-pandora-v1 0
```

### Output

```
Loading dataset: trainticket-pandora-v1, datapack: 0
Running algorithm: simplerca
Processing traces...
Algorithm completed in 1.23s

Results:
  MRR: 0.850
  Avg@1: 0.750
  Avg@3: 2.100
  Avg@5: 3.200
  Top-1 Accuracy: 0.750
  Top-3 Accuracy: 0.900
  Top-5 Accuracy: 0.950

Predicted root causes:
  1. ts-order-service (score: 0.95)
  2. ts-payment-service (score: 0.72)
  3. ts-user-service (score: 0.43)

Ground truth: ts-order-service
```

## Batch Evaluation

Evaluate one algorithm on multiple datapacks:

```bash
./main.py eval batch <algorithm> <dataset> <start_id> <end_id>
```

### Example

```bash
# Evaluate on datapacks 0-9
./main.py eval batch simplerca trainticket-pandora-v1 0 9
```

### Output

```
Evaluating simplerca on 10 datapacks...

Progress: [##########] 10/10 (100%)

Aggregate Results:
  Mean MRR: 0.823
  Mean Avg@1: 0.712
  Mean Avg@3: 2.034
  Mean Top-1 Accuracy: 0.712
  Mean Top-3 Accuracy: 0.887

Per-datapack results saved to: output/batch_results.json
```

## Understanding Metrics

### Mean Reciprocal Rank (MRR)

Average of 1/rank for the first correct answer:

- **1.0**: Perfect (correct answer always ranked first)
- **0.5**: Correct answer typically ranked second
- **0.33**: Correct answer typically ranked third

```python
# Example calculation
ranks = [1, 2, 1, 3, 1]  # Ranks of first correct answer
mrr = sum(1/r for r in ranks) / len(ranks)
# mrr = (1/1 + 1/2 + 1/1 + 1/3 + 1/1) / 5 = 0.767
```

### Average@k (Avg@k)

Average number of correct answers in top-k predictions:

- **Avg@1**: How many correct answers in top-1 (max: 1.0)
- **Avg@3**: How many correct answers in top-3 (max: 3.0)
- **Avg@5**: How many correct answers in top-5 (max: 5.0)

### Top-k Accuracy

Percentage of cases where at least one correct answer appears in top-k:

- **Top-1**: Percentage where correct answer is ranked first
- **Top-3**: Percentage where correct answer is in top-3
- **Top-5**: Percentage where correct answer is in top-5

## Comparing Algorithms

Compare multiple algorithms on the same dataset:

```bash
# Evaluate multiple algorithms
./main.py eval single simplerca trainticket-pandora-v1 0
./main.py eval single my-rca trainticket-pandora-v1 0

# Or use batch evaluation
./main.py eval batch simplerca trainticket-pandora-v1 0 9
./main.py eval batch my-rca trainticket-pandora-v1 0 9
```

Create a comparison script:

```python
# compare_algorithms.py
import json
import pandas as pd

algorithms = ["simplerca", "my-rca", "another-rca"]
results = []

for algo in algorithms:
    with open(f"output/{algo}_results.json") as f:
        data = json.load(f)
        results.append({
            "Algorithm": algo,
            "MRR": data["mean_mrr"],
            "Top-1": data["top1_accuracy"],
            "Top-3": data["top3_accuracy"]
        })

df = pd.DataFrame(results)
print(df.to_string(index=False))
```

## Debugging Your Algorithm

### Enable Verbose Logging

Add print statements in your algorithm:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    print(f"Processing {args.dataset}/{args.datapack}")

    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    print(f"Loaded {len(traces)} traces")

    # Your algorithm logic
    print(f"Found {len(error_spans)} error spans")

    return [
        AlgorithmAnswer(level="service", name=service_name, rank=rank)
        for rank, service_name in enumerate(ranked, start=1)
    ]
```

### Inspect Input Data

Read and examine the input data:

```python
# inspect_data.py
import polars as pl

# Read trace data
traces = pl.read_parquet("data/trainticket-pandora-v1/0/trace.parquet")
print(traces.head())
print(traces.describe())

# Check for errors
errors = traces.filter(pl.col("status_code") == "ERROR")
print(f"Error count: {len(errors)}")
print(errors.select(["service_name", "operation_name"]))

# Read ground truth
gt = pl.read_parquet("data/trainticket-pandora-v1/0/ground_truth.parquet")
print(gt)
```

### Test with Minimal Data

Create a minimal test case:

```python
# test_minimal.py
from pathlib import Path
from rcabench_platform.v2.algorithms import AlgorithmArgs
from my_algorithm import MyRCAAlgorithm

# Create minimal test data
import polars as pl

traces = pl.DataFrame({
    "trace_id": ["t1", "t1", "t2"],
    "span_id": ["s1", "s2", "s3"],
    "service_name": ["svc-a", "svc-b", "svc-a"],
    "status_code": ["OK", "ERROR", "OK"],
    "start_time": [1000, 1100, 2000],
    "end_time": [1050, 1150, 2050]
})

# Create test directory structure
test_input = Path("test_data")
test_input.mkdir(exist_ok=True)
traces.write_parquet(test_input / "abnormal_traces.parquet")

# Test algorithm
args = AlgorithmArgs(
    dataset="test",
    datapack="0",
    input_folder=test_input,
    output_folder=Path("output")
)

algo = MyRCAAlgorithm()
result = algo(args)
print(result)
```

## Performance Profiling

Profile your algorithm to identify bottlenecks:

```python
# profile_algorithm.py
import cProfile
import pstats
from pathlib import Path
from rcabench_platform.v2.algorithms import AlgorithmArgs
from my_algorithm import MyRCAAlgorithm

args = AlgorithmArgs(
    dataset="trainticket-pandora-v1",
    datapack="0",
    input_folder=Path("data/rcabench-platform-v2/data/trainticket-pandora-v1/0"),
    output_folder=Path("output")
)

algo = MyRCAAlgorithm()

# Profile execution
profiler = cProfile.Profile()
profiler.enable()

result = algo(args)

profiler.disable()

# Print stats
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)  # Top 20 functions
```

## Memory Usage

Monitor memory usage for large datasets:

```python
# memory_test.py
import tracemalloc
from pathlib import Path
from rcabench_platform.v2.algorithms import AlgorithmArgs
from my_algorithm import MyRCAAlgorithm

# Start memory tracking
tracemalloc.start()

args = AlgorithmArgs(
    dataset="trainticket-pandora-v1",
    datapack="0",
    input_folder=Path("data/rcabench-platform-v2/data/trainticket-pandora-v1/0"),
    output_folder=Path("output")
)

algo = MyRCAAlgorithm()
result = algo(args)

# Get memory stats
current, peak = tracemalloc.get_traced_memory()
print(f"Current memory: {current / 1024 / 1024:.2f} MB")
print(f"Peak memory: {peak / 1024 / 1024:.2f} MB")

tracemalloc.stop()
```

## Common Issues

### Dataset Not Found

```
Error: Dataset 'trainticket-pandora-v1' not found
```

**Solution**: Verify dataset path and JuiceFS mount:

```bash
ls data/trainticket-pandora-v1/
# Should show numbered directories (0, 1, 2, ...)
```

### Algorithm Not Registered

```
Error: Algorithm 'my-rca' not found in registry
```

**Solution**: Register your algorithm in `v2/cli/main.py`:

```python
def register_builtin_algorithms():
    registry = AlgorithmRegistry.get_instance()
    registry.register("my-rca", MyRCAAlgorithm)
```

### Empty Results

```
Results: ranked_services=[]
```

**Solution**: Check if your algorithm is filtering out all data:

```python
# Add debug prints
print(f"Total traces: {len(traces)}")
print(f"Filtered traces: {len(filtered_traces)}")
```

### Memory Errors

```
MemoryError: Unable to allocate array
```

**Solution**: Use lazy evaluation:

```python
# Instead of read_parquet
traces = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")

# Build query
result = traces.filter(...).group_by(...).collect()
```

## Best Practices

1. **Start small**: Test on single datapacks before batch evaluation
2. **Use lazy evaluation**: Scan instead of read for large datasets
3. **Profile early**: Identify performance issues before remote submission
4. **Validate output**: Ensure ranked_services is not empty
5. **Handle errors**: Gracefully handle missing or malformed data
6. **Log progress**: Add print statements for debugging

## Next Steps

- [Remote Evaluation](../remote-evaluation): Submit for remote execution
- [Containerization](../development-guide/containerization): Package your algorithm
- [Examples](../examples): Study example algorithms
