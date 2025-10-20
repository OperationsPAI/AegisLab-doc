---
title: Data Collection via Fault Injection
weight: 4
---

描述如何进行故障注入，采集数据，包括常见的故障注入场景、参数配置、数据采集、怎么观察数据采集进度、怎么获取到数据集等、怎么调试等

## 🚀 Basic Fault Injection Setup

```python
from rcabench.openapi import (
    ApiClient,
    Configuration,
    DtoSubmitInjectionReq,
    HandlerNode,
    InjectionApi,
)

BASE_URL = "http://localhost:8082"

client = ApiClient(configuration=Configuration(host=BASE_URL))
injector = InjectionApi(api_client=client)
injector.api_v1_injections_post(
    body=DtoSubmitInjectionReq(
      benchmark="ts",
      interval=15,
      pre_duration=15,
      project_name="default",
      specs=[], # Define fault specs here
    )
)
```

---

## 🔧 Fault Injection Scenarios

### Scenario 1: Network Delay Injection

> [!EXPERIMENT]
>
> Test application resilience against network latency issues between microservices. This scenario helps evaluate how your system handles slow network connections and communication delays.

```python
specs=[
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
],
```

### Scenario 2: Memory Pressure

> [!EXPERIMENT]
>
> Evaluate system behavior under high memory consumption conditions. This scenario helps identify memory leaks, inefficient memory usage, and tests memory management strategies.

```python
specs=[
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
],
```

### Scenario 3: Pod Failure

> [!EXPERIMENT]
>
> Test system fault tolerance and recovery mechanisms when pods unexpectedly terminate. This scenario validates auto-scaling, service mesh resilience, and application restart policies.

```python
specs=[
    HandlerNode(
        value=3,
        children={
            "3": HandlerNode(
                children={
                    "0": HandlerNode(value=4), # Duration in seconds
                    "1": HandlerNode(value=0), # Namespace (dynamic)
                    "2": HandlerNode(value=1), # App Index (dynamic)
                }
            )
        },
    )
],
```

---

## 📋 Best Practices

> [!NOTE]
>
> 1. **Start Small**: Begin with low-impact faults to understand > system behavior
> 2. **Monitor Impact**: Watch system metrics during fault injection
> 3. **Document Scenarios**: Keep detailed records of fault > configurations
> 4. **Safety First**: Always have rollback procedures ready

---

## 📚 Reference

For more fault types and parameters, see:

- [Fault Types Documentation](../reference/fault-types)
- [HandlerNode API Reference](../reference/api)
