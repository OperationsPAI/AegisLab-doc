---
title: ML-Based RCA
weight: 2
---

An example machine learning-based root cause analysis algorithm using graph neural networks.

## Overview

This example demonstrates how to build an ML-based RCA algorithm that:
1. Constructs a service dependency graph from traces
2. Extracts features from spans (latency, error rate, call frequency)
3. Uses a graph neural network to predict root causes
4. Ranks services by prediction confidence

## Architecture

```
Traces → Graph Construction → Feature Extraction → GNN Model → Ranked Services
```

## Implementation

### Algorithm Structure

```python
from rcabench_platform.v2.algorithms import Algorithm, AlgorithmArgs, AlgorithmAnswer
import polars as pl
import torch
import torch.nn as nn
from torch_geometric.nn import GCNConv
from torch_geometric.data import Data

class MLBasedRCA(Algorithm):
    def __init__(self):
        super().__init__()
        self.model = self.load_model()

    def load_model(self):
        # Load pre-trained model or initialize new one
        model = GNNModel(input_dim=10, hidden_dim=64, output_dim=1)
        # Load weights if available
        return model

    def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
        # Load data
        traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

        # Build graph
        graph_data = self.build_graph(traces)

        # Extract features
        features = self.extract_features(traces, graph_data)

        # Run model
        predictions = self.model(features, graph_data.edge_index)

        # Rank services
        ranked_services = self.rank_services(predictions, graph_data.service_names)

        return [
            AlgorithmAnswer(level="service", name=service_name, rank=rank)
            for rank, (service_name, score) in enumerate(ranked_services, start=1)
        ]

    def build_graph(self, traces):
        # Implementation details below
        pass

    def extract_features(self, traces, graph_data):
        # Implementation details below
        pass

    def rank_services(self, predictions, service_names):
        # Implementation details below
        pass
```

### Graph Construction

```python
def build_graph(self, traces):
    """Build service dependency graph from traces."""

    # Get unique services
    services = traces.select("service_name").unique().to_series().to_list()
    service_to_idx = {svc: idx for idx, svc in enumerate(services)}

    # Build edges (parent -> child relationships)
    edges = traces.select(["parent_service", "service_name"]).unique()
    edges = edges.filter(
        pl.col("parent_service").is_not_null() &
        pl.col("parent_service").is_in(services)
    )

    # Convert to edge index format
    edge_list = []
    for row in edges.iter_rows(named=True):
        parent_idx = service_to_idx[row["parent_service"]]
        child_idx = service_to_idx[row["service_name"]]
        edge_list.append([parent_idx, child_idx])

    edge_index = torch.tensor(edge_list, dtype=torch.long).t()

    return Data(
        edge_index=edge_index,
        num_nodes=len(services),
        service_names=services
    )
```

### Feature Extraction

```python
def extract_features(self, traces, graph_data):
    """Extract node features for each service."""

    features = []

    for service in graph_data.service_names:
        service_spans = traces.filter(pl.col("service_name") == service)

        # Calculate features
        total_spans = len(service_spans)
        error_count = service_spans.filter(pl.col("status_code") == "ERROR").height
        error_rate = error_count / total_spans if total_spans > 0 else 0

        # Latency features
        latencies = service_spans.select(
            (pl.col("end_time") - pl.col("start_time")).alias("latency")
        )
        avg_latency = latencies["latency"].mean() if len(latencies) > 0 else 0
        max_latency = latencies["latency"].max() if len(latencies) > 0 else 0
        p95_latency = latencies["latency"].quantile(0.95) if len(latencies) > 0 else 0

        # Call frequency
        call_frequency = total_spans

        # Downstream error propagation
        downstream_errors = self.count_downstream_errors(service, traces)

        # Combine features
        feature_vector = [
            error_rate,
            error_count,
            avg_latency,
            max_latency,
            p95_latency,
            call_frequency,
            downstream_errors,
            total_spans,
            # Add more features as needed
        ]

        features.append(feature_vector)

    return torch.tensor(features, dtype=torch.float)
```

### GNN Model

```python
class GNNModel(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim):
        super(GNNModel, self).__init__()
        self.conv1 = GCNConv(input_dim, hidden_dim)
        self.conv2 = GCNConv(hidden_dim, hidden_dim)
        self.conv3 = GCNConv(hidden_dim, output_dim)
        self.relu = nn.ReLU()
        self.sigmoid = nn.Sigmoid()

    def forward(self, x, edge_index):
        # First GCN layer
        x = self.conv1(x, edge_index)
        x = self.relu(x)

        # Second GCN layer
        x = self.conv2(x, edge_index)
        x = self.relu(x)

        # Output layer
        x = self.conv3(x, edge_index)
        x = self.sigmoid(x)  # Probability of being root cause

        return x
```

### Ranking Services

