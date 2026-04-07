---
title: Best Practices
weight: 4
---

This guide covers performance optimization, error handling, logging, testing strategies, and common anti-patterns to avoid when developing RCA algorithms.

## Performance Optimization

### Use Lazy Evaluation with Polars

Always prefer `pl.scan_parquet()` over `pl.read_parquet()` for large datasets. Lazy evaluation builds a query plan that optimizes operations before execution.

```python
import polars as pl

# ✅ Good: Lazy evaluation - only reads required columns and rows
traces = (
    pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")
    .filter(pl.col("service_name") == "ts-order-service")
    .select(["trace_id", "span_id", "duration", "attr.status_code"])
    .collect()
)

# ❌ Bad: Eager evaluation - loads entire file into memory
traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
traces = traces.filter(pl.col("service_name") == "ts-order-service")
```

### Batch Operations with `collect_all()`

When loading multiple files, use `collect_all()` to execute queries in parallel:

```python
# ✅ Good: Parallel execution of multiple lazy queries
metrics_lf = pl.scan_parquet(args.input_folder / "abnormal_metrics.parquet")
traces_lf = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")

metrics, traces = pl.collect_all([
    metrics_lf.select(pl.col("service_name")).unique(),
    traces_lf.select(pl.col("service_name")).unique(),
])

# ❌ Bad: Sequential execution
metrics = pl.read_parquet(args.input_folder / "abnormal_metrics.parquet")
traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
```

### Memory Management

For large datasets, process data in chunks or use streaming:

```python
# Process large files in batches
def process_large_dataset(input_folder: Path) -> list[AlgorithmAnswer]:
    # Use lazy evaluation to filter early
    lf = pl.scan_parquet(input_folder / "abnormal_traces.parquet")

    # Get unique services first (small result)
    services = lf.select("service_name").unique().collect()

    results = []
    for service in services["service_name"]:
        # Process one service at a time
        service_data = (
            lf.filter(pl.col("service_name") == service)
            .collect()
        )
        score = compute_score(service_data)
        results.append((service, score))

    return rank_results(results)
```

### Cache Expensive Computations

For computations that may be repeated (like building dependency graphs), use caching:

```python
import pickle
from pathlib import Path

def get_or_build_graph(args: AlgorithmArgs, cache_file: str = "sdg.pkl") -> Graph:
    cache_path = args.output_folder / cache_file

    if cache_path.exists():
        with open(cache_path, "rb") as f:
            return pickle.load(f)

    # Build graph (expensive operation)
    graph = build_service_dependency_graph(args.input_folder)

    # Cache for future use
    with open(cache_path, "wb") as f:
        pickle.dump(graph, f)

    return graph
```

### Specify CPU Requirements

Implement `needs_cpu_count()` accurately to enable efficient parallel scheduling:

```python
class MyAlgorithm(Algorithm):
    def needs_cpu_count(self) -> int | None:
        # Return specific count for CPU-bound algorithms
        return 4

        # Or return None to use all available cores
        # return None

        # Single-threaded algorithms should return 1
        # return 1
```

## Error Handling

### Graceful Degradation

Handle missing or malformed data without crashing:

```python
from pathlib import Path
import polars as pl

def safe_read_parquet(path: Path) -> pl.DataFrame | None:
    """Read parquet file, returning None if it doesn't exist or is invalid."""
    if not path.exists():
        logger.warning(f"File not found: {path}")
        return None

    try:
        df = pl.read_parquet(path)
        if df.is_empty():
            logger.warning(f"Empty dataframe: {path}")
            return None
        return df
    except Exception as e:
        logger.error(f"Failed to read {path}: {e}")
        return None
```

### Validate Input Data

Check data integrity before processing:

```python
def validate_input(args: AlgorithmArgs) -> bool:
    """Validate required input files exist and have expected schema."""
    required_files = [
        "abnormal_traces.parquet",
        "injection.json",
    ]

    for filename in required_files:
        path = args.input_folder / filename
        if not path.exists():
            logger.error(f"Missing required file: {filename}")
            return False

    # Validate schema
    traces = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")
    required_columns = {"trace_id", "span_id", "service_name", "duration"}
    actual_columns = set(traces.collect_schema().names())

    missing = required_columns - actual_columns
    if missing:
        logger.error(f"Missing required columns: {missing}")
        return False

    return True
```

### Return Empty Results on Failure

Always return a valid (possibly empty) result rather than raising exceptions:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    if not validate_input(args):
        logger.warning("Invalid input, returning empty results")
        return []

    try:
        results = self.analyze(args)
        return results
    except Exception as e:
        logger.error(f"Analysis failed: {e}")
        # Return empty list instead of propagating exception
        return []
