---
title: Contributing
weight: 3
---

Share your RCA algorithm with the community through the rca-algo-contrib repository.

## Overview

The [rca-algo-contrib](https://github.com/OperationsPAI/rca-algo-contrib) repository is a collection of community-contributed RCA algorithms. By contributing your algorithm, you help:
- Advance RCA research
- Provide baselines for comparison
- Share knowledge with the community
- Get feedback on your approach

## Prerequisites

- Algorithm tested locally with good results
- Docker image built and working
- Documentation written
- Code follows best practices

## Contribution Process

### 1. Fork the Repository

```bash
git clone https://github.com/OperationsPAI/rca-algo-contrib.git
cd rca-algo-contrib
git checkout -b add-my-rca-algorithm
```

### 2. Add Your Algorithm

Create a directory for your algorithm:

```bash
mkdir algorithms/my-rca
cd algorithms/my-rca
```

Required files:
- `src/`: Algorithm source code
- `Dockerfile`: Container definition
- `entrypoint.sh`: Entry point script
- `info.toml`: Algorithm metadata
- `README.md`: Documentation
- `requirements.txt`: Python dependencies

### 3. Create info.toml

```toml
[algorithm]
name = "my-rca"
version = "1.0.0"
description = "Brief description of your algorithm"
authors = ["Your Name <your.email@example.com>"]
license = "MIT"

[algorithm.metadata]
type = "ml-based"  # or "heuristic", "graph-based", etc.
requires_training = true
supports_streaming = false

[algorithm.performance]
avg_inference_time_ms = 5000
memory_usage_mb = 2048
recommended_cpu = "2000m"
recommended_memory = "4Gi"

[algorithm.datasets]
tested_on = ["trainticket-pandora-v1", "otel-demo-v1"]
best_performance = "trainticket-pandora-v1"
```

### 4. Write Documentation

Create a comprehensive README.md:

```markdown
# My RCA Algorithm

## Overview

Brief description of your algorithm and its approach.

## Algorithm Details

### Approach

Explain the core methodology:
- What features does it use?
- What techniques does it employ?
- What makes it unique?

### Architecture

Describe the algorithm architecture with diagrams if helpful.

## Performance

### Evaluation Results

| Dataset | MRR | Top-1 | Top-3 |
|---------|-----|-------|-------|
| TrainTicket | 0.856 | 0.745 | 0.912 |
| OTEL Demo | 0.823 | 0.712 | 0.887 |

### Resource Usage

- CPU: 2000m
- Memory: 4Gi
- Inference time: ~5s per datapack

## Usage

### Local Evaluation

```bash
cd rcabench-platform
./main.py eval single my-rca trainticket-pandora-v1 0
```

### Remote Evaluation

```bash
./main.py submit-execution my-rca trainticket-pandora-v1 0 99
```

## Implementation Details

Key implementation details, design decisions, and trade-offs.

## References

- Paper: [Link to paper if applicable]
- Code: [Link to original implementation if applicable]

## License

MIT License
```

### 5. Test Your Contribution

Before submitting, verify:

```bash
# Build Docker image
docker build -t my-rca:test .

# Test locally
cd ../../rcabench-platform
./main.py eval single my-rca trainticket-pandora-v1 0

# Verify output format
cat output/my-rca/0/output.json
```

### 6. Submit Pull Request

```bash
git add algorithms/my-rca/
git commit -m "Add my-rca algorithm"
git push origin add-my-rca-algorithm
```

Create a pull request on GitHub with:
- Clear title: "Add [algorithm-name] algorithm"
- Description of the algorithm
- Performance results
- Any special requirements

## Code Quality Guidelines

### Code Style

- Follow PEP 8 for Python code
- Use type hints where appropriate
- Add docstrings for public functions
- Keep functions focused and small

### Error Handling

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    try:
        traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")
    except Exception as e:
        print(f"Error loading traces: {e}")
        return []

    # Algorithm logic with proper error handling
    try:
        result = self.analyze(traces)
    except Exception as e:
        print(f"Error during analysis: {e}")
        return []

    return result
```

### Performance

- Use lazy evaluation for large datasets
- Avoid unnecessary data copies
- Profile your code before submission
- Document performance characteristics

### Testing

Include unit tests:

```python
# tests/test_my_rca.py
import pytest
from my_rca import MyRCA

def test_basic_functionality():
    algo = MyRCA()
    # Test with minimal data
    result = algo(test_args)
    assert len(result) > 0
    assert all(isinstance(ans, AlgorithmAnswer) for ans in result)

def test_error_handling():
    algo = MyRCA()
    # Test with invalid data
    result = algo(invalid_args)
    assert result == []
```

## Review Process

Your pull request will be reviewed for:

1. **Correctness**: Does the algorithm work as described?
2. **Performance**: Are the performance claims accurate?
3. **Code quality**: Is the code well-written and maintainable?
4. **Documentation**: Is the documentation clear and complete?
5. **Reproducibility**: Can others reproduce your results?

## After Acceptance

Once your PR is merged:

1. Your algorithm will be available in the registry
2. It will be included in benchmark comparisons
3. You'll be credited in the contributors list
4. Others can evaluate and build upon your work

## Getting Help

- Open an issue in the repository
- Join the community Discord
- Email the maintainers

## Example Contributions

Study these examples for reference:
- `algorithms/simplerca/`: Basic heuristic algorithm
- `algorithms/ml-rca/`: ML-based approach
- `algorithms/graph-rca/`: Graph-based analysis

## License

By contributing, you agree to license your code under the repository's license (typically MIT or Apache 2.0).

## Recognition

Contributors are recognized in:
- Repository README
- Research papers using the algorithms
- Community acknowledgments

Thank you for contributing to the RCA research community!
