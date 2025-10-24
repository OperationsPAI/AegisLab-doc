---
title: Workflow Guide
weight: 4
type: docs
sidebar:
  open: true
---

## Overview

> [!Important]
> Before following this guide, ensure you have completed the initial setup as described in the [Getting Started](/rcabench/getting-started/) guide.

This guide provides a comprehensive overview of the workflow for using RCABench by **python SDK**.

## Pre-Work

{{% steps %}}

### Setup Python Environment

```bash
# Install RCABench SDK
pip install rcabench

# Or install from source
cd sdk/python
pip install -e .
```

### Create `.env` File

```ini {filename=".env"}
RCABENCH_BASE_URL=http://127.0.0.1:8082
RCABENCH_USERNAME=admin
RCABENCH_PASSWORD=admin123
```

### Configure RCABench Client

```python {filename="client.py"}
from dotenv import load_dotenv

from rcabench.client import RCABenchClient
from rcabench.openapi.api.system_api import SystemApi

load_dotenv()

# Initialize client to connect to local RCABench server
rcabench_client = RCABenchClient()

# Verify connection
system_api = SystemApi(rcabench_client)
print(system_api.system_health_get())

# Expected response:
# {
#   "code": 200,
#   "message": "Success",
#   "data": {
#     "status": "healthy",
#     "timestamp": "2025-09-25T10:30:45Z",
#     "version": "1.0.0",
#     "uptime": "72h30m15s",
#     "services": {
#       "database": {
#         "status": "healthy",
#         "last_checked": "2025-09-25T10:30:45Z",
#         "response_time": "5ms"
#       }
#     }
#   }
# }
```

{{% /steps %}}
