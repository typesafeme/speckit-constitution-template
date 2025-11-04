# Python Project Constitution

This constitution defines the standards, practices, and principles for building clean, maintainable, and production-ready Python projects. It draws from battle-tested patterns used in successful open-source projects like Flask, FastAPI, requests, pytest, and Click.

## Core Principles

1. **Readability counts** - Code is read far more often than it is written
2. **Explicit is better than implicit** - Clear intentions over clever tricks
3. **Simple is better than complex** - Favor straightforward solutions
4. **Errors should never pass silently** - Handle exceptions properly
5. **Modularity by design** - Each component should have a single responsibility

## Project Structure

```
project-name/
├── src/
│   └── project_name/          # Main package (use underscores)
│       ├── __init__.py         # Package initialization
│       ├── __main__.py         # CLI entry point (if applicable)
│       ├── core/               # Core business logic
│       │   ├── __init__.py
│       │   └── domain.py       # Domain models
│       ├── api/                # API layer (if applicable)
│       │   ├── __init__.py
│       │   ├── routes.py
│       │   └── schemas.py
│       ├── services/           # Business logic services
│       │   ├── __init__.py
│       │   └── service.py
│       ├── repositories/       # Data access layer
│       │   ├── __init__.py
│       │   └── repository.py
│       ├── utils/              # Utility functions
│       │   ├── __init__.py
│       │   └── helpers.py
│       ├── config.py           # Configuration management
│       └── exceptions.py       # Custom exceptions
├── tests/
│   ├── __init__.py
│   ├── conftest.py             # pytest fixtures
│   ├── unit/                   # Unit tests
│   ├── integration/            # Integration tests
│   └── fixtures/               # Test data
├── docs/
│   ├── api.md
│   ├── architecture.md
│   └── contributing.md
├── scripts/                    # Development scripts
│   └── setup.sh
├── pyproject.toml              # Project metadata & dependencies
├── README.md                   # Project overview
├── CHANGELOG.md                # Version history
├── LICENSE                     # License file
├── .gitignore
├── .pre-commit-config.yaml     # Pre-commit hooks
└── Makefile                    # Common tasks

```

## Code Style & Formatting

### PEP 8 Compliance
- Use **Black** (line length: 88) for automatic formatting
- Use **isort** for import sorting
- Use **flake8** for linting
- Use **mypy** for static type checking

### Import Organization
```python
# Standard library imports
import os
import sys
from pathlib import Path

# Third-party imports
import requests
from fastapi import FastAPI

# Local application imports
from project_name.core import models
from project_name.services import user_service
from project_name.utils import helpers
```

### Naming Conventions
- **Modules**: `lowercase_with_underscores.py`
- **Classes**: `PascalCase`
- **Functions/Methods**: `lowercase_with_underscores()`
- **Constants**: `UPPERCASE_WITH_UNDERSCORES`
- **Private members**: `_leading_underscore`
- **Package names**: `lowercasenospaces`

## Type Hints

Use type hints for all function signatures and class attributes (Python 3.10+ syntax preferred):

```python
from typing import Optional, Union
from collections.abc import Callable, Iterable

def process_data(
    data: list[dict[str, Any]],
    filter_fn: Callable[[dict], bool] | None = None,
    max_items: int = 100
) -> list[dict[str, Any]]:
    """Process and filter data items.
    
    Args:
        data: List of data dictionaries to process
        filter_fn: Optional filter function to apply
        max_items: Maximum number of items to return
        
    Returns:
        Filtered and processed data list
        
    Raises:
        ValueError: If data is empty or invalid
    """
    if not data:
        raise ValueError("Data cannot be empty")
    
    result = data if filter_fn is None else [item for item in data if filter_fn(item)]
    return result[:max_items]
```

## Documentation Standards

### Docstrings
Use **Google-style** docstrings for all public modules, classes, functions, and methods:

