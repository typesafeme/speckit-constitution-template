# Python Dask Project Constitution

## Project Philosophy

This project follows the principles of clean code, modularity, and best practices observed in successful open-source Python projects like pandas, scikit-learn, dask, and others. Code should be self-documenting, testable, and maintainable.

## Core Principles

1. **Explicit is better than implicit** - Follow the Zen of Python
2. **Modularity first** - Small, focused, reusable components
3. **Type safety** - Comprehensive type hints for all public APIs
4. **Test-driven** - Write tests before or alongside implementation
5. **Documentation** - Clear docstrings and usage examples
6. **Performance-aware** - Profile before optimizing, use Dask efficiently

---

## Project Structure

```
project_root/
├── src/
│   └── project_name/
│       ├── __init__.py
│       ├── core/              # Core business logic
│       │   ├── __init__.py
│       │   ├── engine.py
│       │   └── models.py
│       ├── data/              # Data handling modules
│       │   ├── __init__.py
│       │   ├── loaders.py
│       │   ├── processors.py
│       │   └── validators.py
│       ├── compute/           # Dask computation modules
│       │   ├── __init__.py
│       │   ├── cluster.py
│       │   ├── distributed.py
│       │   └── pipelines.py
│       ├── utils/             # Utility functions
│       │   ├── __init__.py
│       │   ├── logging.py
│       │   └── config.py
│       └── cli/               # Command-line interface
│           ├── __init__.py
│           └── commands.py
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── fixtures/
│   └── conftest.py
├── docs/
│   ├── api/
│   ├── guides/
│   └── examples/
├── scripts/                   # Deployment/utility scripts
├── configs/                   # Configuration files
├── pyproject.toml
├── README.md
├── CONTRIBUTING.md
└── LICENSE
```

---

## Code Style and Standards

### Python Version
- **Minimum**: Python 3.9+
- **Target**: Python 3.11+ for optimal Dask performance

### Code Formatting
```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
]

[tool.black]
line-length = 100
target-version = ['py311']
```

### Naming Conventions
- **Modules**: `lowercase_with_underscores.py`
- **Classes**: `PascalCase`
- **Functions/Methods**: `lowercase_with_underscores()`
- **Constants**: `UPPERCASE_WITH_UNDERSCORES`
- **Private**: Prefix with single underscore `_private_function()`
- **Internal**: Prefix with double underscore for name mangling `__internal`

### Type Hints

**REQUIRED** for all public functions, methods, and class attributes:

```python
from typing import Optional, Union, Literal
import dask.dataframe as dd
import pandas as pd

def process_dataframe(
    df: dd.DataFrame,
    columns: list[str],
    operation: Literal["mean", "sum", "count"],
    parallel: bool = True,
) -> dd.DataFrame:
    """
    Process a Dask DataFrame with specified operation.
    
    Args:
        df: Input Dask DataFrame
        columns: List of column names to process
        operation: Type of aggregation operation
        parallel: Whether to use parallel computation
        
    Returns:
        Processed Dask DataFrame
        
    Raises:
        ValueError: If columns are not found in DataFrame
        
    Examples:
        >>> df = dd.from_pandas(pd.DataFrame({'a': [1, 2, 3]}), npartitions=2)
        >>> result = process_dataframe(df, ['a'], 'mean')
    """
    ...
```

---

## Dask Best Practices

### 1. DataFrame Operations

**DO:**
```python
import dask.dataframe as dd

# Use lazy evaluation effectively
df = dd.read_parquet('data/*.parquet')
result = (
    df
    .query('value > 100')
    .groupby('category')
    .agg({'value': ['mean', 'sum']})
    .compute()  # Only compute at the end
)

# Set proper index for efficient operations
df = df.set_index('timestamp', sorted=True)
```

**DON'T:**
```python
# Avoid multiple compute() calls
df = dd.read_parquet('data/*.parquet').compute()  # Too early!
result = df.groupby('category').mean()  # Now it's just pandas

# Avoid iterating rows
for row in df.iterrows():  # NEVER do this with Dask
    process(row)
```

### 2. Partitioning Strategy

```python
from dask.distributed import Client

def create_optimized_dataframe(
    data_path: str,
    partition_size: str = "100MB",
) -> dd.DataFrame:
    """
    Create a well-partitioned Dask DataFrame.
    
    Rules:
    - 100-500 MB per partition is optimal
    - Partition on frequently filtered columns
    - Use sorted=True when index is sorted
    """
    df = dd.read_parquet(
        data_path,
        blocksize=partition_size,
    )
    
    # Repartition if needed
    if df.npartitions > 1000:
        df = df.repartition(npartitions=500)
    
    return df
```

### 3. Client Configuration

```python
from dask.distributed import Client, LocalCluster
from typing import Optional

class DaskClientManager:
    """Manages Dask client lifecycle."""
    
    def __init__(
        self,
        n_workers: Optional[int] = None,
        threads_per_worker: int = 2,
        memory_limit: str = "4GB",
    ):
        self.n_workers = n_workers
        self.threads_per_worker = threads_per_worker
        self.memory_limit = memory_limit
        self._client: Optional[Client] = None
    
    def __enter__(self) -> Client:
        """Start Dask client with optimized settings."""
        cluster = LocalCluster(
            n_workers=self.n_workers,
            threads_per_worker=self.threads_per_worker,
            memory_limit=self.memory_limit,
        )
        self._client = Client(cluster)
        return self._client
    
    def __exit__(self, *args) -> None:
        """Clean up client resources."""
        if self._client:
            self._client.close()
```

### 4. Memory Management

```python
import dask
from dask.diagnostics import ProgressBar

# Configure for memory efficiency
dask.config.set({
    'distributed.worker.memory.target': 0.75,    # Target 75% memory usage
    'distributed.worker.memory.spill': 0.85,     # Spill to disk at 85%
    'distributed.worker.memory.pause': 0.95,     # Pause at 95%
    'distributed.worker.memory.terminate': 0.98, # Terminate at 98%
})

# Use context managers for progress tracking
with ProgressBar():
    result = large_computation.compute()
```

