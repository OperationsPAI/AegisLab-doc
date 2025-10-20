---
title: Contributing
date: 2025-09-26
weight: 12
---

## 📋 Code of Conduct

By participating in this project, you agree to abide by our Code of Conduct:

- **Be respectful**: Treat all contributors with respect and professionalism
- **Be inclusive**: Welcome contributors from all backgrounds and skill levels
- **Be collaborative**: Work together constructively to improve the project
- **Be patient**: Help others learn and grow in their contributions
- **Be constructive**: Provide helpful feedback and suggestions

---

## 📝 Contribution Types

### 1. Bug Fixes

- Fix identified bugs in the codebase
- Improve error handling and edge cases
- Resolve performance issues

### 2. Feature Development

- Implement new RCA algorithms
- Add new API endpoints
- Enhance user interface
- Improve observability and monitoring

### 3. Documentation

- Improve existing documentation
- Add new tutorials and examples
- Update API documentation
- Create algorithm documentation

### 4. Testing

- Add unit tests for existing code
- Create integration tests
- Develop performance benchmarks
- Improve test coverage

### 5. Infrastructure

- Improve deployment processes
- Enhance CI/CD pipelines
- Optimize containerization
- Update dependencies

---

## 🔄 Development Workflow

### Branch Naming Convention

Use descriptive branch names with prefixes:

- `feature/` - New features
- `bugfix/` - Bug fixes
- `docs/` - Documentation updates
- `test/` - Testing improvements
- `refactor/` - Code refactoring
- `chore/` - Maintenance tasks

> [!EXAMPLE]
>
> - `feature/add-statistical-rca-algorithm`
> - `bugfix/fix-memory-leak-in-processing`
> - `docs/update-installation-guide`

### Commit Message Format

Use conventional commit format:

```plain
<type>(<scope>): <description>

[optional body]

[optional footer]
```

Types:

- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Maintenance tasks

> [!EXAMPLE]
>
> ```
> feat(algorithm): add random walk RCA algorithm
>
> Implement random walk based root cause analysis algorithm
> with configurable parameters for step count and threshold.
>
> Closes #123
> ```
>
> ```
> fix(api): resolve memory leak in data processing
>
> Fix memory leak in metrics processing that occurred when
> handling large datasets. Added proper cleanup of temporary
> data structures.
>
> Fixes #456
> ```

### Development Process

1. **Sync with upstream**:

   ```bash
   git fetch upstream
   git checkout main
   git merge upstream/main
   ```

2. **Create feature branch**:

   ```bash
   git checkout -b feature/your-feature
   ```

3. **Make changes**:

   - Write code following project conventions
   - Add tests for new functionality
   - Update documentation as needed

4. **Test changes**:

   ```bash
   # Run all tests
   make test

   # Run specific test suites
   go test ./src/handlers/
   python -m pytest sdk/python/tests/
   ```

5. **Commit changes**:

   ```bash
   git add .
   git commit -m "feat(scope): description"
   ```

6. **Push to your fork**:

   ```bash
   git push origin feature/your-feature
   ```

7. **Create pull request** on GitHub

---

## 📏 Code Guidelines

### Go Code Style

Follow standard Go conventions:

```go
// Package documentation
package handlers

import (
    "context"
    "encoding/json"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/your-org/rcabench/dto"
)

// AlgorithmHandler handles algorithm-related requests
type AlgorithmHandler struct {
    service AlgorithmService
    logger  Logger
}

// ListAlgorithms returns all available algorithms
func (h *AlgorithmHandler) ListAlgorithms(c *gin.Context) {
    algorithms, err := h.service.GetAlgorithms(c.Request.Context())
    if err != nil {
        h.logger.Error("Failed to get algorithms", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Internal server error"})
        return
    }

    c.JSON(http.StatusOK, dto.AlgorithmListResponse{
        Success: true,
        Data:    algorithms,
    })
}
```

> [!NOTE]
>
> - Use `gofmt` for formatting
> - Follow effective Go principles
> - Use meaningful variable names
> - Add comments for public functions
> - Handle errors appropriately
> - Use structured logging

### Python Code Style

Development Tools Setup