```python
class UserService:
    """Service for managing user operations.
    
    This service handles all user-related business logic including
    creation, authentication, and profile management.
    
    Attributes:
        repository: User data repository instance
        cache: Cache instance for user data
        
    Example:
        >>> service = UserService(repo, cache)
        >>> user = service.create_user("john@example.com")
    """
    
    def __init__(self, repository: UserRepository, cache: Cache) -> None:
        """Initialize the user service.
        
        Args:
            repository: Repository for user data access
            cache: Cache instance for performance optimization
        """
        self.repository = repository
        self.cache = cache
```

### README.md Structure
```markdown
# Project Name

Brief description (one paragraph)

## Features
- Feature 1
- Feature 2

## Installation
pip install project-name

## Quick Start
[Code example]

## Documentation
Link to full documentation

## Contributing
Link to CONTRIBUTING.md

## License
[License type]
```

## Testing Philosophy

### Test Organization
- **Unit tests**: Test individual functions/methods in isolation
- **Integration tests**: Test component interactions
- **End-to-end tests**: Test complete workflows
- **Aim for 80%+ code coverage** (measured with pytest-cov)

### Test Structure (AAA Pattern)
```python
import pytest
from project_name.services import UserService

def test_create_user_success(user_repository, cache):
    """Test successful user creation with valid data."""
    # Arrange
    service = UserService(user_repository, cache)
    email = "test@example.com"
    
    # Act
    user = service.create_user(email)
    
    # Assert
    assert user.email == email
    assert user.id is not None
    assert user.created_at is not None

def test_create_user_duplicate_email_raises_error(user_repository, cache):
    """Test that duplicate email raises appropriate error."""
    # Arrange
    service = UserService(user_repository, cache)
    email = "test@example.com"
    service.create_user(email)
    
    # Act & Assert
    with pytest.raises(ValueError, match="User already exists"):
        service.create_user(email)
```

### Fixtures (conftest.py)
```python
import pytest
from project_name.repositories import UserRepository
from project_name.core import Cache

@pytest.fixture
def user_repository():
    """Provide a clean user repository for testing."""
    repo = UserRepository(":memory:")
    yield repo
    repo.close()

@pytest.fixture
def cache():
    """Provide a cache instance for testing."""
    return Cache(max_size=100)
```

## Error Handling

### Custom Exceptions
```python
# exceptions.py
class ProjectBaseException(Exception):
    """Base exception for all project exceptions."""
    pass

class ValidationError(ProjectBaseException):
    """Raised when data validation fails."""
    pass

class NotFoundError(ProjectBaseException):
    """Raised when a requested resource is not found."""
    pass

class AuthenticationError(ProjectBaseException):
    """Raised when authentication fails."""
    pass
```

### Exception Handling Pattern
```python
from project_name.exceptions import NotFoundError, ValidationError

def get_user_by_id(user_id: int) -> User:
    """Retrieve user by ID.
    
    Args:
        user_id: The user's unique identifier
        
    Returns:
        User object if found
        
    Raises:
        ValidationError: If user_id is invalid
        NotFoundError: If user doesn't exist
    """
    if user_id <= 0:
        raise ValidationError(f"Invalid user_id: {user_id}")
    
    user = repository.find_by_id(user_id)
    if user is None:
        raise NotFoundError(f"User not found: {user_id}")
    
    return user
```

## Configuration Management

### Using pydantic-settings
```python
# config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache

class Settings(BaseSettings):
    """Application settings with environment variable support."""
    
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False
    )
    
    # Application
    app_name: str = "My Project"
    debug: bool = False
    environment: str = "production"
    
    # Database
    database_url: str = "sqlite:///./app.db"
    db_pool_size: int = 5
    
    # API
    api_key: str
    api_timeout: int = 30
    
    # Logging
    log_level: str = "INFO"

@lru_cache
def get_settings() -> Settings:
    """Get cached settings instance."""
    return Settings()
```

## Logging

### Structured Logging Setup
```python
# logging_config.py
import logging
import sys
from pathlib import Path

def setup_logging(level: str = "INFO", log_file: Path | None = None) -> None:
    """Configure application logging.
    
    Args:
        level: Logging level (DEBUG, INFO, WARNING, ERROR, CRITICAL)
        log_file: Optional file path for log output
    """
    handlers: list[logging.Handler] = [logging.StreamHandler(sys.stdout)]
    
    if log_file:
        handlers.append(logging.FileHandler(log_file))
    
    logging.basicConfig(
        level=getattr(logging, level.upper()),
        format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
        handlers=handlers
    )

# Usage in modules
logger = logging.getLogger(__name__)
```

