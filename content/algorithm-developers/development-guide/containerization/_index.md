---
title: Containerization
weight: 3
---


Package your RCA algorithm as a Docker container for remote execution on AegisLab.

## Overview

Containerization allows your algorithm to:
- Run on AegisLab's Kubernetes cluster
- Execute in isolated environments
- Use specific dependencies and versions
- Scale across multiple nodes

## Prerequisites

- Docker or Podman installed
- Your algorithm tested locally
- Access to container registry (Docker Hub, Harbor, etc.)

## Container Structure

Your algorithm container must include:

```
my-rca-algorithm/
├── Dockerfile              # Container definition
├── entrypoint.sh          # Execution script
├── info.toml              # Algorithm metadata
├── main.py                # Algorithm registration and CLI entry point
├── pyproject.toml         # Python project configuration (for uv)
├── uv.lock                # Locked dependencies (for uv)
└── src/                   # Your algorithm implementation
    └── my_algorithm.py
```

## Creating the Dockerfile

### Option 1: Using rcabench-platform Base Image (Recommended)

This is the simplest approach - use the pre-built rcabench-platform image:

```dockerfile
FROM 10.10.10.240/library/rcabench-platform:latest

COPY entrypoint.sh /entrypoint.sh

CMD ["/entrypoint.sh"]
```

**Advantages:**
- Smallest image size (only adds your entrypoint script)
- All dependencies pre-installed
- Fastest build time
- Consistent environment

**Use when:** Your algorithm only uses built-in rcabench-platform algorithms or has no additional dependencies.

### Option 2: Using uv with Custom Dependencies

For algorithms with additional dependencies, use uv for fast, reproducible builds:

```dockerfile
FROM ghcr.io/astral-sh/uv:bookworm-slim AS builder

ENV UV_LINK_MODE=copy

RUN uv python install 3.13

WORKDIR /app

# Install dependencies first (cached layer)
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev

# Copy algorithm code
ADD . /app

# Install project
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

COPY entrypoint.sh /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

**Advantages:**
- Fast dependency resolution with uv
- Reproducible builds with uv.lock
- Support for custom dependencies
- Docker layer caching for dependencies

**Use when:** Your algorithm requires additional Python packages beyond rcabench-platform.

## Creating the Entrypoint Script

The entrypoint script executes your algorithm using the rcabench-platform container interface:

```bash
#!/bin/bash -ex
export ALGORITHM=${ALGORITHM:-my-algorithm}
LOGURU_COLORIZE=0 .venv/bin/python main.py container run
```

**Key points:**
- `#!/bin/bash -ex`: Enable error exit and command echoing for debugging
- `ALGORITHM`: Environment variable specifying which algorithm to run (set by AegisLab)
- `LOGURU_COLORIZE=0`: Disable colored logs for better readability in K8s logs
- `main.py container run`: Invokes the rcabench-platform container CLI

**Environment variables provided by AegisLab:**
- `ALGORITHM`: Algorithm name to execute
- `INPUT_PATH`: Path to input data (parquet files)
- `OUTPUT_PATH`: Path to write results
- `EXECUTION_ID`: Execution ID for result submission
- `RCABENCH_BASE_URL`: AegisLab API URL for result submission
- `RCABENCH_SUBMITION`: Set to "false" to disable automatic submission (for testing)

## Creating info.toml

Metadata file identifying your algorithm:

```toml
name = "my-algorithm"
tag = "v1.0.0"
```

**Fields:**
- `name`: Algorithm name (must match the name registered in your main.py)
- `tag`: Optional version tag for the container image

Note: Resource requirements (memory, CPU, timeout) are configured at runtime when submitting execution tasks via the AegisLab API, not in the container metadata.

## Algorithm Registration (main.py)

Your algorithm container needs a main.py file that registers your algorithm with the rcabench-platform CLI:

```python
#!/usr/bin/env -S uv run -s
from rcabench_platform.v3.cli.main import main
from rcabench_platform.v3.sdk.algorithms.spec import global_algorithm_registry
from my_algorithm import MyAlgorithm

if __name__ == "__main__":
    # Register your algorithm
    registry = global_algorithm_registry()
    registry["my-algorithm"] = MyAlgorithm

    # Run CLI with builtin algorithms disabled (optional)
    main(enable_builtin_algorithms=False)
```

