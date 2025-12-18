# CLAUDE.md - AI Assistant Guide for pbir-utils

This document provides comprehensive guidance for AI assistants (like Claude) working on the `pbir-utils` codebase. It outlines the project structure, development workflows, coding conventions, and key patterns to follow.

## Project Overview

**pbir-utils** is a Python library and CLI tool designed to streamline tasks that Power BI developers typically handle manually in Power BI Desktop. It provides utilities to efficiently manage and manipulate PBIR (Power BI Enhanced Report Format) metadata.

- **Language**: Python 3.10+
- **License**: MIT
- **Package Manager**: pip
- **Build System**: setuptools with setuptools-scm (version from git tags)
- **Documentation**: MkDocs with Material theme
- **GitHub**: https://github.com/akhilannan/pbir-utils

## Repository Structure

```
pbir-utils/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                 # CI pipeline (tests, linting, security)
│   │   └── python-publish.yml     # PyPI publishing workflow
│   ├── ISSUE_TEMPLATE/
│   └── PULL_REQUEST_TEMPLATE.md
├── docs/                          # MkDocs documentation
│   ├── index.md
│   ├── cli.md
│   └── api.md
├── examples/
│   └── example_usage.ipynb        # Jupyter notebook with examples
├── src/pbir_utils/
│   ├── __init__.py               # Public API exports
│   ├── cli.py                    # CLI entry point
│   ├── command_utils.py          # Shared CLI argument helpers
│   ├── common.py                 # Core utilities (JSON I/O, path resolution)
│   ├── console_utils.py          # Console output utilities
│   ├── commands/                 # CLI command modules
│   │   ├── __init__.py
│   │   ├── batch_update.py
│   │   ├── extract_metadata.py
│   │   ├── filters.py
│   │   ├── interactions.py
│   │   ├── measures.py
│   │   ├── pages.py
│   │   ├── sanitize.py
│   │   └── visualize.py
│   ├── defaults/
│   │   └── sanitize.yaml         # Default sanitization configuration
│   ├── bookmark_utils.py
│   ├── filter_utils.py
│   ├── folder_standardizer.py
│   ├── metadata_extractor.py
│   ├── page_utils.py
│   ├── pbir_measure_utils.py
│   ├── pbir_processor.py
│   ├── pbir_report_sanitizer.py
│   ├── report_wireframe_visualizer.py
│   ├── sanitize_config.py
│   ├── visual_interactions_utils.py
│   └── visual_utils.py
├── tests/                        # 19 test files
│   ├── conftest.py
│   ├── test_*.py
│   └── ...
├── .gitignore
├── CHANGELOG.md
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── LICENSE
├── README.md
├── SECURITY.md
├── mkdocs.yml
└── pyproject.toml
```

## Core Technologies & Dependencies

### Production Dependencies
- **dash**: Web framework for visualizations
- **plotly**: Interactive visualizations
- **PyYAML**: YAML configuration file parsing

### Development Dependencies
- **pytest** & **pytest-cov**: Testing framework and coverage
- **ruff**: Fast Python linter (replaces flake8, isort, etc.)
- **black**: Code formatter
- **bandit**: Security vulnerability scanner
- **pip-audit**: Dependency vulnerability checking
- **build** & **twine**: Package building and publishing

### Documentation Dependencies
- **mkdocs**: Documentation site generator
- **mkdocs-material**: Material Design theme for MkDocs

## Development Setup

### Initial Setup

```bash
# Clone the repository
git clone https://github.com/akhilannan/pbir-utils.git
cd pbir-utils

# Create and activate virtual environment
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
# or
.venv\Scripts\activate     # Windows

# Install in editable mode with dev dependencies
pip install -e .[dev,docs]
```

### Running Tests

```bash
# Run all tests
pytest

# Run with coverage report
pytest --cov=src --cov-report=html

# Run specific test file
pytest tests/test_cli.py

# Run with verbose output
pytest -v
```