## Dependency Management

### pyproject.toml Structure
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "project-name"
version = "0.1.0"
description = "A brief description"
authors = [{name = "Your Name", email = "you@example.com"}]
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.10"
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]

dependencies = [
    "requests>=2.31.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "pytest-asyncio>=0.21.0",
    "black>=23.7.0",
    "isort>=5.12.0",
    "flake8>=6.1.0",
    "mypy>=1.5.0",
    "pre-commit>=3.3.0",
]

[project.scripts]
project-name = "project_name.__main__:main"

[tool.black]
line-length = 88
target-version = ["py310", "py311", "py312"]

[tool.isort]
profile = "black"
line_length = 88

[tool.mypy]
python_version = "3.10"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--strict-markers",
    "--cov=src/project_name",
    "--cov-report=term-missing",
    "--cov-report=html",
]

[tool.coverage.run]
source = ["src/project_name"]
omit = ["*/tests/*", "*/__main__.py"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
]
```

## Architecture Patterns

### Dependency Injection
```python
# main.py
from project_name.repositories import UserRepository
from project_name.services import UserService
from project_name.core import Cache

class Container:
    """Dependency injection container."""
    
    def __init__(self, settings: Settings) -> None:
        self.settings = settings
        self._cache = Cache()
        self._user_repository = UserRepository(settings.database_url)
        self._user_service: UserService | None = None
    
    @property
    def user_service(self) -> UserService:
        """Get or create user service instance."""
        if self._user_service is None:
            self._user_service = UserService(
                self._user_repository,
                self._cache
            )
        return self._user_service
```

### Repository Pattern
```python
# repositories/user_repository.py
from abc import ABC, abstractmethod
from typing import Optional

class UserRepositoryInterface(ABC):
    """Abstract interface for user data access."""
    
    @abstractmethod
    def find_by_id(self, user_id: int) -> Optional[User]:
        """Find user by ID."""
        pass
    
    @abstractmethod
    def find_by_email(self, email: str) -> Optional[User]:
        """Find user by email."""
        pass
    
    @abstractmethod
    def save(self, user: User) -> User:
        """Save user to storage."""
        pass

class UserRepository(UserRepositoryInterface):
    """Concrete implementation of user repository."""
    
    def __init__(self, database_url: str) -> None:
        self.db = Database(database_url)
    
    def find_by_id(self, user_id: int) -> Optional[User]:
        """Find user by ID."""
        row = self.db.query("SELECT * FROM users WHERE id = ?", (user_id,))
        return User(**row) if row else None
```

### Service Layer
```python
# services/user_service.py
class UserService:
    """Business logic for user operations."""
    
    def __init__(
        self,
        repository: UserRepositoryInterface,
        cache: Cache
    ) -> None:
        self.repository = repository
        self.cache = cache
        self.logger = logging.getLogger(__name__)
    
    def create_user(self, email: str, name: str) -> User:
        """Create a new user.
        
        Args:
            email: User email address
            name: User full name
            
        Returns:
            Created user instance
            
        Raises:
            ValidationError: If email format is invalid
            ValueError: If user already exists
        """
        self._validate_email(email)
        
        if self.repository.find_by_email(email):
            raise ValueError(f"User already exists: {email}")
        
        user = User(email=email, name=name)
        user = self.repository.save(user)
        
        self.logger.info(f"Created user: {user.id}")
        self.cache.set(f"user:{user.id}", user)
        
        return user
    
    def _validate_email(self, email: str) -> None:
        """Validate email format."""
        if "@" not in email:
            raise ValidationError(f"Invalid email: {email}")
```

## CLI Development (using Click)

```python
# __main__.py
import click
from project_name.config import get_settings
from project_name.services import UserService

@click.group()
@click.version_option()
def cli() -> None:
    """Project Name CLI tool."""
    pass

