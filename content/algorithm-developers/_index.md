---
title: Algorithm Developer Guide
weight: 1
---

Welcome to the Algorithm Developer path! This guide helps you develop and evaluate root cause analysis (RCA) algorithms using the AegisLab ecosystem.

## What You'll Learn

- How to implement RCA algorithms using the standardized interface
- Understanding input data formats (traces, metrics, logs)
- Local evaluation with rcabench-platform
- Remote evaluation via AegisLab
- Contributing algorithms to the community

## Quick Navigation

{{< cards >}}
  {{< card link="quickstart" title="Quickstart" icon="play" >}}
  {{< card link="development-guide" title="Development Guide" icon="code" >}}
  {{< card link="local-evaluation" title="Local Evaluation" icon="beaker" >}}
  {{< card link="remote-evaluation" title="Remote Evaluation" icon="document" >}}
  {{< card link="examples" title="Examples" icon="sparkles" >}}
  {{< card link="reference" title="Reference" icon="book-open" >}}
{{< /cards >}}

## Typical Workflow

1. **Setup Environment**: Install rcabench-platform and dependencies
2. **Develop Algorithm**: Implement the `Algorithm` base class
3. **Test Locally**: Evaluate on sample datasets using `eval single`
4. **Containerize**: Package as Docker image with metadata
5. **Submit for Evaluation**: Upload to AegisLab for remote execution
6. **Analyze Results**: Review metrics (MRR, Avg@k, Top-k accuracy)

## Key Concepts

### Algorithm Interface

All RCA algorithms implement a standardized interface:

```python
from rcabench_platform.v3.sdk.algorithms.spec import Algorithm, AlgorithmArgs, AlgorithmAnswer

class MyRCAAlgorithm(Algorithm):
    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        # Your algorithm logic here
        pass
```

### Input Data Format

Algorithms receive standardized parquet files containing:
- **Traces**: Distributed traces with spans, operations, and attributes
- **Metrics**: Time-series metrics (latency, error rates, resource usage)
- **Logs**: Structured logs with timestamps and context
- **Ground Truth**: Fault injection metadata for evaluation

### Evaluation Metrics

- **MRR (Mean Reciprocal Rank)**: Average of 1/rank for first correct answer
- **Avg@k**: Average number of correct answers in top-k predictions
- **Top-k Accuracy**: Percentage of cases where correct answer is in top-k

## Prerequisites

- Python 3.10+
- Basic understanding of distributed tracing
- Familiarity with microservices architecture
- Knowledge of root cause analysis concepts

## Next Steps

Start with the [Quickstart Guide](quickstart) to run your first algorithm in under 15 minutes.