**Key points:**
- Import your algorithm class
- Register it in the global registry with a unique name
- The name must match the `ALGORITHM` environment variable
- Set `enable_builtin_algorithms=False` if you only want your algorithm available

**Example from baro algorithm:**

```python
#!/usr/bin/env -S uv run -s
from rcabench_platform.v3.cli.main import main
from rcabench_platform.v3.sdk.algorithms.spec import global_algorithm_registry
from baro.baro import Baro

if __name__ == "__main__":
    registry = global_algorithm_registry()
    registry["baro"] = Baro

    main(enable_builtin_algorithms=False)
```

## Python Project Configuration (pyproject.toml)

For algorithms with custom dependencies, create a pyproject.toml:

```toml
[project]
name = "my-algorithm"
version = "0.1.0"
description = "My RCA algorithm"
requires-python = ">=3.13"
dependencies = [
    "rcabench>=1.1.34",
    "rcabench-platform>=0.4.1",
    # Add your custom dependencies here
]

[build-system]
requires = ["uv_build>=0.7.12,<0.8.0"]
build-backend = "uv_build"
```

Then generate uv.lock:

```bash
uv lock
```

## Building the Container

### Build Image

```bash
# Navigate to your algorithm directory
cd my-rca-algorithm

# Build the image
docker build -t my-algorithm:latest .

# Verify the image was created
docker images | grep my-algorithm
```

### Tag for Harbor Registry

Tag your image for the AegisLab Harbor registry:

```bash
# Format: <registry>/<namespace>/<algorithm>:<tag>
docker tag my-algorithm:latest 10.10.10.240/library/my-algorithm:v1.0.0

# Also tag as latest
docker tag my-algorithm:latest 10.10.10.240/library/my-algorithm:latest
```

**Harbor Registry Configuration:**
- Registry: `10.10.10.240`
- Namespace: `library` (default) or your project namespace
- Use semantic versioning for tags: `v1.0.0`, `v1.1.0`, etc.

### Push to Harbor Registry

```bash
# Login to Harbor registry
docker login 10.10.10.240
# Username: admin (or your Harbor username)
# Password: Harbor12345 (or your Harbor password)

# Push the versioned image
docker push 10.10.10.240/library/my-algorithm:v1.0.0

# Push the latest tag
docker push 10.10.10.240/library/my-algorithm:latest
```

## Testing the Container

### Local Test with Environment Variables

Create a test script that mimics the AegisLab execution environment:

```bash
#!/bin/bash
# test_container.sh

# Prepare test data directory
mkdir -p test_data/input test_data/output

# Copy test dataset to input directory
# (Use an actual datapack from rcabench_dataset)
cp -r /mnt/jfs/rcabench_dataset/ts0-ts-auth-service-stress-jv8m9r/* test_data/input/

# Run container with environment variables
docker run --rm \
    -v $(pwd)/test_data/input:/input:ro \
    -v $(pwd)/test_data/output:/output \
    -e ALGORITHM=my-algorithm \
    -e INPUT_PATH=/input \
    -e OUTPUT_PATH=/output \
    -e RCABENCH_SUBMITION=false \
    my-algorithm:latest

# Check output
ls -la test_data/output/
```

### Interactive Debugging

Run the container interactively to debug issues:

```bash
# Start shell in container
docker run -it --rm \
    -v $(pwd)/test_data/input:/input:ro \
    -v $(pwd)/test_data/output:/output \
    -e ALGORITHM=my-algorithm \
    -e INPUT_PATH=/input \
    -e OUTPUT_PATH=/output \
    --entrypoint /bin/bash \
    my-algorithm:latest

# Inside container, test manually:
export ALGORITHM=my-algorithm
export RCABENCH_SUBMITION=false
.venv/bin/python main.py container run
```

### Verify Output Format

The container should produce results in the expected format:

```python
# verify_output.py
import pandas as pd
from pathlib import Path

output_dir = Path("test_data/output")

# Check for result file
result_files = list(output_dir.glob("*_result_*.csv"))
if result_files:
    df = pd.read_csv(result_files[0])
    print(f"Found {len(df)} results")
    print(df.head())

    # Verify required columns
    required_cols = ["level", "result", "rank"]
    assert all(col in df.columns for col in required_cols), f"Missing columns: {set(required_cols) - set(df.columns)}"
    print("Output format is valid")
```

