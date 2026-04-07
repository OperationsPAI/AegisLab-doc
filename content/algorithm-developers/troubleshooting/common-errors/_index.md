---
title: Common Errors
weight: 1
---

Frequently encountered errors and their solutions.

## Installation Errors

### ModuleNotFoundError: No module named 'polars'

```
ModuleNotFoundError: No module named 'polars'
```

**Cause**: Dependencies not installed.

**Solution**:
```bash
cd rcabench-platform
uv sync --all-extras
```

### Python version mismatch

```
Error: Python 3.10 or higher required
```

**Cause**: Using Python 3.9 or earlier.

**Solution**: Install Python 3.10+:
```bash
# Using pyenv
pyenv install 3.10
pyenv local 3.10

# Or update system Python
sudo apt install python3.10
```

## Dataset Errors

### Dataset not found

```
Error: Dataset 'trainticket-pandora-v1' not found
```

**Cause**: Dataset not accessible in data directory.

**Solution**:
```bash
# Check if JuiceFS is mounted
ls /mnt/jfs/rcabench-platform-v2/

# If not mounted, mount it (use your JUICEFS_REDIS_URL from .env)
sudo juicefs mount ${JUICEFS_REDIS_URL} /mnt/jfs -d
# Default: redis://10.10.10.119:6379/1

# Create symlink
cd rcabench-platform/data
ln -s /mnt/jfs/rcabench-platform-v2 ./
```

### Datapack not found

```
Error: Datapack 150 not found in dataset trainticket-pandora-v1
```

**Cause**: Datapack ID exceeds dataset size.

**Solution**: Check dataset size:
```bash
./main.py dataset-info trainticket-pandora-v1
# Shows total number of datapacks
```

### Permission denied reading dataset

```
PermissionError: [Errno 13] Permission denied: 'data/trainticket-pandora-v1/0/trace.parquet'
```

**Cause**: Insufficient file permissions.

**Solution**:
```bash
# Fix permissions
chmod -R u+r data/

# Or remount JuiceFS with correct permissions (use your JUICEFS_REDIS_URL from .env)
sudo juicefs mount ${JUICEFS_REDIS_URL} /mnt/jfs -d --uid $(id -u) --gid $(id -g)
# Default: redis://10.10.10.119:6379/1
```

## Algorithm Errors

### Algorithm not registered

```
Error: Algorithm 'my-rca' not found in registry
```

**Cause**: Algorithm not registered in the platform.

**Solution**: Register in `v3/cli/main.py`:
```python
def register_builtin_algorithms():
    registry = AlgorithmRegistry.get_instance()
    registry.register("my-rca", MyRCAAlgorithm)
```

### Empty ranked_services

```
Warning: Algorithm returned empty ranked_services
```

**Cause**: Algorithm filtered out all data or failed to produce results.

**Solution**: Add debug logging:
```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    print(f"Loaded {len(traces)} traces")

    filtered = traces.filter(pl.col("status_code") == "ERROR")
    print(f"Found {len(filtered)} error spans")

    if len(filtered) == 0:
        print("WARNING: No error spans found!")
        # Return default ranking or handle gracefully
        return []
```

### Invalid return type

```
TypeError: Algorithm must return AlgorithmAnswer, got list
```

**Cause**: Returning wrong type from algorithm.

**Solution**: Return list of AlgorithmAnswer:
```python
# Wrong
return ranked_services

# Correct
return [
    AlgorithmAnswer(level="service", name=service_name, rank=rank)
    for rank, service_name in enumerate(ranked_services, start=1)
]
```

## Data Processing Errors

### Memory error

```
MemoryError: Unable to allocate array with shape (1000000, 100)
```

**Cause**: Loading too much data into memory.

**Solution**: Use lazy evaluation:
```python
# Instead of read_parquet
traces = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")

# Build query lazily
result = traces.filter(
    pl.col("status_code") == "ERROR"
).group_by("service_name").agg(
    pl.count().alias("error_count")
).collect()  # Execute only when needed
```

### Polars schema mismatch

```
SchemaError: column 'status_code' not found
```

**Cause**: Expected column missing from data.

**Solution**: Check schema and handle missing columns:
```python
traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

# Check if column exists
if "status_code" not in traces.columns:
    print("WARNING: status_code column missing")
    # Handle gracefully
    return []

# Or use with_columns to add default
traces = traces.with_columns(
    pl.col("status_code").fill_null("UNSET")
)
```

### Invalid timestamp format

```
ValueError: time data '2025-01-18T10:30:00' does not match format
```

**Cause**: Timestamp parsing issue.

**Solution**: Use Polars datetime parsing:
```python
traces = traces.with_columns(
    pl.col("start_time").str.strptime(pl.Datetime, "%Y-%m-%dT%H:%M:%S")
)
```

## Remote Execution Errors

### Authentication failed

```
Error: 401 Unauthorized
```

**Cause**: Invalid or missing API credentials.

