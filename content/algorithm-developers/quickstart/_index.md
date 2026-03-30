---
title: Quickstart
weight: 1
---

This quickstart demonstrates how to run a simple RCA algorithm using rcabench-platform in under 15 minutes.  

You will:  

- install the platform
- verify the environment
- run a baseline algorithm on a datapack

## Prerequisites

- Python 3.10 or higher
- Git
- uv (Python package manager)
- Basic understanding of distributed tracing concepts

## Pre-Flight Checks

Before starting, verify your environment is properly configured:

### 1. Check Python Version

```bash
python3 --version
# Expected: Python 3.10.x or higher
```

### 2. Check uv Installation

```bash
uv --version
# Expected: uv 0.x.x

# If not installed:
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 3. Check Git Installation

```bash
git --version
# Expected: git version 2.x.x
```

### Verification Script

Run this script to verify all prerequisites:

```bash
#!/bin/bash
# Save as check_prereqs.sh and run with: bash check_prereqs.sh

echo "Checking prerequisites..."

# Check Python version
python_version=$(python3 --version 2>&1 | cut -d' ' -f2)
python_major=$(echo $python_version | cut -d'.' -f1)
python_minor=$(echo $python_version | cut -d'.' -f2)
if [ "$python_major" -ge 3 ] && [ "$python_minor" -ge 10 ]; then
    echo "✓ Python $python_version"
else
    echo "✗ Python 3.10+ required (found $python_version)"
    exit 1
fi

# Check uv
if command -v uv &> /dev/null; then
    echo "✓ uv $(uv --version | head -1)"
else
    echo "✗ uv not found. Install with: curl -LsSf https://astral.sh/uv/install.sh | sh"
    exit 1
fi

# Check git
if command -v git &> /dev/null; then
    echo "✓ git $(git --version | cut -d' ' -f3)"
else
    echo "✗ git not found"
    exit 1
fi

echo "All prerequisites satisfied!"
```

## Step 1: Install rcabench-platform

Clone and install the rcabench-platform:

```bash
git clone https://github.com/OperationsPAI/rcabench-platform.git
cd rcabench-platform
uv sync --all-extras
```
`uv sync --all-extras` installs all dependencies and automatically creates a virtual environment in .venv.

If you use an IDE, configure the interpreter to: `.venv/bin/python`.

This installs the platform with all dependencies including algorithm development tools.

Example commands below assume you are running them inside the ```rcabench-platform``` project root.

## Step 2: Verify Installation

Test the installation by running the self-test:

```bash
./main.py self test
```

**Expected output:**

```
[INFO] Testing rcabench-platform environment...
[ERROR] ClickHouse ping failed: ... (expected if ClickHouse not running)
[ERROR] RCABench ping failed: ... (expected if credentials not configured)
[INFO] config.data is found: data/rcabench-platform-v2
[INFO] Hello from rcabench-platform!
```

**Note**: ClickHouse and RCABench API errors are expected if you're only doing local evaluation. The important check is that the data directory is found.

### Check Dataset Availability

Verify you have access to datasets:

```bash
./main.py eval show-datasets
```

**Expected output:**

```
[INFO] Available datasets (37):
[INFO]     rcabench_filtered        (1124 datapacks)
[INFO]     rcaeval_re2_tt           (  90 datapacks)
[INFO]     rcaeval_re2_ob           (  90 datapacks)
[INFO]     aiops21                  ( 159 datapacks)
[INFO]     eadro_tt                 (  81 datapacks)
...
```

The number in parentheses indicates how many datasets were discovered in the data directory.

Running the command: `./main.py eval show-datasets` to lists all datasets found in this directory.  
You can also inspect them manually: 'ls data/rcabench-platform-v2/data/'  

If you see "Available datasets (0)", your data directory is not properly configured. See troubleshooting below.

## Step 3: Download Sample Dataset

For this quickstart, we'll use a small sample dataset. Create a data directory and download the sample:

```bash
mkdir -p data/sample
# Sample dataset will be provided - for now, use existing datasets
# You may also refer to the Dataset Creator to generate test datasets.
# If you have JuiceFS access:
# sudo juicefs mount ${JUICEFS_REDIS_URL} /mnt/jfs -d
# Default: redis://10.10.10.119:6379/1 (see .env.example)
# ln -s /mnt/jfs/rcabench-platform-v2 data/
```

## Step 4: Run Your First Algorithm

Run the built-in `random` algorithm on a sample dataset:

```bash
./main.py eval single random rcabench_filtered ts0-ts-auth-service-stress-jv8m9r
```

Datapack IDs represent specific fault injection scenarios, for example:  
`ts0-ts-auth-service-stress-jv8m9r` means a scenario where ts-auth-service experiences stress.

You can list available datapacks using:  
'ls data/rcabench-platform-v2/data/rcabench_filtered/'

This command:
- Evaluates the `random` algorithm (randomly ranks all services)
- Uses the `rcabench_filtered` dataset
- Processes the specified datapack (fault injection scenario)

**Note**: The random algorithm currently only supports datasets whose names start with:
- `rcabench`
- `rcaeval`

**Expected output:**

```
[DEBUG] enter single
[INFO] enter run_single
[DEBUG] enter Random.__call__
[DEBUG] found 33 service names
[DEBUG] exit Random.__call__ duration=0.163444s
[DEBUG] len(answers)=33
[DEBUG] hit: {'level': 'service', 'name': 'ts-auth-service', 'rank': 31, ...}
[DEBUG] saved parquet to output/.../random/output.parquet
[DEBUG] saved parquet to output/.../random/perf.parquet
[INFO] exit run_single duration=0.384085s
```

**Note**: If you see `skipping output/.../random` message, the algorithm was already run. Use `--clear` flag to re-run:

```bash
./main.py eval single random rcabench_filtered ts0-ts-auth-service-stress-jv8m9r --clear
```

## Understanding the Output

The evaluation produces two key outputs:

1. **Ranked Services** (`output.parquet`): A list of all services ranked by suspected root cause probability
2. **Performance Metrics** (`perf.parquet`): Evaluation metrics comparing predictions to ground truth

Example metrics from the random algorithm:

```
MRR: 0.045455
Avg@1: 0.0
Avg@3: 0.0
Avg@5: 0.0
AC@1: 0.0
AC@3: 0.0
AC@5: 0.0
```

**Metrics explained:**
- **MRR (Mean Reciprocal Rank)**: Average of 1/rank for the first correct answer (higher is better, max 1.0)
- **Avg@k**: Average number of correct answers in top-k predictions
- **AC@k (Top-k Accuracy)**: Percentage of cases where at least one correct answer appears in top-k

The random algorithm serves as a baseline and is expected to perform poorly, but demonstrates the evaluation framework works correctly.

## Step 5: Explore the Algorithm Code

Look at the `random` implementation to understand the algorithm interface:

```bash
cat src/rcabench_platform/v2/algorithms/random_.py
```

Key components:
- `Algorithm` base class with `__call__` method
- `AlgorithmArgs` input containing dataset name, datapack ID, and input folder path
- `AlgorithmAnswer` output with ranked root cause predictions (level, name, rank)
- The algorithm reads parquet files from `input_folder` and returns a list of ranked services

## Next Steps

Now that you've run your first algorithm:

1. **Understand the data format**: Read [Data Formats](../development-guide/data-formats) to learn about input parquet files
2. **Implement your algorithm**: Follow the [Development Guide](../development-guide) to create your own RCA algorithm
3. **Evaluate locally**: Use [Local Evaluation](../local-evaluation) to test on multiple datasets
4. **Submit remotely**: Package and submit via [Remote Evaluation](../remote-evaluation)

## Troubleshooting

### Dataset Not Found Error

**Error**: `FileNotFoundError: No such file or directory ... data/rcabench-platform-v2/data/...`

**Solution**: Ensure datasets are properly mounted and symlinked:

```bash
# Check if JuiceFS is mounted
ls /mnt/jfs/rcabench-platform-v2