```python
def rank_services(self, predictions, service_names):
    """Rank services by prediction confidence."""

    # Convert predictions to numpy
    scores = predictions.detach().numpy().flatten()

    # Create (service, score) pairs
    service_scores = list(zip(service_names, scores))

    # Sort by score descending
    service_scores.sort(key=lambda x: x[1], reverse=True)

    return service_scores
```

## Training the Model

### Data Preparation

```python
def prepare_training_data(dataset_path):
    """Prepare training data from multiple datapacks."""

    training_samples = []

    for datapack in datapack_list:  # Iterate through available datapacks
        traces = pl.read_parquet(f"{dataset_path}/{datapack}/abnormal_traces.parquet")

        # Read ground truth from injection.json
        with open(f"{dataset_path}/{datapack}/injection.json") as f:
            injection = json.load(f)
            ground_truth_services = injection["ground_truth"]["service"]

        # Build graph and extract features
        graph_data = build_graph(traces)
        features = extract_features(traces, graph_data)

        # Create labels (1 for root cause, 0 for others)
        labels = torch.zeros(graph_data.num_nodes)
        for service in ground_truth_services:
            if service in graph_data.service_names:
                idx = graph_data.service_names.index(service)
                labels[idx] = 1.0

        training_samples.append({
            "features": features,
            "edge_index": graph_data.edge_index,
            "labels": labels
        })

    return training_samples
```

### Training Loop

```python
def train_model(model, training_data, epochs=100):
    """Train the GNN model."""

    optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
    criterion = nn.BCELoss()

    for epoch in range(epochs):
        total_loss = 0

        for sample in training_data:
            optimizer.zero_grad()

            # Forward pass
            predictions = model(sample["features"], sample["edge_index"])

            # Calculate loss
            loss = criterion(predictions.flatten(), sample["labels"])

            # Backward pass
            loss.backward()
            optimizer.step()

            total_loss += loss.item()

        avg_loss = total_loss / len(training_data)
        print(f"Epoch {epoch+1}/{epochs}, Loss: {avg_loss:.4f}")

    return model
```

## Containerization

### Dockerfile

```dockerfile
FROM python:3.10-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy model and code
COPY model.pth ./
COPY src/ ./src/
COPY entrypoint.sh info.toml ./

RUN chmod +x entrypoint.sh

ENTRYPOINT ["./entrypoint.sh"]
```

### requirements.txt

```
polars==0.20.0
torch==2.1.0
torch-geometric==2.4.0
numpy==1.24.0
```

### entrypoint.sh

```bash
#!/bin/bash
set -e

python -m src.ml_rca \
    --trace-path "$TRACE_PATH" \
    --ground-truth-path "$GROUND_TRUTH_PATH" \
    --output-path "$OUTPUT_PATH" \
    --dataset-name "$DATASET_NAME" \
    --datapack-id "$DATAPACK_ID"
```

## Advanced Features

### Attention Mechanism

```python
class GNNWithAttention(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim):
        super(GNNWithAttention, self).__init__()
        from torch_geometric.nn import GATConv

        self.conv1 = GATConv(input_dim, hidden_dim, heads=4)
        self.conv2 = GATConv(hidden_dim * 4, hidden_dim, heads=4)
        self.conv3 = GATConv(hidden_dim * 4, output_dim, heads=1)
        self.relu = nn.ReLU()
        self.sigmoid = nn.Sigmoid()

    def forward(self, x, edge_index):
        x = self.conv1(x, edge_index)
        x = self.relu(x)

        x = self.conv2(x, edge_index)
        x = self.relu(x)

        x = self.conv3(x, edge_index)
        x = self.sigmoid(x)

        return x
```

### Temporal Features

```python
def extract_temporal_features(traces):
    """Extract time-series features."""

    # Group spans by time windows
    traces = traces.with_columns(
        (pl.col("start_time") // 60000).alias("time_window")  # 1-minute windows
    )

    # Calculate features per time window
    temporal_features = traces.group_by(["service_name", "time_window"]).agg([
        pl.col("status_code").filter(pl.col("status_code") == "ERROR").count().alias("errors"),
        pl.col("span_id").count().alias("total_spans"),
        (pl.col("end_time") - pl.col("start_time")).mean().alias("avg_latency")
    ])

    return temporal_features
```

## Evaluation

```bash
# Train model
python train.py --dataset trainticket-pandora-v1 --epochs 100

# Evaluate locally
cd rcabench-platform
./main.py eval single ml-rca trainticket-pandora-v1 0

# Submit for remote evaluation
./main.py submit-execution ml-rca trainticket-pandora-v1 0 99
```

## Performance Considerations

1. **Model Size**: Keep model small for fast inference (<100MB)
2. **Inference Time**: Target <5 seconds per datapack
3. **Memory Usage**: Limit to <2GB for standard evaluation
4. **GPU Support**: Optional, but can speed up inference

## Next Steps

- [Containerization](../../development-guide/containerization): Package your ML model
- [Remote Evaluation](../../remote-evaluation): Submit for evaluation
- [Contributing](../contributing): Share your algorithm