---

## Module Design Patterns

### 1. Data Loaders

```python
from abc import ABC, abstractmethod
from pathlib import Path
import dask.dataframe as dd

class DataLoader(ABC):
    """Abstract base class for data loaders."""
    
    @abstractmethod
    def load(self, path: Path) -> dd.DataFrame:
        """Load data from specified path."""
        pass
    
    @abstractmethod
    def validate(self, df: dd.DataFrame) -> bool:
        """Validate loaded data."""
        pass

class ParquetLoader(DataLoader):
    """Load data from Parquet files."""
    
    def __init__(self, columns: Optional[list[str]] = None):
        self.columns = columns
    
    def load(self, path: Path) -> dd.DataFrame:
        """Load Parquet files with optional column selection."""
        return dd.read_parquet(
            path,
            columns=self.columns,
            engine='pyarrow',
        )
    
    def validate(self, df: dd.DataFrame) -> bool:
        """Validate DataFrame structure."""
        if self.columns:
            return all(col in df.columns for col in self.columns)
        return True
```

### 2. Data Processors

```python
from typing import Protocol

class DataProcessor(Protocol):
    """Protocol for data processing classes."""
    
    def process(self, df: dd.DataFrame) -> dd.DataFrame:
        """Process DataFrame and return result."""
        ...

class ChainedProcessor:
    """Chain multiple processors together."""
    
    def __init__(self, processors: list[DataProcessor]):
        self.processors = processors
    
    def process(self, df: dd.DataFrame) -> dd.DataFrame:
        """Apply all processors in sequence."""
        result = df
        for processor in self.processors:
            result = processor.process(result)
        return result
```

### 3. Configuration Management

```python
from dataclasses import dataclass, field
from pathlib import Path
import yaml

@dataclass
class DaskConfig:
    """Configuration for Dask operations."""
    
    n_workers: int = 4
    threads_per_worker: int = 2
    memory_limit: str = "4GB"
    scheduler: str = "threads"
    
    @classmethod
    def from_yaml(cls, path: Path) -> "DaskConfig":
        """Load configuration from YAML file."""
        with open(path) as f:
            data = yaml.safe_load(f)
        return cls(**data.get('dask', {}))

@dataclass
class AppConfig:
    """Main application configuration."""
    
    data_path: Path
    output_path: Path
    dask: DaskConfig = field(default_factory=DaskConfig)
    log_level: str = "INFO"
    
    @classmethod
    def load(cls, path: Path) -> "AppConfig":
        """Load complete configuration."""
        with open(path) as f:
            data = yaml.safe_load(f)
        
        return cls(
            data_path=Path(data['data_path']),
            output_path=Path(data['output_path']),
            dask=DaskConfig(**data.get('dask', {})),
            log_level=data.get('log_level', 'INFO'),
        )
```

---

## Testing Standards

### Test Structure

```python
import pytest
import pandas as pd
import dask.dataframe as dd
from project_name.data.processors import DataProcessor

@pytest.fixture
def sample_dataframe() -> dd.DataFrame:
    """Create a sample Dask DataFrame for testing."""
    pdf = pd.DataFrame({
        'id': range(100),
        'value': range(100, 200),
        'category': ['A', 'B'] * 50,
    })
    return dd.from_pandas(pdf, npartitions=4)

class TestDataProcessor:
    """Tests for DataProcessor class."""
    
    def test_process_basic(self, sample_dataframe):
        """Test basic processing operation."""
        processor = DataProcessor()
        result = processor.process(sample_dataframe).compute()
        
        assert len(result) == 100
        assert 'id' in result.columns
    
    def test_process_empty_dataframe(self):
        """Test handling of empty DataFrame."""
        empty_df = dd.from_pandas(pd.DataFrame(), npartitions=1)
        processor = DataProcessor()
        
        with pytest.raises(ValueError, match="Empty DataFrame"):
            processor.process(empty_df)
    
    @pytest.mark.parametrize("npartitions", [1, 2, 4, 8])
    def test_process_various_partitions(self, sample_dataframe, npartitions):
        """Test processing with different partition counts."""
        df = sample_dataframe.repartition(npartitions=npartitions)
        processor = DataProcessor()
        result = processor.process(df).compute()
        
        assert len(result) == 100
```

### Test Coverage Requirements

- **Unit tests**: Minimum 85% coverage
- **Integration tests**: Critical paths must be covered
- **Property-based tests**: Use `hypothesis` for complex logic

```python
from hypothesis import given, strategies as st
import hypothesis.extra.pandas as pdst

@given(
    df=pdst.data_frames(
        columns=[
            pdst.column('value', dtype=int),
            pdst.column('category', dtype=str),
        ],
        rows=st.integers(min_value=10, max_value=1000),
    )
)
def test_processor_properties(df):
    """Property-based test for processor."""
    ddf = dd.from_pandas(df, npartitions=4)
    processor = DataProcessor()
    result = processor.process(ddf).compute()
    
    # Properties that should always hold
    assert len(result) <= len(df)
    assert result['value'].dtype == df['value'].dtype
```

---

## Documentation Standards

### Docstring Format (Google Style)

```python
def compute_rolling_stats(
    df: dd.DataFrame,
    window: int,
    columns: list[str],
    min_periods: Optional[int] = None,
) -> dd.DataFrame:
    """
    Compute rolling statistics on specified columns.
    
    This function calculates rolling mean and standard deviation for
    the specified columns using a sliding window approach.
    
    Args:
        df: Input Dask DataFrame with time series data
        window: Size of the rolling window in number of rows
        columns: List of column names to compute statistics for
        min_periods: Minimum number of observations required. If None,
            defaults to window size.
    
    Returns:
        Dask DataFrame with original columns plus computed statistics.
        New columns are named as '{column}_mean' and '{column}_std'.
    
    Raises:
        ValueError: If columns are not present in DataFrame
        ValueError: If window is less than 1
    
    Examples:
        >>> import pandas as pd
        >>> import dask.dataframe as dd
        >>> df = dd.from_pandas(
        ...     pd.DataFrame({'value': [1, 2, 3, 4, 5]}),
        ...     npartitions=2
        ... )
        >>> result = compute_rolling_stats(df, window=3, columns=['value'])
        >>> print(result.compute())
           value  value_mean  value_std
        0      1         NaN        NaN
        1      2         NaN        NaN
        2      3         2.0   1.000000
        3      4         3.0   1.000000
        4      5         4.0   1.000000
    
    Note:
        For optimal performance, ensure your DataFrame is properly
        partitioned and indexed before calling this function.
    
    See Also:
        pandas.DataFrame.rolling: The underlying pandas operation
    """
    if min_periods is None:
        min_periods = window
    
    # Implementation here
    ...
```