**Important**: CI requires minimum 60% test coverage (`--cov-fail-under=60`).

### Code Quality Tools

#### Formatting with Black
```bash
# Check formatting
black --check src tests

# Format code
black src tests
```

**Note**: The project uses `ruff format` (which is black-compatible) as mentioned in CONTRIBUTING.md:
```bash
# Format code with ruff
ruff format .
```

#### Linting with Ruff
```bash
# Check for linting issues
ruff check src tests

# Auto-fix issues where possible
ruff check --fix src tests
```

#### Security Scanning
```bash
# Check for security vulnerabilities in code
bandit -r src

# Check for vulnerabilities in dependencies
pip-audit
```

### Documentation

```bash
# Serve documentation locally (with live reload)
mkdocs serve

# Build documentation
mkdocs build

# Documentation will be available at http://127.0.0.1:8000/
```

## Code Style & Conventions

### Python Version
- **Minimum**: Python 3.10
- **CI Testing**: Python 3.10, 3.11, 3.12, 3.13
- Use modern Python features available in 3.10+ (pattern matching, type unions with `|`, etc.)

### Type Hints
- Use type hints for function parameters and return values
- Use modern union syntax: `str | None` instead of `Optional[str]`
- Use `typing` module imports when needed

Example:
```python
from typing import Dict, Set

def parse_filters(filters_str: str) -> Dict[str, Set[str]] | None:
    """Parse a JSON string into a filters dictionary."""
    if not filters_str:
        return None
    # ...
```

### Docstrings
- Use clear, concise docstrings for public functions
- Include Args, Returns, and Raises sections when applicable
- See existing code for examples (e.g., `common.py`, `command_utils.py`)

### Naming Conventions
- **Functions/Variables**: `snake_case`
- **Classes**: `PascalCase`
- **Constants**: `UPPER_SNAKE_CASE`
- **Private/Internal**: Prefix with `_` (e.g., `_preserve_float`)

### Import Organization
Organize imports in this order:
1. Standard library imports
2. Third-party imports
3. Local/relative imports

Example from `command_utils.py`:
```python
import argparse
import json
import sys
from typing import Dict, Set, Optional

from .console_utils import console
```

## Key Architectural Patterns

### 1. JSON Precision Preservation

The codebase has a critical pattern for preserving float precision in JSON files. Power BI JSON files require exact float representation.

**Pattern** (in `common.py`):
```python
# Magic prefix for preserving float precision during JSON round-trips
_FLOAT_PRESERVE_PREFIX = "@@__PRESERVE_FLOAT__@@"
_FLOAT_RESTORE_PATTERN = re.compile(...)

def _preserve_float(s: str) -> str:
    """Hook for json.load to preserve float precision as a prefixed string."""
    return _FLOAT_PRESERVE_PREFIX + s

def load_json(file_path: str) -> dict:
    with open(file_path, "r", encoding="utf-8") as file:
        return json.load(file, parse_float=_preserve_float)

def write_json(file_path: str, data: dict) -> None:
    json_str = json.dumps(data, indent=2)
    # Restore preserved floats: remove quotes and magic prefix
    json_str = _FLOAT_RESTORE_PATTERN.sub(r"\1", json_str)
    with open(file_path, "w", encoding="utf-8") as file:
        file.write(json_str)
```

**Rule**: ALWAYS use `common.load_json()` and `common.write_json()` instead of direct `json.load()`/`json.dump()` when working with PBIR files.

### 2. CLI Command Registration Pattern

CLI commands are modularized in the `commands/` directory. Each command module follows this pattern:

```python
# src/pbir_utils/commands/example.py

def register(subparsers):
    """Register the command with argparse."""
    parser = subparsers.add_parser(
        "example",
        help="Short help text",
        description="Detailed description",
        formatter_class=argparse.RawTextHelpFormatter,
    )
    parser.add_argument("arg1", help="Argument help")
    parser.set_defaults(func=handle)

def handle(args):
    """Handle the command execution."""
    # Lazy imports to speed up CLI startup
    from ..common import resolve_report_path
    from ..some_utils import do_something

    report_path = resolve_report_path(args.report_path)
    do_something(report_path)
```

