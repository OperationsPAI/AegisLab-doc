---
title: Algorithm Interface
weight: 1
---

Complete reference for implementing RCA algorithms using the standardized interface.

## Base Class

All RCA algorithms must inherit from the `Algorithm` base class:

```python
from rcabench_platform.v2.algorithms import Algorithm, AlgorithmArgs, AlgorithmAnswer

class MyRCAAlgorithm(Algorithm):
    def needs_cpu_count(self) -> int | None:
        """
        Returns the number of CPU cores needed by the algorithm.
        Return None to use all available cores, or a positive integer for specific core count.
        """
        return 1  # Use 1 core, or None for all cores

    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        """
        Main entry point for the algorithm.

        Args:
            args: Input data paths and metadata

        Returns:
            List of AlgorithmAnswer with ranked root cause predictions
        """
        # Your algorithm implementation
        pass
```  
**Note**: Algorithms should not rely on persistent in-memory state across datapacks. Each evaluation run should treat the input datapack independently.

## AlgorithmArgs

Input data structure passed to your algorithm:

```python
from pathlib import Path

@dataclass(kw_only=True, frozen=True, slots=True)
class AlgorithmArgs:
    dataset: str           # Dataset name (e.g., "trainticket-pandora-v1")
    datapack: str          # Datapack identifier (e.g., "0", "1", "2")
    input_folder: Path     # Path to folder containing parquet files
    output_folder: Path    # Path to folder for writing results
```  
The `input_folder` corresponds to a single datapack directory containing telemetry parquet files.  
The `output_folder` can be used to store intermediate results, debug outputs, or additional artifacts produced by the algorithm. Files written here will be preserved alongside the evaluation outputs.

### Input Folder Structure

The `input_folder` contains standardized parquet files:

```
input_folder/
├── abnormal_traces.parquet           # Distributed trace data during fault (required)
├── normal_traces.parquet             # Baseline trace data before fault (required)
├── abnormal_logs.parquet             # Application logs during fault (optional)
├── normal_logs.parquet               # Baseline logs before fault (optional)
├── abnormal_metrics.parquet          # Time-series metrics during fault (optional)
├── normal_metrics.parquet            # Baseline metrics before fault (optional)
├── abnormal_metrics_sum.parquet      # Sum metrics during fault (optional)
├── abnormal_metrics_histogram.parquet # Histogram metrics during fault (optional)
├── metrics_sli.parquet               # SLI metrics (optional)
├── injection.json                    # Ground truth fault injection info
└── conclusion.parquet                # Span-level performance comparison
```

**Note**: The dataset provides both abnormal (during fault) and normal (baseline) data for comparison. Most algorithms focus on the abnormal data files.

### Example Usage

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    # Read trace data from input folder
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

    # Read ground truth for fault information
    with open(args.input_folder / "injection.json") as f:
        injection = json.load(f)

    # Optional: read metrics if available
    metrics_path = args.input_folder / "abnormal_metrics.parquet"
    if metrics_path.exists():
        metrics = pl.read_parquet(metrics_path)
```
The `injection.json` file contains ground truth information about the injected fault. It is provided for reference and evaluation purposes. RCA algorithms should not use this file when generating predictions.

## AlgorithmAnswer

Output structure your algorithm must return:

```python
@dataclass(kw_only=True, frozen=True, slots=True)
class AlgorithmAnswer:
    level: str    # Level of the root cause (e.g., "service", "pod", "container")
    name: str     # Name of the suspected root cause
    rank: int     # Rank of this answer (1，2，3..., 1 = most likely root cause)
```  
Your algorithm returns a **list** of `AlgorithmAnswer` objects, ranked by likelihood.  
For service-level RCA, this typically means returning all services sorted by suspected root cause likelihood.

### Example Usage

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    # Your algorithm logic identifies root causes

    return [
        AlgorithmAnswer(level="service", name="ts-order-service", rank=1),
        AlgorithmAnswer(level="service", name="ts-payment-service", rank=2),
        AlgorithmAnswer(level="service", name="ts-user-service", rank=3),
    ]
```

## Complete Example

Here's a complete minimal algorithm:

```python
from rcabench_platform.v2.algorithms import Algorithm, AlgorithmArgs, AlgorithmAnswer
#This example uses polars, but you can use other libraries like panda
import polars as pl
import json
from datetime import datetime

class ErrorCountRCA(Algorithm):
    """Simple RCA algorithm that ranks services by error count."""

    def needs_cpu_count(self) -> int | None:
        return 1  # Use single core

    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        try:
            # Read trace data from input folder
            traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

            # Read injection info for fault time window
            with open(args.input_folder / "injection.json") as f:
                injection = json.load(f)

            # Parse fault time window
            fault_start = datetime.fromisoformat(injection["start_time"].replace("Z", "+00:00"))
            fault_end = datetime.fromisoformat(injection["end_time"].replace("Z", "+00:00"))

            # Filter to fault window
            traces = traces.filter(
                (pl.col("time") >= fault_start) &
                (pl.col("time") <= fault_end)
            )

            # Count errors by service
            error_counts = (
                traces
                .filter(pl.col("attr.status_code") == "Error")
                .group_by("service_name")
                .agg(pl.len().alias("error_count"))
                .sort("error_count", descending=True)
            )

            # Build ranked list of answers
            results = []
            for rank, row in enumerate(error_counts.iter_rows(named=True), start=1):
                results.append(AlgorithmAnswer(
                    level="service",
                    name=row["service_name"],
                    rank=rank
                ))

            return results

        except Exception as e:
            # Return empty result on error
            print(f"Error in algorithm: {e}")
            return []
```
The `service_name` field represents the logical service that emitted the span. It is typically derived from the tracing system (e.g., the OpenTelemetry `service.name` attribute).  
`time`, `attr.status_code` and `service_name` are possible columns in `abnormal_traces.parquet`.