### README Structure

Every module should have clear README documentation:

```markdown
# Module Name

Brief description of what this module does.

## Features

- Feature 1
- Feature 2
- Feature 3

## Quick Start

\`\`\`python
from project_name.module import MainClass

# Basic usage example
obj = MainClass()
result = obj.process(data)
\`\`\`

## API Reference

See [API documentation](../docs/api/module.md) for detailed information.

## Performance Considerations

- Important performance tips
- Memory usage guidelines
- Scalability notes
```

---

## Error Handling

### Custom Exceptions

```python
class ProjectBaseException(Exception):
    """Base exception for all project exceptions."""
    pass

class DataValidationError(ProjectBaseException):
    """Raised when data validation fails."""
    pass

class ConfigurationError(ProjectBaseException):
    """Raised when configuration is invalid."""
    pass

class ComputationError(ProjectBaseException):
    """Raised when Dask computation fails."""
    
    def __init__(self, message: str, partition_info: Optional[dict] = None):
        super().__init__(message)
        self.partition_info = partition_info
```

### Error Handling Patterns

```python
import logging
from typing import Optional

logger = logging.getLogger(__name__)

def safe_compute(
    df: dd.DataFrame,
    timeout: Optional[int] = None,
) -> Optional[pd.DataFrame]:
    """
    Safely compute Dask DataFrame with error handling.
    
    Args:
        df: Dask DataFrame to compute
        timeout: Optional timeout in seconds
        
    Returns:
        Computed pandas DataFrame, or None if computation fails
    """
    try:
        if timeout:
            result = df.compute(scheduler='threads', timeout=timeout)
        else:
            result = df.compute()
        return result
        
    except MemoryError as e:
        logger.error("Out of memory during computation: %s", e)
        logger.info("Try reducing partition size or enabling spilling")
        raise ComputationError("Insufficient memory") from e
        
    except Exception as e:
        logger.exception("Unexpected error during computation")
        raise ComputationError(f"Computation failed: {e}") from e
```

---

## Logging Standards

```python
import logging
import sys
from pathlib import Path

def setup_logging(
    level: str = "INFO",
    log_file: Optional[Path] = None,
) -> None:
    """
    Configure project-wide logging.
    
    Args:
        level: Logging level (DEBUG, INFO, WARNING, ERROR)
        log_file: Optional path to log file
    """
    handlers: list[logging.Handler] = [
        logging.StreamHandler(sys.stdout)
    ]
    
    if log_file:
        handlers.append(
            logging.FileHandler(log_file)
        )
    
    logging.basicConfig(
        level=getattr(logging, level.upper()),
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=handlers,
    )
    
    # Set Dask logging to WARNING to reduce noise
    logging.getLogger('dask').setLevel(logging.WARNING)
    logging.getLogger('distributed').setLevel(logging.WARNING)

# Usage in each module
logger = logging.getLogger(__name__)
```

---

## Dependency Management

### pyproject.toml

```toml
[build-system]
requires = ["setuptools>=65.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "project-name"
version = "0.1.0"
description = "A well-architected Dask project"
authors = [{name = "Your Name", email = "you@example.com"}]
readme = "README.md"
requires-python = ">=3.9"
license = {text = "MIT"}
keywords = ["dask", "dataframes", "distributed-computing"]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
]

dependencies = [
    "dask[complete]>=2024.1.0",
    "pandas>=2.0.0",
    "pyarrow>=14.0.0",
    "pyyaml>=6.0",
    "click>=8.1.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "pytest-xdist>=3.3.0",
    "hypothesis>=6.82.0",
    "ruff>=0.1.0",
    "black>=23.0.0",
    "mypy>=1.5.0",
    "pre-commit>=3.3.0",
]

docs = [
    "sphinx>=7.0.0",
    "sphinx-rtd-theme>=1.3.0",
    "myst-parser>=2.0.0",
]

[project.scripts]
project-cli = "project_name.cli:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = "--cov=src --cov-report=html --cov-report=term-missing"

[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
```

---

## Performance Guidelines

### 1. Profiling

```python
import cProfile
import pstats
from functools import wraps
import time

def profile_function(func):
    """Decorator to profile function execution."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        
        result = func(*args, **kwargs)
        
        profiler.disable()
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        stats.print_stats(20)
        
        return result
    return wrapper

def time_function(func):
    """Decorator to time function execution."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        
        logger.info(f"{func.__name__} took {elapsed:.2f}s")
        return result
    return wrapper
```

### 2. Optimization Checklist

Before optimizing:
- ✅ Profile to find bottlenecks
- ✅ Verify assumptions with benchmarks
- ✅ Document performance requirements

For Dask operations:
- ✅ Check partition size (aim for 100-500MB)
- ✅ Use `persist()` for frequently accessed data
- ✅ Avoid `.compute()` in loops
- ✅ Use `.map_partitions()` for custom operations
- ✅ Set index on join/merge keys
- ✅ Use categorical dtypes for low-cardinality strings

---

## CI/CD Configuration

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11"]
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -e ".[dev]"
    
    - name: Run ruff
      run: ruff check src tests
    
    - name: Run mypy
      run: mypy src
    
    - name: Run tests
      run: pytest -v --cov
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
```

### Pre-commit Configuration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
  
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
  
  - repo: https://github.com/psf/black
    rev: 23.9.1
    hooks:
      - id: black
  
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.5.1
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
```