Commands are registered in `commands/__init__.py`:
```python
from . import sanitize, filters, measures, ...

def register_all(subparsers):
    sanitize.register(subparsers)
    filters.register(subparsers)
    measures.register(subparsers)
    # ...
```

**Rule**: Use lazy imports in command handlers to keep CLI startup fast.

### 3. Dry Run Pattern

Most operations support a `--dry-run` flag. This pattern is used throughout:

```python
def some_operation(report_path: str, dry_run: bool = False):
    """Perform operation on report."""
    changes_made = False

    # Perform analysis
    items_to_modify = find_items_to_modify(report_path)

    if items_to_modify:
        changes_made = True
        if not dry_run:
            # Actually make changes
            apply_modifications(items_to_modify)
        else:
            # Just report what would be done
            console.print_info(f"Would modify {len(items_to_modify)} items")

    return changes_made
```

### 4. Console Output Utilities

Use `console_utils.py` for consistent output formatting:

```python
from .console_utils import console

console.print_info("Information message")
console.print_success("Success message")
console.print_warning("Warning message")
console.print_error("Error message")
```

### 5. Report Path Resolution

The CLI supports both explicit paths and auto-detection when running inside a `.Report` folder:

```python
from .common import resolve_report_path

# In CLI command handler:
report_path = resolve_report_path(args.report_path)
# If args.report_path is None and CWD ends with .Report, uses CWD
# Otherwise exits with error if path not provided
```

### 6. YAML-Based Configuration

The sanitization pipeline uses YAML configuration (`defaults/sanitize.yaml`):

**Structure**:
- `definitions`: Named action definitions with descriptions, implementations, and parameters
- `actions`: Default execution order (list of action names)
- `options`: Global options like `dry_run` and `summary`

**Pattern**:
```python
from .sanitize_config import load_config

# Load config (auto-discovers pbir-sanitize.yaml in project or uses defaults)
config = load_config(config_path=args.config, report_path=report_path)

# Get action names in order
action_names = config.get_action_names()
```

### 7. Common CLI Arguments

Use `command_utils.py` helpers for consistency:

```python
from ..command_utils import (
    add_common_args,
    validate_error_on_change,
    check_error_on_change,
)

# In register():
add_common_args(parser, include_report_path=True, include_summary=True)

# In handle():
validate_error_on_change(args)  # Validates --error-on-change requires --dry-run
# ... perform operation ...
check_error_on_change(args, has_changes, "command-name")  # Exits if needed
```

## Testing Conventions

### Test File Organization
- One test file per source module: `test_<module>.py`
- Integration tests in `test_integration.py`
- CLI tests in `test_cli.py`
- Shared fixtures in `conftest.py`

### Fixtures
Common fixtures from `conftest.py`:
- `simple_report`: Creates a temporary PBIR report structure for testing
- `run_cli`: Helper to run CLI commands in tests

### Test Patterns

```python
def test_operation_dry_run(simple_report, run_cli):
    """Test operations should have dry-run tests."""
    result = run_cli(["command", simple_report, "--dry-run"])
    assert result.returncode == 0

def test_operation_with_error(simple_report, run_cli):
    """Test error handling."""
    result = run_cli(["command", simple_report, "--invalid-flag"])
    assert result.returncode != 0
    assert "Error:" in result.stderr
```

### Coverage Requirements
- Minimum 60% coverage enforced by CI
- Focus on core logic and edge cases
- Use `pytest --cov=src --cov-report=html` to identify gaps

## CI/CD Pipeline

### GitHub Actions Workflows

