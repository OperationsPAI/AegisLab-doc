---
title: Observation
weight: 3
type: docs
---

## Introduction

In the RCABench Platform, a `Trace` allows you to observe the entire workflow as some serial tasks. There are 4 kinds of `Trace`:

- **Container Trace**: (First Task `BuildImage`) Represents the workflow of building container images for RCABench.
- **Injection Trace**: (First Task `RestartService`) Represents the workflow of a fault injection experiment.
- **Datapacks Trace**: (First Task `BuildDataset`) Represents the workflow of building datapacks for root cause analysis.
- **Algorithm Trace**: (First Task `RunAlgorithm`)Represents the workflow of root cause analysis.

Each task type has some event types to show the task status.

## Basic Setup

{{< cards cols="1" >}}
{{< card link="../../workflow-guide#Pre-Work" title="Pre-Work" icon="arrow-circle-right">}}
{{< /cards >}}

```python {filename="client.py"}
from rcabench.openapi.api import TraceApi

trace_api = TraceApi(rcabench_client)
```

## Viewing Trace Progress

> [!IMPORTANT]
> All tasks are **asynchronous**, so that we should use **SSE(Server-Sent Events)** in server to observe the progress of tasks in real-time.

```python {filename="observe_progress.py"}
from rcabench.openapi.models import ConstsSSEEventName, DtoStreamEvent

try:
    sse_client = traces_api.api_v2_traces_id_stream_get_sse(
        id="bd1d7e15-f15d-4d77-b4f2-46f82e95ee96",  # Trace ID
        last_id="0",  # Last Event ID in Redis defaults to "0"
    )
    for e in sse_client.events():
        if e.event == ConstsSSEEventName.EventEnd:
            break

        assert e.event == ConstsSSEEventName.EventUpdate, f"Unexpected event type: {e.event}"

        streamEvent = DtoStreamEvent.from_json(e.data)
        assert isinstance(streamEvent, DtoStreamEvent), "Expected event to be DtoStreamEvent"
        pprint(streamEvent)

except Exception as e:
    pprint(f"Trace stream failed unexpectedly: {e}")

finally:
    sse_client.close()

# Expected Output:
# DtoStreamEvent(event_name=<ConstsEventType.EventAlgoRunSucceed: 'algorithm.run.succeed'>, payload={'algorithm': {'name': 'detector', 'tag': 'latest', 'env_vars': {}}, 'dataset': 'ts0-ts-assurance-service-cpu-exhaustion-cgm9dx', 'execution_id': 8, 'timestamp': '20251024_123258'}, task_id='8cf0e85e-e554-4893-9783-c541429040d4', task_type=<ConstsTaskType.TaskTypeRunAlgorithm: 'RunAlgorithm'>, additional_properties={'file_name': 'controller.go', 'function_name': 'k8s.io/client-go/tools/cache.ResourceEventHandlerFuncs.OnUpdate', 'line': 253})
```