---

## Code Review Checklist

Before submitting code:

### Functionality
- [ ] Code does what it's supposed to do
- [ ] Edge cases are handled
- [ ] Error handling is appropriate

### Code Quality
- [ ] Follows naming conventions
- [ ] Type hints on all public functions
- [ ] No code duplication (DRY principle)
- [ ] Functions are focused (single responsibility)
- [ ] Maximum function length: ~50 lines

### Testing
- [ ] Unit tests written and passing
- [ ] Test coverage meets minimum (85%)
- [ ] Integration tests for critical paths
- [ ] Tests are deterministic

### Documentation
- [ ] Docstrings follow Google style
- [ ] Complex logic has inline comments
- [ ] README updated if needed
- [ ] CHANGELOG updated

### Performance
- [ ] No obvious performance issues
- [ ] Dask operations are efficient
- [ ] Memory usage is reasonable
- [ ] Profiled if working on hot paths

### Dependencies
- [ ] New dependencies are justified
- [ ] Versions are pinned appropriately
- [ ] No security vulnerabilities

---

## Git Workflow

### Branch Naming
- `feature/description` - New features
- `bugfix/description` - Bug fixes
- `refactor/description` - Code refactoring
- `docs/description` - Documentation updates

### Commit Messages
```
type(scope): subject

body

footer
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

Example:
```
feat(data): add parquet loader with column selection

Implement ParquetLoader class that supports selective column loading
for improved memory efficiency with large datasets.

Closes #123
```

---

## Security Guidelines

### Never Commit
- Credentials or API keys
- Personal data or PII
- Large binary files
- Environment-specific configs

### Use Environment Variables
```python
import os
from typing import Optional

def get_secret(key: str, default: Optional[str] = None) -> str:
    """
    Get secret from environment variable.
    
    Args:
        key: Environment variable name
        default: Default value if not found
        
    Returns:
        Secret value
        
    Raises:
        ConfigurationError: If secret not found and no default
    """
    value = os.getenv(key, default)
    if value is None:
        raise ConfigurationError(f"Required secret '{key}' not found")
    return value
```

---

## Maintenance

### Regular Tasks
- Update dependencies monthly
- Review and update documentation quarterly
- Audit security vulnerabilities weekly
- Benchmark performance on release candidates
- Clean up deprecated code paths

### Deprecation Process
```python
import warnings
from typing import Any