#### CI Workflow (`.github/workflows/ci.yml`)
Runs on push/PR to `main`:
1. Matrix testing across Python 3.10, 3.11, 3.12, 3.13
2. Install dependencies with pip cache
3. Lint with ruff
4. Format check with black
5. Security check with bandit
6. Dependency vulnerability check with pip-audit
7. Run tests with pytest (60% coverage minimum)

**Important**: All checks must pass for PR approval.

#### Publish Workflow (`.github/workflows/python-publish.yml`)
Triggered by GitHub releases to publish to PyPI.

### Pre-commit Checks

Before committing, ensure:
```bash
# Format code
ruff format .

# Check linting
ruff check .

# Run tests with coverage
pytest --cov=src --cov-fail-under=60

# Security check
bandit -r src
```

## Common Development Tasks

### Adding a New CLI Command

1. Create `src/pbir_utils/commands/new_command.py`:
```python
def register(subparsers):
    parser = subparsers.add_parser("new-command", help="...")
    # Add arguments
    parser.set_defaults(func=handle)

def handle(args):
    # Lazy imports
    from ..common import resolve_report_path
    # Implementation
```

2. Register in `src/pbir_utils/commands/__init__.py`:
```python
from . import new_command

def register_all(subparsers):
    # ...
    new_command.register(subparsers)
```

3. Add tests in `tests/test_cli.py` or create `tests/test_new_command.py`

4. Update documentation in `docs/cli.md`

### Adding a New Utility Function

1. Create or update appropriate utility module (e.g., `visual_utils.py`)
2. Export from `src/pbir_utils/__init__.py` if it's a public API
3. Add comprehensive tests in `tests/test_<module>.py`
4. Update documentation in `docs/api.md`

### Adding a New Sanitization Action

1. Implement function in appropriate utility module (or create new one)
2. Add definition to `src/pbir_utils/defaults/sanitize.yaml`:
```yaml
definitions:
  new_action_name:
    description: What this action does
    implementation: function_name  # If different from action name
    params:                        # Optional parameters
      param1: value1
```

3. Optionally add to default `actions` list
4. Add tests in `tests/test_pbir_report_sanitizer.py`

## PBIR File Structure Understanding

### Key PBIR Components

PBIR reports have this structure:
```
MyReport.Report/
├── definition/
│   ├── report.json           # Main report definition
│   └── pages/
│       ├── <page-guid>/
│       │   ├── page.json
│       │   └── visuals/
│       │       └── <visual-guid>/
│       │           └── visual.json
│       └── ...
└── ...
```

### Important JSON Structures

**Measures**: Located in `report.json` under specific paths
**Filters**: Report-level filters in `report.json`
**Pages**: Each page has metadata (name, type, dimensions, display options)
**Visuals**: Each visual has configuration, queries, and interactions
**Bookmarks**: Saved states that may reference pages/visuals

## Error Handling

### Error Reporting Pattern

```python
from .console_utils import console
import sys

def some_function(arg):
    try:
        # Operation
        pass
    except SpecificError as e:
        console.print_error(f"Failed to do something: {e}")
        sys.exit(1)  # For CLI commands
        # or
        raise  # For library functions
```

### Validation Pattern

```python
if not valid_condition:
    console.print_error("Error message explaining the issue")
    sys.exit(1)
```

## Documentation Guidelines

### Updating Documentation

1. **CLI docs** (`docs/cli.md`): Update when adding/changing CLI commands
2. **API docs** (`docs/api.md`): Update when adding/changing public API
3. **README.md**: Update for major features or setup changes
4. **CHANGELOG.md**: Document all user-facing changes
5. **This file** (`CLAUDE.md`): Update when patterns or architecture changes

### Documentation Style
- Use clear, concise language
- Provide code examples
- Include both CLI and Python API examples where applicable
- Use admonitions (tip, warning, note) in MkDocs when helpful

## Git Workflow & Branching

### Branch Naming
- Feature branches: `feature/<description>` or `<username>/<description>`
- Bug fixes: `fix/<description>`
- AI assistant branches: `claude/<description>-<session-id>`

