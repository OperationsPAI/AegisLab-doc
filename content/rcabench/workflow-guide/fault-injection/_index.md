---
title: Fault Injection
weight: 1
type: docs
---

描述如何进行故障注入，采集数据，包括常见的故障注入场景、参数配置、数据采集、怎么观察数据采集进度、怎么获取到数据集等、怎么调试等

## Basic Setup

{{< cards cols="1" >}}
{{< card link="../../workflow-guide" title="Pre-Work" icon="arrow-circle-right">}}
{{< /cards >}}

```python {filename="client.py"}
from rcabench.openapi.api import DatasetApi, InjectionApi

dataset_api = DatasetApi(rcabench_client)
injection_api = InjectionApi(rcabench_client)
```

## Fault Injection

> [!NOTE]
> This task is **asynchronous**. The response of submission will indicate submission success, but the actual fault injection and data collection will run in the background.

### Submit Request

```python {filename="fault_injection.py"}
from rcabench.openapi.models import (
    DtoAlgorithmItem,
    DtoLabelItem,
    DtoSubmitInjectionReq,
    HandlerNode,
)

req = DtoSubmitInjectionReq(
    algorithms=[
        DtoAlgorithmItem(name="traceback"),
    ],  # Algorithm execution container name (With a value can execute root cause analysis)
    benchmark="clickhouse",  # Data collection container name (With a value can execute data collection)
    container_name="ts_cn",  # Pedestal container name
    container_tag="v1.0.0-213-gf9294111",  # Pedestal container tag
    interval=2,  # The whole period in minutes
    project_name="pair_diagnosis",
    pre_duration=1,  # Normal time in minutes
    specs=[ # Fault specification list
        HandlerNode(
            children={
                "4": HandlerNode(
                    children={
                        "0": HandlerNode(value=1),
                        "1": HandlerNode(value=0),
                        "2": HandlerNode(value=5),
                        "3": HandlerNode(value=1),
                        "4": HandlerNode(value=1),
                    },
                )
            },
            value=4,
        )
    ],
    labels=[ # Labels for all injections
        DtoLabelItem(key="env", value="testing"),
        DtoLabelItem(key="batch", value="bootstrap"),
    ],
)

resp = injection_api.api_v1_injections_post(body=req)
```

### Observation

{{< cards cols="1" >}}
{{< card link="../observe-progress" title="Observation" icon="arrow-circle-right">}}
{{< /cards >}}

## Fault Types

The fault injection parameters are defined using the `HandlerNode` structure. Each fault type has its own set of parameters that need to be configured.

{{% details title="Types" closed="true" %}}

### Network Delay

This scenario helps evaluate how your system handles slow network connections and communication delays.

```python
HandlerNode(
    value=17,
    children={
        "17": HandlerNode(
            children={
                "0": HandlerNode(value=4), # Duration in seconds
                "1": HandlerNode(value=0), # Namespace (dynamic)
                "2": HandlerNode(value=1), # NetworkPairIdx (dynamic)
                "3": HandlerNode(value=1000), # Latency in milliseconds
                "4": HandlerNode(value=10), # Correlation percentage
                "5": HandlerNode(value=500), # Jitter in milliseconds
                "6": HandlerNode(value=1), # Direction (1=to, 2=from, 3=both)
            }
        )
    },
)
```

### Memory Pressure

This scenario helps identify memory leaks, inefficient memory usage, and tests memory management strategies.

```python
HandlerNode(
    value=3,
    children={
        "3": HandlerNode(
            children={
                "0": HandlerNode(value=4), # Duration in seconds
                "1": HandlerNode(value=0), # Namespace (dynamic)
                "2": HandlerNode(value=1), # Container Index (dynamic)
                "3": HandlerNode(value=512), # Memory Size Unit MB
                "4": HandlerNode(value=1), # Memory Stress Threads
            }
        )
    },
)
```

### Pod Failure

This scenario validates auto-scaling, service mesh resilience, and application restart policies.

```python
HandlerNode(
    value=1,
    children={
        "1": HandlerNode(
            children={
                "0": HandlerNode(value=4), # Duration in seconds
                "1": HandlerNode(value=0), # Namespace (dynamic)
                "2": HandlerNode(value=1), # App Index (dynamic)
            }
        )
    },
)
```

{{% /details %}}s

## Reference

For more fault types and parameters, see:

- [Fault Types Documentation](../reference/fault-types)
- [HandlerNode API Reference](../reference/api)
