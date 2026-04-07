---
title: Contributing
weight: 5
---

Thank you for your interest in contributing to the AegisLab ecosystem! This guide will help you get started.

## Ways to Contribute

### For Algorithm Developers

- **Contribute RCA algorithms** to [rca-algo-contrib](https://github.com/OperationsPAI/rca-algo-contrib)
- **Improve algorithm evaluation** metrics and frameworks
- **Add dataset converters** for new trace formats
- **Enhance documentation** with examples and tutorials

### For Dataset Creators

- **Share datasets** created through fault injection experiments
- **Document fault patterns** and their characteristics
- **Contribute fault injection handlers** for new fault types
- **Improve intelligent scheduling** algorithms

### For Infrastructure Contributors

- **Enhance AegisLab core** platform features
- **Add support for new benchmark systems** beyond TrainTicket
- **Improve observability** collection and processing
- **Optimize performance** and scalability

## Getting Started

### 1. Set Up Development Environment

```bash
# Clone the repository
git clone https://github.com/OperationsPAI/AegisLab.git
cd AegisLab

# Install dependencies
make setup-dev-env

# Run tests
make test
```

### 2. Find an Issue

Browse open issues on GitHub:
- [AegisLab Issues](https://github.com/OperationsPAI/AegisLab/issues)
- [rcabench-platform Issues](https://github.com/OperationsPAI/rcabench-platform/issues)
- [chaos-experiment Issues](https://github.com/LGU-SE-Internal/chaos-experiment/issues)

Look for issues labeled:
- `good first issue` - Good for newcomers
- `help wanted` - Community contributions welcome
- `documentation` - Documentation improvements

### 3. Create a Branch

```bash
# Create feature branch
git checkout -b feature/your-feature-name

# Or bug fix branch
git checkout -b fix/issue-description
```

## Contributing an RCA Algorithm

### 1. Implement the Algorithm

Create your algorithm in `rcabench-platform/src/rcabench_platform/v3/sdk/algorithms/`:

```python
# my_algorithm.py
from rcabench_platform.v3.sdk.algorithms.spec import Algorithm, AlgorithmArgs, AlgorithmAnswer
import polars as pl

class MyRCAAlgorithm(Algorithm):
    """
    Brief description of your algorithm.

    This algorithm uses [technique] to identify root causes by [approach].
    """

    def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
        # Read input data
        traces = pl.read_parquet(args.trace_path)

        # Your algorithm logic
        # ...

        return AlgorithmAnswer(
            ranked_services=ranked_services,
            scores=scores,
            metadata={"algorithm": "my-rca", "version": "1.0.0"}
        )
```

### 2. Register the Algorithm

Add to `rcabench-platform/src/rcabench_platform/v3/cli/main.py`:

```python
from rcabench_platform.v3.sdk.algorithms.my_algorithm import MyRCAAlgorithm

def register_builtin_algorithms():
    registry = AlgorithmRegistry.get_instance()
    # ... existing registrations
    registry.register("my-rca", MyRCAAlgorithm)
```

### 3. Test Locally

```bash
# Test on sample dataset
./main.py eval single my-rca trainticket-pandora-v1 0

# Run on multiple datapacks
./main.py eval batch my-rca trainticket-pandora-v1 0 9
```

### 4. Add Documentation

Create `docs/algorithms/my-rca.md`:

```markdown
# My RCA Algorithm

## Overview
Brief description of the algorithm.

## Approach
Detailed explanation of the technique used.

## Performance
Expected MRR, Top-k accuracy on standard datasets.

## References
- Paper: [Title](URL)
- Code: [Repository](URL)
```

### 5. Submit Pull Request

```bash
git add .
git commit -m "Add MyRCA algorithm"
git push origin feature/my-rca-algorithm
```

Create a pull request with:
- Clear description of the algorithm
- Performance metrics on test datasets
- Documentation and examples

## Contributing a Fault Type

### 1. Add Low-Level Spec

Create spec builder in `chaos-experiment/chaos/`:

```go
// my_fault.go
package chaos

func NewMyFault(params map[string]interface{}) (*MyFaultSpec, error) {
    // Build Chaos Mesh CRD spec
    return &MyFaultSpec{
        // ... spec fields
    }, nil
}
```

### 2. Add Orchestration

Create controller in `chaos-experiment/controllers/`:

```go
// my_fault_controller.go
package controllers

func ApplyMyFault(params HandlerParams) error {
    // Generate spec
    spec, err := chaos.NewMyFault(params)
    if err != nil {
        return err
    }

    // Apply to Kubernetes
    return k8sClient.Apply(spec)
}
```

### 3. Add Handler

Create handler in `chaos-experiment/handler/`:

```go
// my_fault_handler.go
package handler

func HandleMyFault(node HandlerNode) error {
    params := extractParams(node)
    return controllers.ApplyMyFault(params)
}
```

### 4. Register Handler

Add to handler registry:

```go
func init() {
    RegisterHandler("my_fault", HandleMyFault)
}
```

### 5. Add Tests

```go
// my_fault_test.go
func TestMyFault(t *testing.T) {
    params := map[string]interface{}{
        "target_service": "test-service",
        // ... test parameters
    }

    spec, err := chaos.NewMyFault(params)
    assert.NoError(t, err)
    assert.NotNil(t, spec)
}
```

## Code Style Guidelines

### Go Code

Follow standard Go conventions:

```go
// Good: Clear function names, proper error handling
func ProcessFaultInjection(task *Task) error {
    if task == nil {
        return fmt.Errorf("task cannot be nil")
    }

    chaos, err := GenerateChaos(task.HandlerNodes)
    if err != nil {
        return fmt.Errorf("failed to generate chaos: %w", err)
    }

    return ApplyChaos(chaos, task.Duration)
}
```

### Python Code

Follow PEP 8 and use type hints:

```python
# Good: Type hints, docstrings, clear logic
def analyze_traces(traces: pl.DataFrame) -> List[str]:
    """
    Analyze traces to identify suspected root causes.

    Args:
        traces: DataFrame containing trace data

    Returns:
        List of service names ranked by suspicion
    """
    error_counts = (
        traces
        .filter(pl.col("status_code") == "ERROR")
        .group_by("service_name")
        .agg(pl.count().alias("error_count"))
        .sort("error_count", descending=True)
    )

    return error_counts["service_name"].to_list()
```

### Documentation

Use clear, concise language:

```markdown
# Good: Clear structure, examples, practical guidance

## Network Delay Fault

Adds latency to network packets between services.

### Parameters

- `delay` (required): Delay duration (e.g., "100ms")
- `jitter` (optional): Random variation

### Example

\`\`\`python
{
    "handler": "network_delay",
    "params": {
        "delay": "100ms",
        "target_service": "ts-order-service"
    }
}
\`\`\`
```

## Testing Requirements

### Unit Tests

All new code must include unit tests:

```go
// Go tests
func TestGenerateChaos(t *testing.T) {
    nodes := []HandlerNode{
        {Handler: "network_delay", Params: map[string]interface{}{"delay": "100ms"}},
    }

    chaos, err := GenerateChaos(nodes)
    assert.NoError(t, err)
    assert.NotNil(t, chaos)
}
```

```python
# Python tests
def test_algorithm():
    args = AlgorithmArgs(
        trace_path="test_data/trace.parquet",
        ground_truth_path="test_data/ground_truth.parquet",
        output_path="output/",
        dataset_name="test",
        datapack_id="0",
        benchmark="test"
    )

    algo = MyRCAAlgorithm()
    result = algo(args)

    assert len(result.ranked_services) > 0
```

### Integration Tests

Test end-to-end workflows:

```bash
# Test fault injection workflow
make test-integration

# Test algorithm evaluation
./main.py eval single my-rca test-dataset 0
```

## Pull Request Process

### 1. Before Submitting

- [ ] Code follows style guidelines
- [ ] All tests pass
- [ ] Documentation updated
- [ ] CHANGELOG.md updated (if applicable)
- [ ] Commit messages are clear

### 2. Commit Messages

Use conventional commit format:

```
feat: add network bandwidth fault type
fix: correct error handling in consumer
docs: update fault injection guide
test: add tests for chaos generation
refactor: simplify handler registration
```

### 3. Pull Request Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Documentation update
- [ ] Performance improvement

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Documentation updated
- [ ] Tests pass
- [ ] CHANGELOG updated
```

### 4. Review Process

1. Automated checks run (tests, linting)
2. Maintainers review code
3. Address feedback
4. Approval and merge

## Community Guidelines

### Code of Conduct

- Be respectful and inclusive
- Welcome newcomers
- Focus on constructive feedback
- Assume good intentions

### Communication Channels

- **GitHub Issues**: Bug reports, feature requests
- **GitHub Discussions**: Questions, ideas, general discussion
- **Pull Requests**: Code contributions

### Getting Help

- Check existing documentation
- Search closed issues
- Ask in GitHub Discussions
- Tag maintainers if urgent

## Recognition

Contributors are recognized in:
- GitHub contributors page
- CHANGELOG.md for significant contributions
- Documentation acknowledgments

## License

By contributing, you agree that your contributions will be licensed under the same license as the project (typically Apache 2.0 or MIT).

## Questions?

If you have questions about contributing:
1. Check the documentation
2. Search existing issues
3. Open a new discussion on GitHub
4. Contact maintainers

Thank you for contributing to AegisLab!