### Commit Messages
- Use clear, descriptive commit messages
- Start with a verb (Add, Update, Fix, Remove, etc.)
- Keep subject line under 72 characters
- Use body for detailed explanations when needed

Example:
```
Add clear-filters command with summary option

- Implement --summary flag for concise output
- Improve slicer type detection to match all variants
- Add tests for new functionality
```

### Pull Request Process

1. Fork/branch from `main`
2. Make changes with tests
3. Ensure all CI checks pass (linting, tests, security)
4. Update documentation
5. Submit PR with clear description
6. Address review feedback

## Common Pitfalls & Solutions

### ❌ Don't: Use standard json.load/dump
```python
# BAD
with open(file_path) as f:
    data = json.load(f)
```

### ✅ Do: Use common.load_json/write_json
```python
# GOOD
from .common import load_json
data = load_json(file_path)
```

### ❌ Don't: Import everything at module level in CLI commands
```python
# BAD - slows CLI startup
from ..common import resolve_report_path
from ..some_utils import do_something

def handle(args):
    # ...
```

### ✅ Do: Use lazy imports in command handlers
```python
# GOOD
def handle(args):
    # Lazy imports
    from ..common import resolve_report_path
    from ..some_utils import do_something
    # ...
```

### ❌ Don't: Use print() for output
```python
# BAD
print("Operation completed")
```

### ✅ Do: Use console utilities
```python
# GOOD
from .console_utils import console
console.print_success("Operation completed")
```

### ❌ Don't: Hard-code paths or assume Windows
```python
# BAD
path = "C:\\Reports\\MyReport.Report"
```

### ✅ Do: Use os.path or pathlib for cross-platform compatibility
```python
# GOOD
import os
path = os.path.join(base_path, "definition", "report.json")
```

## Performance Considerations

1. **Lazy imports**: Keep CLI startup fast by importing only what's needed
2. **JSON parsing**: Use the preserve-float pattern but be mindful it adds overhead
3. **File I/O**: Batch operations when possible
4. **Testing**: Use fixtures to avoid recreating test data repeatedly

## Security Considerations

1. **Path traversal**: Validate and sanitize file paths
2. **Input validation**: Validate JSON structures and arguments
3. **Dependencies**: Use `pip-audit` to check for vulnerabilities
4. **Code scanning**: Use `bandit` for security issues
5. **Secrets**: Never commit credentials or sensitive data (`.gitignore` includes `.env`)

## Versioning

- **Version source**: Git tags (via setuptools-scm)
- **Versioning scheme**: Semantic versioning (MAJOR.MINOR.PATCH)
- **Releases**: Created via GitHub releases, triggers PyPI publish

## Resources

- **Documentation**: https://akhilannan.github.io/pbir-utils/
- **PyPI**: https://pypi.org/project/pbir-utils/
- **GitHub Issues**: https://github.com/akhilannan/pbir-utils/issues
- **Examples**: `examples/example_usage.ipynb`

## Quick Reference Commands

```bash
# Development
pip install -e .[dev,docs]        # Install in dev mode
ruff format .                      # Format code
ruff check .                       # Lint code
pytest --cov=src                   # Run tests with coverage
bandit -r src                      # Security scan
mkdocs serve                       # Serve docs locally

# CLI Usage
pbir-utils --help                  # Show all commands
pbir-utils sanitize --help         # Command-specific help
pbir-utils sanitize path --dry-run # Preview changes

# Building & Publishing (maintainers)
python -m build                    # Build distribution
twine upload dist/*                # Upload to PyPI (automated via GH)
```

## Questions or Issues?

- Check existing issues: https://github.com/akhilannan/pbir-utils/issues
- Review CONTRIBUTING.md for contribution guidelines
- Review CODE_OF_CONDUCT.md for community standards
- Review SECURITY.md for security policy

---

**Last Updated**: 2025-12-18
**For**: AI Assistants working on pbir-utils