## Registering with AegisLab

After pushing your container to Harbor, register it with AegisLab using the Python SDK:

```python
from rcabench.openapi import ApiClient, Configuration, ContainersApi
from rcabench.openapi.models import CreateContainerReq, ContainerType
import os

# Configure API client
config = Configuration(host=os.getenv("AEGISLAB_API_URL", "http://10.10.10.220:32080"))
client = ApiClient(config)
api = ContainersApi(client)

# Register algorithm container
req = CreateContainerReq(
    name="my-algorithm",
    type=ContainerType.Algorithm,
    image="10.10.10.240/library/my-algorithm",
    tag="v1.0.0",
    description="My RCA algorithm for microservices root cause analysis"
)

response = api.create_container(req)
print(f"Container registered with ID: {response.data.id}")
```

**Note:** You can also register containers through the AegisLab web UI at `${AEGISLAB_API_URL}/containers`.

## Best Practices

### Use the Base Image When Possible

The simplest and most reliable approach is to use the rcabench-platform base image:

```dockerfile
FROM 10.10.10.240/library/rcabench-platform:latest

COPY entrypoint.sh /entrypoint.sh

CMD ["/entrypoint.sh"]
```

This is how the builtin algorithms (random, traceback) are containerized.

### Keep Images Small

When using custom dependencies:

```dockerfile
# Use uv's slim base image
FROM ghcr.io/astral-sh/uv:bookworm-slim AS builder

# Use cache mounts for faster rebuilds
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# Don't include dev dependencies
uv sync --frozen --no-dev
```

### Pin Dependencies with uv.lock

Always include a `uv.lock` file for reproducible builds:

```bash
# Generate lock file
uv lock

# Verify lockfile is up to date
uv lock --check
```

### Use Multi-Stage Builds for Compiled Dependencies

```dockerfile
FROM ghcr.io/astral-sh/uv:bookworm-slim AS builder

RUN uv python install 3.13
WORKDIR /app

# Install dependencies
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev

ADD . /app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

### Handle Errors Gracefully

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    try:
        # Algorithm logic
        pass
    except Exception as e:
        logger.error(f"Algorithm error: {e}")
        # Return empty result instead of crashing
        return []
```

### Log Progress for Debugging

```python
from rcabench_platform.v3.sdk.logging import logger

def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    logger.info(f"Loading traces from {args.input_folder}")
    traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    logger.info(f"Loaded {traces.height} traces")

    logger.info("Analyzing error patterns...")
    # Analysis logic

    logger.info(f"Found {len(ranked_services)} suspected services")
    return ranked_services
```

## Troubleshooting

### Container Fails to Start

**Symptom:**
```
Error: container exited with code 1
```

**Solutions:**

1. Check container logs:
```bash
docker logs <container-id>
```

2. Run interactively for debugging:
```bash
docker run -it --entrypoint /bin/bash my-algorithm:latest
```

3. Verify entrypoint.sh is executable:
```dockerfile
RUN chmod +x /entrypoint.sh
```

4. Check for missing environment variables:
```bash
docker run --rm \
    -e ALGORITHM=my-algorithm \
    -e INPUT_PATH=/input \
    -e OUTPUT_PATH=/output \
    my-algorithm:latest
```

### Algorithm Not Found in Registry

**Symptom:**
```
AssertionError: Unknown algorithm: my-algorithm
```

**Solutions:**

1. Verify algorithm is registered in main.py:
```python
registry = global_algorithm_registry()
registry["my-algorithm"] = MyAlgorithm  # Name must match ALGORITHM env var
```

2. Check ALGORITHM environment variable matches registered name:
```bash
export ALGORITHM=my-algorithm  # Must match registry key
```

3. Verify algorithm class is imported correctly:
```python
from my_algorithm import MyAlgorithm  # Check import path
```

### Permission Errors

**Symptom:**
```
PermissionError: [Errno 13] Permission denied: '/output/result.csv'
```

**Solutions:**