def deprecated(
    reason: str,
    version: str,
) -> callable:
    """
    Decorator to mark functions as deprecated.
    
    Args:
        reason: Why the function is deprecated
        version: Version when it will be removed
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            warnings.warn(
                f"{func.__name__} is deprecated and will be removed in "
                f"version {version}. {reason}",
                category=DeprecationWarning,
                stacklevel=2,
            )
            return func(*args, **kwargs)
        return wrapper
    return decorator

# Usage
@deprecated(reason="Use new_function instead", version="2.0.0")
def old_function():
    pass
```

---

## Metrics Collection and Monitoring

### Philosophy

Metrics enable data-driven decisions, performance optimization, and operational visibility. Track both business outcomes and technical performance to understand the complete picture of your application.

### Metrics Categories

#### 1. Business/Functional Metrics
- Data processing volume and throughput
- Data quality indicators
- Business rule violations
- Feature usage statistics
- Pipeline completion rates
- Data lineage and provenance

#### 2. Non-Functional/Technical Metrics
- Execution time and latency
- Memory and CPU utilization
- Partition distribution and skew
- Cache hit rates
- Error rates and types
- Dask scheduler performance

---

### Metrics Collection Framework

```python
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Any, Optional
import time
from contextlib import contextmanager
import psutil
import json
from pathlib import Path

class MetricType(Enum):
    """Types of metrics to collect."""
    COUNTER = "counter"          # Cumulative count
    GAUGE = "gauge"              # Point-in-time value
    HISTOGRAM = "histogram"      # Distribution of values
    TIMER = "timer"              # Duration measurements

@dataclass
class Metric:
    """Single metric measurement."""
    name: str
    value: float
    metric_type: MetricType
    timestamp: datetime = field(default_factory=datetime.utcnow)
    tags: dict[str, str] = field(default_factory=dict)
    unit: Optional[str] = None
    
    def to_dict(self) -> dict[str, Any]:
        """Convert metric to dictionary."""
        return {
            'name': self.name,
            'value': self.value,
            'type': self.metric_type.value,
            'timestamp': self.timestamp.isoformat(),
            'tags': self.tags,
            'unit': self.unit,
        }

class MetricsCollector:
    """
    Central metrics collection system.
    
    Collects both business and technical metrics with support for
    multiple backend storage options (file, database, monitoring service).
    
    Examples:
        >>> collector = MetricsCollector(namespace="data_pipeline")
        >>> collector.increment("records_processed", 100)
        >>> collector.gauge("memory_usage_mb", 512.5)
        >>> with collector.timer("computation_time"):
        ...     result = expensive_computation()
    """
    
    def __init__(
        self,
        namespace: str,
        enabled: bool = True,
        buffer_size: int = 1000,
    ):
        self.namespace = namespace
        self.enabled = enabled
        self.buffer_size = buffer_size
        self._metrics: list[Metric] = []
        self._counters: dict[str, float] = {}
    
    def increment(
        self,
        name: str,
        value: float = 1.0,
        tags: Optional[dict[str, str]] = None,
    ) -> None:
        """
        Increment a counter metric.
        
        Args:
            name: Metric name
            value: Amount to increment by
            tags: Optional tags for metric context
        """
        if not self.enabled:
            return
        
        key = f"{self.namespace}.{name}"
        self._counters[key] = self._counters.get(key, 0) + value
        
        self._add_metric(
            name=key,
            value=self._counters[key],
            metric_type=MetricType.COUNTER,
            tags=tags or {},
        )
    
    def gauge(
        self,
        name: str,
        value: float,
        tags: Optional[dict[str, str]] = None,
        unit: Optional[str] = None,
    ) -> None:
        """
        Record a gauge metric (point-in-time value).
        
        Args:
            name: Metric name
            value: Current value
            tags: Optional tags for metric context
            unit: Optional unit of measurement
        """
        if not self.enabled:
            return
        
        self._add_metric(
            name=f"{self.namespace}.{name}",
            value=value,
            metric_type=MetricType.GAUGE,
            tags=tags or {},
            unit=unit,
        )
    
    def histogram(
        self,
        name: str,
        value: float,
        tags: Optional[dict[str, str]] = None,
    ) -> None:
        """Record a histogram metric (for distributions)."""
        if not self.enabled:
            return
        
        self._add_metric(
            name=f"{self.namespace}.{name}",
            value=value,
            metric_type=MetricType.HISTOGRAM,
            tags=tags or {},
        )
    
    @contextmanager
    def timer(
        self,
        name: str,
        tags: Optional[dict[str, str]] = None,
    ):
        """
        Context manager to time code execution.
        
        Examples:
            >>> with collector.timer("data_loading"):
            ...     df = load_large_dataset()
        """
        start_time = time.perf_counter()
        try:
            yield
        finally:
            elapsed = time.perf_counter() - start_time
            self._add_metric(
                name=f"{self.namespace}.{name}",
                value=elapsed,
                metric_type=MetricType.TIMER,
                tags=tags or {},
                unit="seconds",
            )
    
    def _add_metric(
        self,
        name: str,
        value: float,
        metric_type: MetricType,
        tags: dict[str, str],
        unit: Optional[str] = None,
    ) -> None:
        """Add metric to buffer."""
        metric = Metric(
            name=name,
            value=value,
            metric_type=metric_type,
            tags=tags,
            unit=unit,
        )
        self._metrics.append(metric)
        
        # Flush if buffer is full
        if len(self._metrics) >= self.buffer_size:
            self.flush()
    
    def flush(self, output_path: Optional[Path] = None) -> None:
        """
        Flush metrics to storage.
        
        Args:
            output_path: Optional path to write metrics
        """
        if not self._metrics:
            return
        
        if output_path:
            self._write_to_file(output_path)
        
        # Clear buffer after flush
        self._metrics.clear()
    
    def _write_to_file(self, path: Path) -> None:
        """Write metrics to JSON file."""
        path.parent.mkdir(parents=True, exist_ok=True)
        
        with open(path, 'a') as f:
            for metric in self._metrics:
                f.write(json.dumps(metric.to_dict()) + '\n')
    
    def get_summary(self) -> dict[str, Any]:
        """Get summary statistics of collected metrics."""
        summary = {
            'total_metrics': len(self._metrics),
            'counters': dict(self._counters),
            'namespace': self.namespace,
        }
        return summary
```

---

### Business Metrics Implementation

```python
from typing import Protocol
import dask.dataframe as dd

class BusinessMetrics(Protocol):
    """Protocol for business metrics tracking."""
    
    def track_data_quality(self, df: dd.DataFrame) -> dict[str, Any]:
        """Track data quality metrics."""
        ...
    
    def track_processing_volume(self, record_count: int) -> None:
        """Track volume of data processed."""
        ...

class DataQualityMetrics:
    """Track data quality and validation metrics."""
    
    def __init__(self, collector: MetricsCollector):
        self.collector = collector
    
    def track_completeness(
        self,
        df: dd.DataFrame,
        required_columns: list[str],
    ) -> float:
        """
        Track data completeness rate.
        
        Returns:
            Completeness score between 0 and 1
        """
        total_cells = len(df) * len(required_columns)
        
        missing_count = sum(
            df[col].isna().sum().compute()
            for col in required_columns
        )
        
        completeness = 1 - (missing_count / total_cells)
        
        self.collector.gauge(
            "data_quality.completeness",
            completeness,
            tags={"columns": ",".join(required_columns)},
        )
        
        return completeness
    
    def track_duplicates(self, df: dd.DataFrame, key_columns: list[str]) -> int:
        """Track duplicate records."""
        total = len(df)
        unique = df.drop_duplicates(subset=key_columns).compute().shape[0]
        duplicates = total - unique
        
        self.collector.gauge(
            "data_quality.duplicates",
            duplicates,
            tags={"keys": ",".join(key_columns)},
        )
        
        duplicate_rate = duplicates / total if total > 0 else 0
        self.collector.gauge(
            "data_quality.duplicate_rate",
            duplicate_rate,
            tags={"keys": ",".join(key_columns)},
        )
        
        return duplicates
    
    def track_validation_errors(
        self,
        df: dd.DataFrame,
        validation_rules: dict[str, callable],
    ) -> dict[str, int]:
        """
        Track validation rule violations.
        
        Args:
            df: DataFrame to validate
            validation_rules: Dict mapping rule names to validation functions
            
        Returns:
            Dict of rule names to violation counts
        """
        violations = {}
        
        for rule_name, rule_func in validation_rules.items():
            invalid_count = (~df.map_partitions(
                rule_func,
                meta=bool,
            )).sum().compute()
            
            violations[rule_name] = invalid_count
            
            self.collector.gauge(
                "data_quality.validation_errors",
                invalid_count,
                tags={"rule": rule_name},
            )
        
        return violations

class ProcessingMetrics:
    """Track data processing pipeline metrics."""
    
    def __init__(self, collector: MetricsCollector):
        self.collector = collector
    
    def track_pipeline_execution(
        self,
        pipeline_name: str,
        stage: str,
        status: str,
        record_count: int,
        duration: float,
    ) -> None:
        """Track pipeline stage execution."""
        tags = {
            "pipeline": pipeline_name,
            "stage": stage,
            "status": status,
        }
        
        self.collector.increment(
            "pipeline.executions",
            tags=tags,
        )
        
        self.collector.gauge(
            "pipeline.records_processed",
            record_count,
            tags=tags,
            unit="records",
        )
        
        self.collector.histogram(
            "pipeline.duration",
            duration,
            tags=tags,
        )
        
        if record_count > 0 and duration > 0:
            throughput = record_count / duration
            self.collector.gauge(
                "pipeline.throughput",
                throughput,
                tags=tags,
                unit="records/second",
            )
    
    def track_data_transformations(
        self,
        transformation: str,
        input_rows: int,
        output_rows: int,
    ) -> None:
        """Track data transformation metrics."""
        tags = {"transformation": transformation}
        
        self.collector.gauge(
            "transformation.input_rows",
            input_rows,
            tags=tags,
        )
        
        self.collector.gauge(
            "transformation.output_rows",
            output_rows,
            tags=tags,
        )
        
        if input_rows > 0:
            reduction_rate = 1 - (output_rows / input_rows)
            self.collector.gauge(
                "transformation.reduction_rate",
                reduction_rate,
                tags=tags,
            )
```

---

### Technical/Non-Functional Metrics

```python
import threading
from dask.distributed import get_client
from typing import Optional

class SystemMetrics:
    """Track system resource utilization."""
    
    def __init__(self, collector: MetricsCollector):
        self.collector = collector
        self._monitor_thread: Optional[threading.Thread] = None
        self._monitoring = False
    
    def capture_snapshot(self, tags: Optional[dict[str, str]] = None) -> None:
        """Capture current system metrics snapshot."""
        tags = tags or {}
        
        # CPU metrics
        cpu_percent = psutil.cpu_percent(interval=1)
        self.collector.gauge(
            "system.cpu_percent",
            cpu_percent,
            tags=tags,
            unit="percent",
        )
        
        # Memory metrics
        memory = psutil.virtual_memory()
        self.collector.gauge(
            "system.memory_used_mb",
            memory.used / (1024 * 1024),
            tags=tags,
            unit="MB",
        )
        self.collector.gauge(
            "system.memory_percent",
            memory.percent,
            tags=tags,
            unit="percent",
        )
        
        # Disk I/O
        disk_io = psutil.disk_io_counters()
        if disk_io:
            self.collector.gauge(
                "system.disk_read_mb",
                disk_io.read_bytes / (1024 * 1024),
                tags=tags,
                unit="MB",
            )
            self.collector.gauge(
                "system.disk_write_mb",
                disk_io.write_bytes / (1024 * 1024),
                tags=tags,
                unit="MB",
            )
    
    def start_monitoring(self, interval: int = 30) -> None:
        """Start background system monitoring."""
        if self._monitoring:
            return
        
        self._monitoring = True
        
        def monitor():
            while self._monitoring:
                self.capture_snapshot()
                time.sleep(interval)
        
        self._monitor_thread = threading.Thread(target=monitor, daemon=True)
        self._monitor_thread.start()
    
    def stop_monitoring(self) -> None:
        """Stop background monitoring."""
        self._monitoring = False
        if self._monitor_thread:
            self._monitor_thread.join(timeout=5)

class DaskMetrics:
    """Track Dask-specific performance metrics."""
    
    def __init__(self, collector: MetricsCollector):
        self.collector = collector
    
    def track_partition_stats(
        self,
        df: dd.DataFrame,
        operation: str,
    ) -> None:
        """Track partition distribution and balance."""
        partition_sizes = df.map_partitions(len).compute()
        
        tags = {"operation": operation}
        
        self.collector.gauge(
            "dask.partitions.count",
            df.npartitions,
            tags=tags,
        )
        
        self.collector.gauge(
            "dask.partitions.mean_size",
            partition_sizes.mean(),
            tags=tags,
            unit="rows",
        )
        
        self.collector.gauge(
            "dask.partitions.std_size",
            partition_sizes.std(),
            tags=tags,
            unit="rows",
        )
        
        # Track partition skew
        if partition_sizes.mean() > 0:
            skew = partition_sizes.std() / partition_sizes.mean()
            self.collector.gauge(
                "dask.partitions.skew",
                skew,
                tags=tags,
            )
    
    def track_task_stream(self, operation: str) -> None:
        """Track Dask task execution metrics from scheduler."""
        try:
            client = get_client()
            
            # Get scheduler info
            info = client.scheduler_info()
            
            tags = {"operation": operation}
            
            # Worker metrics
            self.collector.gauge(
                "dask.workers.count",
                len(info['workers']),
                tags=tags,
            )
            
            # Task metrics
            tasks = client.who_has()
            self.collector.gauge(
                "dask.tasks.count",
                len(tasks),
                tags=tags,
            )
            
        except ValueError:
            # No client available
            pass
    
    def track_memory_usage(
        self,
        df: dd.DataFrame,
        stage: str,
    ) -> None:
        """Track DataFrame memory usage."""
        # Estimate memory per partition
        sample = df.head(1000, npartitions=1)
        bytes_per_row = sample.memory_usage(deep=True).sum() / len(sample)
        
        estimated_memory_mb = (
            len(df) * bytes_per_row / (1024 * 1024)
        )
        
        self.collector.gauge(
            "dask.dataframe.estimated_memory_mb",
            estimated_memory_mb,
            tags={"stage": stage},
            unit="MB",
        )

class PerformanceMetrics:
    """Track performance and latency metrics."""
    
    def __init__(self, collector: MetricsCollector):
        self.collector = collector
    
    def track_operation_timing(
        self,
        operation: str,
        duration: float,
        success: bool = True,
    ) -> None:
        """Track operation execution time."""
        tags = {
            "operation": operation,
            "status": "success" if success else "failure",
        }
        
        self.collector.histogram(
            "performance.operation_duration",
            duration,
            tags=tags,
        )
        
        self.collector.increment(
            "performance.operations",
            tags=tags,
        )
    
    def track_cache_performance(
        self,
        cache_name: str,
        hits: int,
        misses: int,
    ) -> None:
        """Track cache hit/miss rates."""
        tags = {"cache": cache_name}
        
        total = hits + misses
        hit_rate = hits / total if total > 0 else 0
        
        self.collector.gauge(
            "performance.cache_hit_rate",
            hit_rate,
            tags=tags,
        )
        
        self.collector.increment(
            "performance.cache_hits",
            hits,
            tags=tags,
        )
        
        self.collector.increment(
            "performance.cache_misses",
            misses,
            tags=tags,
        )
```

---

### Metrics Decorators

```python
from functools import wraps
import traceback

def track_execution(
    collector: MetricsCollector,
    operation_name: Optional[str] = None,
    track_args: bool = False,
):
    """
    Decorator to automatically track function execution metrics.
    
    Examples:
        >>> @track_execution(collector, "data_loading")
        ... def load_data(path: str) -> dd.DataFrame:
        ...     return dd.read_parquet(path)
    """
    def decorator(func):
        name = operation_name or func.__name__
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            tags = {"function": name}
            
            if track_args:
                tags.update({
                    f"arg_{i}": str(arg)[:50]
                    for i, arg in enumerate(args[:3])
                })
            
            start_time = time.perf_counter()
            success = False
            
            try:
                result = func(*args, **kwargs)
                success = True
                return result
                
            except Exception as e:
                collector.increment(
                    "errors",
                    tags={**tags, "error_type": type(e).__name__},
                )
                raise
                
            finally:
                duration = time.perf_counter() - start_time
                
                collector.histogram(
                    "function_duration",
                    duration,
                    tags={**tags, "status": "success" if success else "failure"},
                )
                
                collector.increment(
                    "function_calls",
                    tags=tags,
                )
        
        return wrapper
    return decorator

def track_dataframe_operation(
    collector: MetricsCollector,
    operation_type: str,
):
    """
    Decorator specifically for DataFrame operations.
    
    Tracks input/output row counts and operation performance.
    """
    def decorator(func):
        @wraps(func)
        def wrapper(df: dd.DataFrame, *args, **kwargs):
            input_rows = len(df)
            
            with collector.timer(f"dataframe.{operation_type}"):
                result = func(df, *args, **kwargs)
            
            output_rows = len(result) if isinstance(result, dd.DataFrame) else 0
            
            tags = {"operation": operation_type}
            
            collector.gauge(
                "dataframe.input_rows",
                input_rows,
                tags=tags,
            )
            
            if isinstance(result, dd.DataFrame):
                collector.gauge(
                    "dataframe.output_rows",
                    output_rows,
                    tags=tags,
                )
            
            return result
        
        return wrapper
    return decorator
```

---

### Metrics Integration Example

```python
class DataPipeline:
    """Example pipeline with comprehensive metrics."""
    
    def __init__(self, config: AppConfig):
        self.config = config
        
        # Initialize metrics collectors
        self.metrics = MetricsCollector(namespace="pipeline")
        self.quality_metrics = DataQualityMetrics(self.metrics)
        self.processing_metrics = ProcessingMetrics(self.metrics)
        self.system_metrics = SystemMetrics(self.metrics)
        self.dask_metrics = DaskMetrics(self.metrics)
        self.perf_metrics = PerformanceMetrics(self.metrics)
    
    @track_execution(MetricsCollector("pipeline"), "load_data")
    def load_data(self, path: Path) -> dd.DataFrame:
        """Load data with metrics tracking."""
        with self.metrics.timer("data.loading"):
            df = dd.read_parquet(path)
        
        # Track data quality
        self.quality_metrics.track_completeness(
            df,
            required_columns=['id', 'timestamp', 'value'],
        )
        
        # Track partition stats
        self.dask_metrics.track_partition_stats(df, "load")
        
        return df
    
    @track_dataframe_operation(MetricsCollector("pipeline"), "transform")
    def transform(self, df: dd.DataFrame) -> dd.DataFrame:
        """Transform data with metrics."""
        input_rows = len(df)
        
        # Capture system state before heavy computation
        self.system_metrics.capture_snapshot(tags={"stage": "transform"})
        
        with self.metrics.timer("transform.computation"):
            result = (
                df
                .query('value > 0')
                .groupby('category')
                .agg({'value': ['mean', 'sum', 'count']})
            )
        
        output_rows = len(result)
        
        # Track transformation metrics
        self.processing_metrics.track_data_transformations(
            transformation="aggregation",
            input_rows=input_rows,
            output_rows=output_rows,
        )
        
        return result
    
    def run(self, input_path: Path, output_path: Path) -> None:
        """Execute pipeline with full metrics tracking."""
        pipeline_start = time.perf_counter()
        
        try:
            # Start system monitoring
            self.system_metrics.start_monitoring(interval=10)
            
            # Load data
            df = self.load_data(input_path)
            
            # Validate data
            validation_rules = {
                'positive_values': lambda x: x['value'] > 0,
                'valid_category': lambda x: x['category'].isin(['A', 'B', 'C']),
            }
            violations = self.quality_metrics.track_validation_errors(
                df,
                validation_rules,
            )
            
            # Transform
            result = self.transform(df)
            
            # Save
            with self.metrics.timer("data.saving"):
                result.to_parquet(output_path)
            
            # Track pipeline success
            duration = time.perf_counter() - pipeline_start
            self.processing_metrics.track_pipeline_execution(
                pipeline_name="main_pipeline",
                stage="complete",
                status="success",
                record_count=len(result),
                duration=duration,
            )
            
        except Exception as e:
            duration = time.perf_counter() - pipeline_start
            self.processing_metrics.track_pipeline_execution(
                pipeline_name="main_pipeline",
                stage="failed",
                status="error",
                record_count=0,
                duration=duration,
            )
            raise
            
        finally:
            # Stop monitoring and flush metrics
            self.system_metrics.stop_monitoring()
            self.metrics.flush(output_path=Path("metrics/pipeline_metrics.jsonl"))
            
            # Print summary
            summary = self.metrics.get_summary()
            logger.info(f"Pipeline metrics: {summary}")
```

---

### Metrics Visualization and Reporting

```python
import pandas as pd
import json
from pathlib import Path
from typing import Optional

class MetricsReporter:
    """Generate reports from collected metrics."""
    
    @staticmethod
    def load_metrics(path: Path) -> pd.DataFrame:
        """Load metrics from JSONL file."""
        metrics = []
        with open(path) as f:
            for line in f:
                metrics.append(json.loads(line))
        
        df = pd.DataFrame(metrics)
        df['timestamp'] = pd.to_datetime(df['timestamp'])
        return df
    
    @staticmethod
    def generate_summary_report(
        metrics_df: pd.DataFrame,
        output_path: Optional[Path] = None,
    ) -> dict[str, Any]:
        """Generate summary report from metrics."""
        report = {
            'period': {
                'start': metrics_df['timestamp'].min().isoformat(),
                'end': metrics_df['timestamp'].max().isoformat(),
            },
            'business_metrics': {},
            'technical_metrics': {},
        }
        
        # Business metrics
        quality_metrics = metrics_df[
            metrics_df['name'].str.contains('data_quality')
        ]
        if not quality_metrics.empty:
            report['business_metrics']['data_quality'] = {
                'avg_completeness': quality_metrics[
                    quality_metrics['name'] == 'data_quality.completeness'
                ]['value'].mean(),
                'total_duplicates': quality_metrics[
                    quality_metrics['name'] == 'data_quality.duplicates'
                ]['value'].sum(),
            }
        
        # Technical metrics
        perf_metrics = metrics_df[
            metrics_df['name'].str.contains('performance')
        ]
        if not perf_metrics.empty:
            report['technical_metrics']['performance'] = {
                'avg_duration': perf_metrics[
                    perf_metrics['name'] == 'performance.operation_duration'
                ]['value'].mean(),
                'p95_duration': perf_metrics[
                    perf_metrics['name'] == 'performance.operation_duration'
                ]['value'].quantile(0.95),
            }
        
        if output_path:
            with open(output_path, 'w') as f:
                json.dump(report, f, indent=2)
        
        return report
```

---

### Integration with Monitoring Services

```python
from typing import Protocol

class MetricsBackend(Protocol):
    """Protocol for metrics storage backends."""
    
    def send_metrics(self, metrics: list[Metric]) -> None:
        """Send metrics to backend."""
        ...

class PrometheusBackend:
    """Send metrics to Prometheus pushgateway."""
    
    def __init__(self, pushgateway_url: str, job_name: str):
        self.pushgateway_url = pushgateway_url
        self.job_name = job_name
    
    def send_metrics(self, metrics: list[Metric]) -> None:
        """Push metrics to Prometheus."""
        # Implementation would use prometheus_client library
        pass

class CloudWatchBackend:
    """Send metrics to AWS CloudWatch."""
    
    def __init__(self, namespace: str, region: str):
        self.namespace = namespace
        self.region = region
    
    def send_metrics(self, metrics: list[Metric]) -> None:
        """Put metrics to CloudWatch."""
        # Implementation would use boto3
        pass
```

---

### Metrics Best Practices

#### Collection Guidelines
- ✅ Use appropriate metric types (counter for cumulative, gauge for snapshots)
- ✅ Add meaningful tags for filtering and grouping
- ✅ Track both success and failure cases
- ✅ Use consistent naming conventions (namespace.category.metric)
- ✅ Set appropriate buffer sizes to balance memory and I/O

#### What to Track
**Business Metrics:**
- Data volume processed
- Data quality scores
- Pipeline completion rates
- SLA compliance
- Feature usage

**Technical Metrics:**
- Execution time (p50, p95, p99)
- Memory and CPU usage
- Error rates by type
- Cache hit rates
- Partition balance
- Query performance

#### What to Avoid
- ❌ High-cardinality tags (e.g., unique IDs)
- ❌ Excessive metric collection that impacts performance
- ❌ Metrics without clear business or technical value
- ❌ Inconsistent naming across modules

---

### Alerting Configuration

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class AlertRule:
    """Define alerting rule based on metrics."""
    
    metric_name: str
    condition: Callable[[float], bool]
    severity: str  # "warning", "error", "critical"
    message: str
    
    def evaluate(self, value: float) -> Optional[str]:
        """Evaluate if alert should fire."""
        if self.condition(value):
            return f"[{self.severity.upper()}] {self.message}: {value}"
        return None

class MetricsAlerting:
    """Monitor metrics and trigger alerts."""
    
    def __init__(self, collector: MetricsCollector):
        self.collector = collector
        self.rules: list[AlertRule] = []
    
    def add_rule(self, rule: AlertRule) -> None:
        """Add alerting rule."""
        self.rules.append(rule)
    
    def check_alerts(self) -> list[str]:
        """Check all rules and return fired alerts."""
        alerts = []
        
        for metric in self.collector._metrics:
            for rule in self.rules:
                if rule.metric_name in metric.name:
                    alert = rule.evaluate(metric.value)
                    if alert:
                        alerts.append(alert)
                        logger.warning(alert)
        
        return alerts

# Usage example
alerting = MetricsAlerting(collector)

# Add rules
alerting.add_rule(AlertRule(
    metric_name="data_quality.completeness",
    condition=lambda x: x < 0.95,
    severity="warning",
    message="Data completeness below threshold",
))

alerting.add_rule(AlertRule(
    metric_name="system.memory_percent",
    condition=lambda x: x > 90,
    severity="critical",
    message="Memory usage critical",
))
```

---

## Resources and References

### Essential Reading
- [Dask Best Practices](https://docs.dask.org/en/latest/best-practices.html)
- [Effective Python](https://effectivepython.com/)
- [Clean Code in Python](https://github.com/zedr/clean-code-python)

### Tools
- **Ruff**: Fast Python linter
- **Black**: Code formatter
- **mypy**: Static type checker
- **pytest**: Testing framework
- **Dask Dashboard**: Monitoring at localhost:8787

### Community Standards
Follow PEPs:
- PEP 8: Style Guide
- PEP 257: Docstring Conventions
- PEP 484: Type Hints
- PEP 621: Metadata in pyproject.toml

---

## Conclusion

This constitution is a living document. As the project evolves and new best practices emerge, update this guide. Code quality is not a destination but a continuous journey. Write code that your future self will thank you for.

**Remember**: Good code is code that is easy to delete.