```

### Handle Dataset Format Differences

Different datasets have different file structures:

```python
def load_traces(args: AlgorithmArgs) -> pl.LazyFrame:
    """Load traces, handling different dataset formats."""
    if args.dataset.startswith("rcabench"):
        path = args.input_folder / "abnormal_traces.parquet"
    elif args.dataset.startswith("rcaeval"):
        path = args.input_folder / "traces.parquet"
    else:
        raise NotImplementedError(f"Unknown dataset format: {args.dataset}")

    if not path.exists():
        raise FileNotFoundError(f"Trace file not found: {path}")

    return pl.scan_parquet(path)
```

## Logging and Debugging

### Use the Built-in Logger

Import and use the platform's logger for consistent output:

```python
from rcabench_platform.v3.sdk.logging import logger, timeit

class MyAlgorithm(Algorithm):
    @timeit()  # Auto-logs entry/exit with duration
    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        logger.info(f"Processing dataset: {args.dataset}/{args.datapack}")

        # Debug-level for verbose output
        logger.debug(f"Input folder contents: {list(args.input_folder.iterdir())}")

        results = self.analyze(args)
        logger.info(f"Found {len(results)} root cause candidates")

        return results
```

### Use `@timeit()` Decorator

Track execution time of key functions:

```python
from rcabench_platform.v3.sdk.logging import timeit

@timeit(log_level="INFO")  # Log at INFO level instead of DEBUG
def build_dependency_graph(traces: pl.DataFrame) -> Graph:
    # ... expensive operation
    pass

@timeit(log_args={"service_name"})  # Only log specific arguments
def analyze_service(service_name: str, traces: pl.DataFrame) -> float:
    # ... analysis
    pass
```

### Debug Mode for Verbose Output

Check the DEBUG environment variable for conditional logging:

```python
import os

def debug() -> bool:
    return os.environ.get("DEBUG", "").lower() in ("true", "1", "yes")

class MyAlgorithm(Algorithm):
    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        if debug():
            logger.debug(f"Full args: {args}")

        results = self.analyze(args)

        if debug():
            # Save intermediate results for debugging
            self.save_debug_output(args, results)

        return results

    def save_debug_output(self, args: AlgorithmArgs, results: list[AlgorithmAnswer]):
        """Save intermediate results for debugging."""
        import json

        debug_path = args.output_folder / "debug_results.json"
        with open(debug_path, "w") as f:
            json.dump([
                {"level": r.level, "name": r.name, "rank": r.rank}
                for r in results
            ], f, indent=2)

        logger.debug(f"Debug output saved to: {debug_path}")
```

### Progress Tracking

Use tqdm for progress bars in long-running operations:

```python
from tqdm.auto import tqdm

def analyze_services(services: list[str], traces: pl.DataFrame) -> dict[str, float]:
    scores = {}

    for service in tqdm(services, desc="Analyzing services"):
        service_traces = traces.filter(pl.col("service_name") == service)
        scores[service] = compute_score(service_traces)

    return scores
```

## Testing Strategies

### Unit Test Individual Components

Test helper functions in isolation:

```python
# test_my_algorithm.py
import polars as pl
import pytest
from my_algorithm import compute_error_rate, rank_services

def test_compute_error_rate():
    traces = pl.DataFrame({
        "service_name": ["svc-a", "svc-a", "svc-a", "svc-b"],
        "attr.status_code": ["Ok", "Error", "Ok", "Ok"],
    })

    rate = compute_error_rate(traces, "svc-a")
    assert abs(rate - 0.333) < 0.01  # 1/3 errors

def test_rank_services_empty():
    scores = {}
    result = rank_services(scores)
    assert result == []

def test_rank_services_sorted():
    scores = {"svc-a": 0.5, "svc-b": 0.8, "svc-c": 0.3}
    result = rank_services(scores)

    assert result[0].name == "svc-b"  # Highest score first
    assert result[0].rank == 1
    assert result[1].name == "svc-a"
    assert result[2].name == "svc-c"
```

### Integration Test with Real Data

Test the full algorithm with a sample datapack:

```python
# test_integration.py
from pathlib import Path
from my_algorithm import MyAlgorithm
from rcabench_platform.v3.sdk.algorithms.spec import AlgorithmArgs

def test_algorithm_on_sample_data():
    args = AlgorithmArgs(
        dataset="rcabench_filtered",
        datapack="ts0-ts-auth-service-stress-jv8m9r",
        input_folder=Path("data/rcabench-platform-v2/rcabench_filtered/ts0-ts-auth-service-stress-jv8m9r"),
        output_folder=Path("output/test"),
    )

    algo = MyAlgorithm()
    results = algo(args)

    # Basic sanity checks
    assert len(results) > 0
    assert all(r.rank > 0 for r in results)
    assert len(set(r.rank for r in results)) == len(results)  # Unique ranks