1. Ensure output directory is writable:
```bash
docker run --rm \
    -v $(pwd)/output:/output:rw \  # Add :rw for read-write
    my-algorithm:latest
```

2. Create non-root user in Dockerfile (if needed):
```dockerfile
RUN useradd -m -u 1000 algorithm
USER algorithm
```

3. Check directory ownership:
```bash
ls -la output/
chmod 777 output/  # For testing only
```

### Missing Dependencies

**Symptom:**
```
ModuleNotFoundError: No module named 'polars'
```

**Solutions:**

1. Verify dependencies are in pyproject.toml:
```toml
dependencies = [
    "rcabench-platform>=0.4.1",
    "polars>=0.19.0",
]
```

2. Regenerate uv.lock:
```bash
uv lock
```

3. Test dependencies in container:
```bash
docker run -it my-algorithm:latest /bin/bash
.venv/bin/python -c "import polars; print(polars.__version__)"
```

### Large Image Size

**Symptom:**
```
Image size: 2.5GB
```

**Solutions:**

1. Use rcabench-platform base image (smallest):
```dockerfile
FROM 10.10.10.240/library/rcabench-platform:latest
COPY entrypoint.sh /entrypoint.sh
CMD ["/entrypoint.sh"]
```

2. Use slim base images:
```dockerfile
FROM ghcr.io/astral-sh/uv:bookworm-slim  # Not bookworm
```

3. Check image layers:
```bash
docker history my-algorithm:latest
```

4. Remove unnecessary files:
```dockerfile
# Add .dockerignore file
.git
__pycache__
*.pyc
.venv
data/
output/
```

### Harbor Push Fails

**Symptom:**
```
Error: unauthorized: authentication required
```

**Solutions:**

1. Login to Harbor registry:
```bash
docker login 10.10.10.240
# Username: admin
# Password: Harbor12345
```

2. Verify image tag format:
```bash
# Correct format: <registry>/<namespace>/<image>:<tag>
docker tag my-algorithm:latest 10.10.10.240/library/my-algorithm:v1.0.0
```

3. Check Harbor project permissions:
- Ensure you have push access to the `library` project
- Contact Harbor administrator if needed

### Container Runs But No Output

**Symptom:**
Container completes successfully but no results in output directory.

**Solutions:**

1. Check if submission is disabled:
```bash
# Enable local output for testing
docker run --rm \
    -e RCABENCH_SUBMITION=false \
    -v $(pwd)/output:/output \
    my-algorithm:latest
```

2. Verify algorithm returns results:
```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    # Must return non-empty list
    return [
        AlgorithmAnswer(level="service", name="service-a", rank=1),
        AlgorithmAnswer(level="service", name="service-b", rank=2),
    ]
```

3. Check container logs for errors:
```bash
docker logs <container-id> 2>&1 | grep -i error
```

### Input Data Not Found

**Symptom:**
```
FileNotFoundError: [Errno 2] No such file or directory: '/input/injection.json'
```

**Solutions:**

1. Verify input directory is mounted:
```bash
docker run --rm \
    -v /path/to/datapack:/input:ro \  # Must contain injection.json
    my-algorithm:latest
```

2. Check datapack structure:
```bash
ls -la /path/to/datapack/
# Should contain: injection.json, abnormal_traces.parquet, etc.
```

3. Use actual datapack from JuiceFS:
```bash
# Copy from JuiceFS mount
cp -r /mnt/jfs/rcabench_dataset/ts0-ts-auth-service-stress-jv8m9r test_data/input/
```

### Build Fails with uv

**Symptom:**
```
Error: failed to solve: process "/bin/sh -c uv sync" did not complete successfully
```

**Solutions:**

1. Verify uv.lock is committed:
```bash
git add uv.lock
git commit -m "Add uv.lock"
```

2. Use cache mounts for faster builds:
```dockerfile
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev
```

3. Check pyproject.toml syntax:
```bash
uv lock --check
```

4. Ensure Python version is available:
```dockerfile
RUN uv python install 3.13  # Before uv sync
```

## Next Steps

- [Remote Evaluation](../remote-evaluation): Submit your containerized algorithm
- [Examples](../examples): Study example containerized algorithms
- [Troubleshooting](../troubleshooting): Common containerization issues