**Solution**:
```bash
# Set API token
export AEGISLAB_TOKEN="your-api-token"

# Or configure in code (use your AEGISLAB_API_URL from .env)
config = Configuration(
    host="${AEGISLAB_API_URL}",  # Default: http://10.10.10.220:32080
    api_key={"Authorization": "Bearer your-token"}
)
```

### Docker image not found

```
Error: Failed to pull image harbor.aegislab.io/algorithms/my-rca:latest
```

**Cause**: Image not pushed to Harbor or wrong name.

**Solution**:
```bash
# Verify image exists locally
docker images | grep my-rca

# Push to Harbor
docker tag my-rca:latest harbor.aegislab.io/algorithms/my-rca:latest
docker push harbor.aegislab.io/algorithms/my-rca:latest
```

### Task timeout

```
Error: Task exceeded timeout limit (1800s)
```

**Cause**: Algorithm taking too long to execute.

**Solution**: Optimize algorithm or increase timeout:
```python
request = DtoSubmitExecutionReq(
    algorithm_name="my-rca",
    dataset="trainticket-pandora-v1",
    datapack_start=0,
    datapack_end=99,
    timeout_seconds=3600  # Increase to 1 hour
)
```

### Resource quota exceeded

```
Error: Insufficient resources to schedule task
```

**Cause**: Cluster resources exhausted.

**Solution**: Reduce resource limits or wait:
```python
request = DtoSubmitExecutionReq(
    algorithm_name="my-rca",
    dataset="trainticket-pandora-v1",
    datapack_start=0,
    datapack_end=99,
    cpu_limit="1000m",  # Reduce from 2000m
    memory_limit="2Gi"  # Reduce from 4Gi
)
```

## Containerization Errors

### Docker build failed

```
Error: failed to solve: process "/bin/sh -c pip install -r requirements.txt" did not complete successfully
```

**Cause**: Dependency installation failed.

**Solution**: Check requirements.txt for invalid packages:
```bash
# Test locally first
pip install -r requirements.txt

# Check for typos or version conflicts
cat requirements.txt
```

### Entrypoint not executable

```
Error: exec format error
```

**Cause**: entrypoint.sh not executable or wrong line endings.

**Solution**:
```bash
# Make executable
chmod +x entrypoint.sh

# Fix line endings (if on Windows)
dos2unix entrypoint.sh

# Verify in Dockerfile
RUN chmod +x entrypoint.sh
```

### Missing environment variables

```
Error: TRACE_PATH environment variable not set
```

**Cause**: Algorithm expects environment variables not provided.

**Solution**: Use command-line arguments instead:
```bash
#!/bin/bash
# entrypoint.sh

python -m src.my_rca \
    --trace-path "$TRACE_PATH" \
    --ground-truth-path "$GROUND_TRUTH_PATH" \
    --output-path "$OUTPUT_PATH"
```

## Performance Issues

### Slow execution

**Symptoms**: Algorithm takes >30s per datapack.

**Solutions**:

1. **Use lazy evaluation**:
```python
traces = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")  # Lazy
result = traces.filter(...).collect()  # Execute
```

2. **Reduce data size early**:
```python
# Filter early
traces = traces.filter(
    pl.col("start_time") > injection_time
)
```

3. **Profile your code**:
```python
import cProfile
profiler = cProfile.Profile()
profiler.enable()
result = algo(args)
profiler.disable()
profiler.print_stats()
```

### High memory usage

**Symptoms**: Memory usage >4GB.

**Solutions**:

1. **Stream data**:
```python
# Process in chunks
for batch in pl.read_parquet_batched(args.input_folder / "abnormal_traces.parquet", batch_size=10000):
    process_batch(batch)
```

2. **Use efficient data types**:
```python
traces = traces.with_columns([
    pl.col("service_name").cast(pl.Categorical),
    pl.col("status_code").cast(pl.Categorical)
])
```

## Getting More Help

If your issue isn't listed here:

1. Check [Debugging Tips](../debugging-tips)
2. Review algorithm examples in [Examples](../../examples)
3. Search GitHub issues
4. Ask in community Discord
5. Open a new issue with full error details

## Reporting Bugs

When reporting bugs, include:

1. **Error message**: Full traceback
2. **Environment**: OS, Python version, package versions
3. **Steps to reproduce**: Minimal example
4. **Expected vs actual behavior**
5. **Relevant code**: Algorithm implementation

Example bug report:

```
**Environment:**
- OS: Ubuntu 22.04
- Python: 3.10.12
- Polars: 0.20.0
- rcabench-platform: v1.0.0

**Error:**
```
MemoryError: Unable to allocate array
```

**Steps to reproduce:**
1. Run: ./main.py eval single my-rca trainticket-pandora-v1 0
2. Algorithm loads full dataset into memory

**Expected:** Should use lazy evaluation
**Actual:** Crashes with memory error

**Code:**
```python
traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")  # Loads all data
```
```