```bash
# Install Ruff
pip install ruff

# Run linting
ruff check .

# Run formatting
ruff format .

# Run both linting and formatting with fixes
ruff check --fix . && ruff format .
```

> [!TIP]
>
> **VSCode Users**: Install the [Ruff extension](https://marketplace.visualstudio.com/items?itemName=charliermarsh.ruff) for real-time linting and formatting support. Add this to your VSCode settings:
>
> ```
> {
>   "ruff.lint.enable": true,
>   "ruff.format.enable": true,
>   "[python]": {
>     "editor.formatOnSave": true,
>     "editor.codeActionsOnSave": {
>       "source.fixAll.ruff": true
>     }
>   }
> }
> ```

Follow **Ruff** formatting and linting standards:

```python
"""Algorithm execution module.

This module provides functionality for executing RCA algorithms
and managing their lifecycle.
"""

from __future__ import annotations

import asyncio
import logging
from typing import Any

class AlgorithmExecutor:
    """Handles execution of RCA algorithms."""

    def __init__(self, config: dict[str, Any]) -> None:
        """Initialize algorithm executor.

        Args:
            config: Configuration dictionary containing execution parameters
        """
        self.config = config
        self.logger = logging.getLogger(__name__)
        self._running_tasks: dict[str, asyncio.Task[Any]] = {}

    async def execute_algorithm(
        self,
        algorithm_name: str,
        dataset_path: str,
        parameters: dict[str, Any] | None = None,
    ) -> str:
        """Execute an RCA algorithm asynchronously.

        Args:
            algorithm_name: Name of the algorithm to execute
            dataset_path: Path to the dataset
            parameters: Optional algorithm parameters

        Returns:
            Task ID for tracking execution

        Raises:
            ValueError: If algorithm name is invalid
            FileNotFoundError: If dataset path doesn't exist
        """
        if not algorithm_name:
            msg = "Algorithm name cannot be empty"
            raise ValueError(msg)

        task_id = self._generate_task_id()
        task = asyncio.create_task(
            self._run_algorithm(algorithm_name, dataset_path, parameters)
        )
        self._running_tasks[task_id] = task

        self.logger.info(
            "Started algorithm execution",
            extra={
                "task_id": task_id,
                "algorithm": algorithm_name,
                "dataset": dataset_path,
            },
        )

        return task_id
```

> [!NOTE]
>
> - Use **Ruff** for both linting and formatting (replaces Black, >isort, flake8)
> - Use modern Python type hints (`dict[str, Any]` instead of `Dict>[str, Any]`)
> - Follow PEP 585 (use built-in generics)
> - Use `from __future__ import annotations` for forward compatibility
> - Use descriptive error messages with variables
> - Handle exceptions with proper error messages
> - Use structured logging with extra context

---

## 🚨 Error Handling Guidelines

### Go Error Handling

```go
func (s *AlgorithmService) GetAlgorithm(ctx context.Context, id string) (*Algorithm, error) {
    if id == "" {
        return nil, NewValidationError("algorithm ID cannot be empty")
    }

    algorithm, err := s.repository.FindByID(ctx, id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            return nil, NewNotFoundError("algorithm not found")
        }
        return nil, fmt.Errorf("failed to get algorithm: %w", err)
    }

    return algorithm, nil
}
```

### Python Error Handling

```python
async def get_algorithm_results(self, task_id: str) -> dict[str, Any]:
    """Get algorithm execution results."""
    try:
        if not task_id:
            msg = "Task ID cannot be empty"
            raise ValueError(msg)

        results = await self.storage.get_results(task_id)
        if not results:
            msg = f"Results not found for task {task_id}"
            raise NotFoundError(msg)

        return results

    except ValidationError:
        raise
    except StorageError as e:
        self.logger.error("Storage error retrieving results: %s", e)
        msg = "Failed to retrieve results"
        raise InternalError(msg) from e
    except Exception as e:
        self.logger.error("Unexpected error: %s", e)
        msg = "Internal server error"
        raise InternalError(msg) from e
```

---

## 🧪 Testing Guidelines

### Go Tests

```go
func TestAlgorithmHandler_ListAlgorithms(t *testing.T) {
    tests := []struct {
        name           string
        mockAlgorithms []Algorithm
        mockError      error
        expectedStatus int
        expectedCount  int
    }{
        {
            name: "successful retrieval",
            mockAlgorithms: []Algorithm{
                {ID: "1", Name: "Algorithm 1"},
                {ID: "2", Name: "Algorithm 2"},
            },
            expectedStatus: http.StatusOK,
            expectedCount:  2,
        },
        {
            name:           "service error",
            mockError:      errors.New("service error"),
            expectedStatus: http.StatusInternalServerError,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            mockService := &MockAlgorithmService{}
            mockService.On("GetAlgorithms", mock.Anything).Return(tt.mockAlgorithms, tt.mockError)

            handler := &AlgorithmHandler{service: mockService}

            // Execute
            w := httptest.NewRecorder()
            c, _ := gin.CreateTestContext(w)
            handler.ListAlgorithms(c)

            // Assert
            assert.Equal(t, tt.expectedStatus, w.Code)

            if tt.expectedStatus == http.StatusOK {
                var response dto.AlgorithmListResponse
                err := json.Unmarshal(w.Body.Bytes(), &response)
                assert.NoError(t, err)
                assert.Len(t, response.Data, tt.expectedCount)
            }
        })
    }
}
```

### Python Tests

```python
import pytest
from unittest.mock import AsyncMock

from rcabench.execution import AlgorithmExecutor


class TestAlgorithmExecutor:
    @pytest.fixture
    def executor(self) -> AlgorithmExecutor:
        config = {"timeout": 3600, "max_concurrent": 5}
        return AlgorithmExecutor(config)

    @pytest.mark.asyncio
    async def test_execute_algorithm_success(
        self, executor: AlgorithmExecutor
    ) -> None:
        """Test successful algorithm execution."""
        # Setup
        algorithm_name = "test-algorithm"
        dataset_path = "/test/data"

        # Execute
        task_id = await executor.execute_algorithm(algorithm_name, dataset_path)

        # Assert
        assert task_id is not None
        assert task_id in executor._running_tasks

    @pytest.mark.asyncio
    async def test_execute_algorithm_invalid_name(
        self, executor: AlgorithmExecutor
    ) -> None:
        """Test algorithm execution with invalid name."""
        with pytest.raises(ValueError, match="Algorithm name cannot be empty"):
            await executor.execute_algorithm("", "/test/data")

    @pytest.mark.parametrize(
        ("algorithm_name", "dataset_path", "expected_error"),
        [
            ("", "/test/data", ValueError),
            ("test-algo", "", ValueError),
            (None, "/test/data", TypeError),
        ],
    )
    @pytest.mark.asyncio
    async def test_execute_algorithm_validation(
        self,
        executor: AlgorithmExecutor,
        algorithm_name: str | None,
        dataset_path: str,
        expected_error: type[Exception],
    ) -> None:
        """Test algorithm execution validation."""
        with pytest.raises(expected_error):
            await executor.execute_algorithm(algorithm_name, dataset_path)
```

---

## 📚 Documentation Guidelines

### Code Documentation

#### Go Documentation

```go
// Package algorithms provides RCA algorithm management functionality.
//
// This package includes interfaces and implementations for registering,
// executing, and managing root cause analysis algorithms within RCABench.
package algorithms

// Algorithm represents an RCA algorithm with its metadata and configuration.
type Algorithm struct {
    // ID is the unique identifier for the algorithm
    ID string `json:"id" db:"id"`

    // Name is the human-readable name of the algorithm
    Name string `json:"name" db:"name" validate:"required,min=1,max=100"`

    // Version is the semantic version of the algorithm
    Version string `json:"version" db:"version" validate:"required,semver"`

    // Image is the Docker image reference for the algorithm
    Image string `json:"image" db:"image" validate:"required"`

    // CreatedAt is the timestamp when the algorithm was registered
    CreatedAt time.Time `json:"created_at" db:"created_at"`
}

// ExecuteOptions contains parameters for algorithm execution.
type ExecuteOptions struct {
    // Timeout specifies the maximum execution time in seconds
    Timeout int `json:"timeout,omitempty"`

    // Parameters contains algorithm-specific parameters
    Parameters map[string]interface{} `json:"parameters,omitempty"`

    // Resources specifies resource limits for execution
    Resources *ResourceLimits `json:"resources,omitempty"`
}
```

#### Python Documentation

```python
class RCABenchSDK:
    """Python SDK for interacting with RCABench platform.

    This SDK provides a convenient interface for managing algorithms,
    executing fault injections, and analyzing results in RCABench.

    Example:
        >>> sdk = RCABenchSDK("http://localhost:8082")
        >>> algorithms = sdk.algorithm.list()
        >>> print(f"Found {len(algorithms)} algorithms")

    Attributes:
        base_url: The base URL of the RCABench API
        timeout: Default timeout for API requests in seconds
        algorithm: Interface for algorithm operations
        injection: Interface for fault injection operations
        dataset: Interface for dataset operations
    """

    def __init__(self, base_url: str, api_key: Optional[str] = None, timeout: int = 30):
        """Initialize the RCABench SDK.

        Args:
            base_url: Base URL of the RCABench API (e.g., "http://localhost:8082")
            api_key: Optional API key for authentication
            timeout: Default timeout for API requests in seconds

        Raises:
            ValueError: If base_url is empty or invalid
            ConnectionError: If unable to connect to the API
        """
```

### API Documentation

Use OpenAPI/Swagger annotations:

```go
// ListAlgorithms godoc
// @Summary List all available algorithms
// @Description Get a paginated list of all registered RCA algorithms
// @Tags algorithms
// @Accept json
// @Produce json
// @Param page query int false "Page number" default(1)
// @Param limit query int false "Items per page" default(20)
// @Param category query string false "Filter by algorithm category"
// @Success 200 {object} dto.AlgorithmListResponse
// @Failure 400 {object} dto.ErrorResponse
// @Failure 500 {object} dto.ErrorResponse
// @Router /algorithms [get]
func (h *AlgorithmHandler) ListAlgorithms(c *gin.Context) {
    // Implementation
}
```

---

## 🔀 Pull Request Process

### Before Submitting

1. **Ensure all tests pass**:

   ```bash
   make test
   ```

2. **Run linting and formatting**:

   ```bash
   make lint
   make format
   ```

3. **Update documentation** if needed

4. **Add changelog entry** if applicable

### Pull Request Template

```md
## Description

Brief description of the changes made.

## Type of Change

- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## How Has This Been Tested?

Describe the tests you ran and/or added.

## Checklist

- [ ] Code follows project style guidelines
- [ ] Self-review of code completed
- [ ] Code comments added where necessary
- [ ] Documentation updated
- [ ] Tests added/updated and passing
- [ ] No breaking changes or breaking changes documented
```

### Review Process

> [!IMPORTANT]
>
> 1.  **Automated checks** must pass (CI/CD pipeline)
> 2.  **Code review** by at least one maintainer
> 3.  **Documentation review** if applicable
> 4.  **Testing verification** in staging environment
> 5.  **Final approval** and merge

---

## 🐛 Issue Reporting

### Bug Reports

Use the bug report template:

```md
**Describe the bug**
A clear and concise description of what the bug is.

**To Reproduce**
Steps to reproduce the behavior:

1. Go to '...'
2. Click on '....'
3. See error

**Expected behavior**
A clear and concise description of what you expected to happen.

**Screenshots**
If applicable, add screenshots to help explain your problem.

**Environment (please complete the following information):**

- OS: [e.g. Ubuntu 20.04]
- Kubernetes version: [e.g. 1.25.0]
- RCABench version: [e.g. 1.0.1]
- Browser (if applicable): [e.g. Chrome 91.0]

**Additional context**
Add any other context about the problem here.
```

### Feature Requests

Use the feature request template:

```md
**Is your feature request related to a problem? Please describe.**
A clear and concise description of what the problem is.

**Describe the solution you'd like**
A clear and concise description of what you want to happen.

**Describe alternatives you've considered**
A clear and concise description of any alternative solutions or features you've considered.

**Additional context**
Add any other context or screenshots about the feature request here.
```

---

## 🏆 Recognition

Contributors will be recognized in:

- Project README
- Release notes
- Contributor documentation
- Annual contributor recognition

Thank you for contributing to RCABench!