@cli.command()
@click.option("--email", required=True, help="User email address")
@click.option("--name", required=True, help="User full name")
def create_user(email: str, name: str) -> None:
    """Create a new user."""
    settings = get_settings()
    service = UserService(settings)
    
    try:
        user = service.create_user(email, name)
        click.echo(f"Created user: {user.id}")
    except Exception as e:
        click.echo(f"Error: {e}", err=True)
        raise click.Abort()

if __name__ == "__main__":
    cli()
```

## Pre-commit Configuration

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
      - id: check-json
      - id: check-toml
      
  - repo: https://github.com/psf/black
    rev: 23.7.0
    hooks:
      - id: black
      
  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
      
  - repo: https://github.com/pycqa/flake8
    rev: 6.1.0
    hooks:
      - id: flake8
        args: [--max-line-length=88, --extend-ignore=E203]
        
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.5.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
```

## Makefile for Common Tasks

```makefile
.PHONY: install test lint format clean docs

install:
	pip install -e ".[dev]"
	pre-commit install

test:
	pytest tests/ -v --cov=src/project_name --cov-report=html

test-watch:
	pytest-watch tests/ -- -v

lint:
	flake8 src/ tests/
	mypy src/
	black --check src/ tests/
	isort --check-only src/ tests/

format:
	black src/ tests/
	isort src/ tests/

clean:
	rm -rf build/
	rm -rf dist/
	rm -rf *.egg-info
	rm -rf .pytest_cache
	rm -rf .coverage
	rm -rf htmlcov/
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -type f -name "*.pyc" -delete

docs:
	cd docs && make html

run:
	python -m project_name

build:
	python -m build

publish:
	python -m twine upload dist/*
```

## Best Practices Checklist

### Before Committing
- [ ] All tests pass
- [ ] Code coverage is maintained or improved
- [ ] Type hints are added for new functions
- [ ] Docstrings are added for public APIs
- [ ] Code is formatted with Black and isort
- [ ] No linting errors from flake8
- [ ] Mypy type checking passes
- [ ] CHANGELOG.md is updated (if applicable)

### Code Review Focus
- [ ] Is the code readable and self-explanatory?
- [ ] Are there appropriate tests?
- [ ] Is error handling comprehensive?
- [ ] Are there any security concerns?
- [ ] Is the solution over-engineered?
- [ ] Are there any performance implications?
- [ ] Is documentation adequate?

### Performance Considerations
- Use generators for large datasets
- Profile before optimizing (`cProfile`, `line_profiler`)
- Cache expensive operations (`functools.lru_cache`)
- Use appropriate data structures (dict for lookups, set for membership)
- Consider async/await for I/O-bound operations
- Batch database operations where possible

### Security Guidelines
- Never commit secrets or API keys (use environment variables)
- Validate and sanitize all user inputs
- Use parameterized queries to prevent SQL injection
- Keep dependencies up to date (`pip-audit`, `safety`)
- Follow principle of least privilege
- Use secure random for security-sensitive operations (`secrets` module)

## Anti-Patterns to Avoid

1. **God Objects**: Classes that know/do too much
2. **Circular Dependencies**: Modules importing each other
3. **Mutable Default Arguments**: Use `None` and initialize inside function
4. **Catching Generic Exceptions**: Be specific about what you catch
5. **Not Using Context Managers**: Always use `with` for resources
6. **String Concatenation in Loops**: Use `str.join()` instead
7. **Global State**: Minimize use of global variables
8. **Monkey Patching**: Avoid modifying third-party code at runtime

## Resources & References

- [PEP 8](https://peps.python.org/pep-0008/) - Style Guide
- [PEP 257](https://peps.python.org/pep-0257/) - Docstring Conventions
- [Python Packaging Guide](https://packaging.python.org/)
- [Real Python](https://realpython.com/) - Tutorials and Best Practices
- [Effective Python](https://effectivepython.com/) - Book by Brett Slatkin
- [Clean Code](https://www.oreilly.com/library/view/clean-code-a/9780136083238/) - Principles by Robert C. Martin

---

**Remember**: This is a living document. Adapt these guidelines to your specific project needs while maintaining the core principles of clean, maintainable, and well-tested code.