## Algorithm Registration

To make your algorithm available via the CLI, register it in the global registry.

### Step 1: Create Your Algorithm File

Create your algorithm in `rcabench_platform/v2/algorithms/`:

```python
# rcabench_platform/v2/algorithms/my_algorithm.py
from .spec import Algorithm, AlgorithmArgs, AlgorithmAnswer
import polars as pl

class MyRCAAlgorithm(Algorithm):
    def needs_cpu_count(self) -> int | None:
        return 1

    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        # Your algorithm implementation
        traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
        # ... your logic ...
        return results
```

### Step 2: Register in main.py

Add your algorithm to the registration function in `rcabench_platform/v2/cli/main.py`:

```python
def register_builtin_algorithms():
    from ..algorithms.random_ import Random
    from ..algorithms.my_algorithm import MyRCAAlgorithm  # Add your import

    getters = {
        "random": Random,
        "my-rca": MyRCAAlgorithm,  # Add your algorithm
    }

    registry = global_algorithm_registry()
    for name, getter in getters.items():
        registry[name] = getter
```

### Step 3: Test Your Algorithm

Run your algorithm via CLI:

```bash
./main.py eval single my-rca rcabench_filtered <datapack-name>
```
Use --clear to re-run if results already exist.

## Best Practices

### Error Handling

Always handle errors gracefully:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    try:
        # Your algorithm logic
        pass
    except FileNotFoundError as e:
        print(f"Data file not found: {e}")
        return []
    except Exception as e:
        print(f"Unexpected error: {e}")
        return []
```

### Data Validation

Validate input data before processing:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

    # Check if data is empty
    if traces.is_empty():
        return []

    # Check required columns exist
    required_cols = ["service_name", "attr.status_code", "time"]
    if not all(col in traces.columns for col in required_cols):
        print(f"Missing required columns")
        return []
```

### Memory Efficiency

Use lazy evaluation for large datasets:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    # Use scan instead of read for lazy evaluation
    traces = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")

    # Build query without executing
    result = (
        traces
        .filter(pl.col("attr.status_code") == "Error")
        .group_by("service_name")
        .agg(pl.count())
        .collect()  # Execute only when needed
    )
```

### Logging

Use print statements for logging (captured by platform):

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    print(f"Processing dataset: {args.dataset}")
    print(f"Datapack ID: {args.datapack}")

    # Your algorithm logic

    print(f"Found {len(results)} suspected root causes")
    return results
```

## Advanced Features

### Using Multiple Data Sources

Combine traces, metrics, and logs:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    # Read all available data from input folder
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

    # Check for optional data sources
    metrics_path = args.input_folder / "abnormal_metrics.parquet"
    if metrics_path.exists():
        metrics = pl.read_parquet(metrics_path)

    log_path = args.input_folder / "abnormal_logs.parquet"
    if log_path.exists():
        logs = pl.read_parquet(log_path)

    # Combine insights from all sources
    trace_suspects = analyze_traces(traces)

    if metrics_path.exists():
        metric_suspects = analyze_metrics(metrics)
        # Merge results

    if log_path.exists():
        log_suspects = analyze_logs(logs)
        # Merge results
```

### Configurable Parameters

Support configuration via constructor parameters:

```python
class ConfigurableRCA(Algorithm):
    def __init__(self, threshold: float = 0.5, window_size: int = 60):
        self.threshold = threshold
        self.window_size = window_size

    def needs_cpu_count(self) -> int | None:
        return 1

    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        # Use self.threshold and self.window_size
        pass
```

### Caching Intermediate Results

Cache expensive computations:

```python
from functools import lru_cache

class CachedRCA(Algorithm):
    def needs_cpu_count(self) -> int | None:
        return 1

    @lru_cache(maxsize=128)
    def compute_service_graph(self, trace_data_hash: str):
        # Expensive computation
        pass

    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

        # Create hash of trace data
        data_hash = hash(traces.to_pandas().to_json())

        # Use cached result if available
        graph = self.compute_service_graph(data_hash)
```

## Testing Your Algorithm

Test locally before submission:

```python
# test_my_algorithm.py
from pathlib import Path
from rcabench_platform.v2.algorithms import AlgorithmArgs
from my_algorithm import MyRCAAlgorithm

def test_algorithm():
    args = AlgorithmArgs(
        dataset="rcabench_filtered",
        datapack="ts0-ts-auth-service-stress-jv8m9r",
        input_folder=Path("data/rcabench-platform-v2/data/rcabench_filtered/ts0-ts-auth-service-stress-jv8m9r"),
        output_folder=Path("output/test")
    )

    algo = MyRCAAlgorithm()
    results = algo(args)

    assert len(results) > 0
    print(f"Test passed! Predicted: {[r.name for r in results[:5]]}")

if __name__ == "__main__":
    test_algorithm()
```

Run via CLI:

```bash
# Test on a single datapack
./main.py eval single my-rca rcabench_filtered ts0-ts-auth-service-stress-jv8m9r

# Test on entire dataset
./main.py eval batch my-rca rcabench_filtered
```

## Next Steps

- [Data Formats](data-formats): Understanding input data schemas
- [Containerization](containerization): Package your algorithm for deployment
- [Local Evaluation](../local-evaluation): Test on real datasets