```

### Test with CLI

Use the platform CLI to test your algorithm:

```bash
# Run on a single datapack
./main.py eval single my-algorithm rcabench_filtered ts0-ts-auth-service-stress-jv8m9r

# Enable debug output
DEBUG=true ./main.py eval single my-algorithm rcabench_filtered ts0-ts-auth-service-stress-jv8m9r

# Clear cached output and re-run
./main.py eval single my-algorithm rcabench_filtered ts0-ts-auth-service-stress-jv8m9r --clear
```

### Benchmark Performance

Compare execution time and memory usage:

```python
import time
import tracemalloc

def benchmark_algorithm(algo: Algorithm, args: AlgorithmArgs):
    tracemalloc.start()
    start_time = time.perf_counter()

    results = algo(args)

    end_time = time.perf_counter()
    current, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()

    print(f"Duration: {end_time - start_time:.2f}s")
    print(f"Peak memory: {peak / 1024 / 1024:.1f} MB")
    print(f"Results: {len(results)} candidates")
```

## Anti-Patterns to Avoid

### Don't Use Eager Loading for Large Files

```python
# ❌ Bad: Loads entire file into memory immediately
df = pl.read_parquet(path)
filtered = df.filter(pl.col("service_name") == target)

# ✅ Good: Lazy evaluation with predicate pushdown
lf = pl.scan_parquet(path)
filtered = lf.filter(pl.col("service_name") == target).collect()
```

### Don't Ignore Dataset Format Differences

```python
# ❌ Bad: Assumes single file format
traces = pl.read_parquet(input_folder / "traces.parquet")

# ✅ Good: Handle different datasets
if dataset.startswith("rcabench"):
    traces = pl.read_parquet(input_folder / "abnormal_traces.parquet")
elif dataset.startswith("rcaeval"):
    traces = pl.read_parquet(input_folder / "traces.parquet")
```

### Don't Hardcode Column Names Without Verification

```python
# ❌ Bad: Assumes column exists
error_rate = df.filter(pl.col("status_code") == "Error")

# ✅ Good: Use correct column name and verify
schema = df.collect_schema()
status_col = "attr.status_code" if "attr.status_code" in schema.names() else "status_code"
error_rate = df.filter(pl.col(status_col) == "Error")
```

### Don't Use Deprecated Polars Functions

```python
# ❌ Bad: pl.count() is deprecated
counts = df.group_by("service").agg(pl.count())

# ✅ Good: Use pl.len()
counts = df.group_by("service").agg(pl.len())
```

### Don't Silently Swallow Exceptions

```python
# ❌ Bad: Hides errors completely
try:
    result = process(data)
except:
    pass

# ✅ Good: Log the error and handle gracefully
try:
    result = process(data)
except Exception as e:
    logger.error(f"Processing failed: {e}")
    result = default_value
```

### Don't Create Unnecessary Intermediate Files

```python
# ❌ Bad: Creates temp files for intermediate results
df.write_parquet("temp1.parquet")
df2 = pl.read_parquet("temp1.parquet")
df2.write_parquet("temp2.parquet")

# ✅ Good: Chain operations in memory
result = (
    df
    .filter(...)
    .group_by(...)
    .agg(...)
)
```

### Don't Forget to Register Your Algorithm

```python
# ❌ Bad: Algorithm class exists but not registered
# (Algorithm won't be available via CLI)

# ✅ Good: Register in cli/main.py
def register_builtin_algorithms():
    getters = {
        "my-algorithm": MyAlgorithm,
        # ... other algorithms
    }
    registry = global_algorithm_registry()
    for name, getter in getters.items():
        registry[name] = getter
```

## Summary Checklist

Before submitting your algorithm:

- [ ] Uses lazy evaluation (`pl.scan_parquet()`) for large files
- [ ] Implements `needs_cpu_count()` accurately
- [ ] Handles missing files and empty data gracefully
- [ ] Supports both rcabench and rcaeval dataset formats (if applicable)
- [ ] Uses correct column names (`attr.status_code`, not `status_code`)
- [ ] Logs progress and errors appropriately
- [ ] Has unit tests for key functions
- [ ] Tested with real data via CLI
- [ ] Registered in the algorithm registry
- [ ] No deprecated Polars functions (`pl.len()` not `pl.count()`)

## Next Steps

- [Algorithm Interface](../algorithm-interface): Detailed API reference
- [Data Formats](../data-formats): Understanding input parquet schemas
- [Containerization](../containerization): Packaging for remote execution