# Create symlink if missing
cd rcabench-platform
mkdir -p data
ln -s /mnt/jfs/rcabench-platform-v2 data/
```

### Datapack Not Found Error

**Error**: `AssertionError: Labels for datapack '0' not found in dataset 'xxx'`

**Cause**: Datapacks use descriptive names (not numeric IDs like `0`).

**Solution**: List available datapacks and use the full name:

```bash
# List all datasets
./main.py eval show-datasets

# List datapacks in a dataset
ls data/rcabench-platform-v2/data/rcabench_filtered/

# Use the full datapack name
./main.py eval single random rcabench_filtered ts0-ts-auth-service-stress-jv8m9r
```

### Algorithm Not Supported for Dataset

**Error**: `NotImplementedError` when running algorithm on certain datasets

**Cause**: The `random` algorithm only supports datasets starting with `rcabench` or `rcaeval`.

**Solution**: Use a compatible dataset:

```bash
# These work with random algorithm
./main.py eval single random rcabench_filtered <datapack-name>
./main.py eval single random rcaeval_re2_tt <datapack-name>

# Check available datasets
./main.py eval show-datasets
```

### Import Errors

**Error**: `ModuleNotFoundError` or import errors

**Solution**: Verify installation with correct Python version:

```bash
# Check Python version (must be 3.10+)
python --version

# Reinstall dependencies
uv sync --all-extras

# Verify installation
./main.py self test
```

### Permission Errors

**Error**: Permission denied when accessing data directory

**Solution**: Check file permissions and JuiceFS mount:

```bash
# Check JuiceFS mount status
mount | grep juicefs

# Remount if necessary (use your JUICEFS_REDIS_URL from .env)
sudo juicefs mount ${JUICEFS_REDIS_URL} /mnt/jfs -d
# Default: redis://10.10.10.119:6379/1

# Check data directory permissions
ls -la data/
```

For more help, see [Troubleshooting](../troubleshooting).
