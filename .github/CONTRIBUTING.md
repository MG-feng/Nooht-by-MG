
# ============================================================
# 2. .github/CONTRIBUTING.md — Contribution guidelines
# ============================================================

contributing_md = r'''# Contributing to Nooht Framework

Thank you for your interest in contributing to Nooht! This document outlines the process, standards, and expectations for all contributors.

---

## Table of Contents

1. [Code of Conduct](#code-of-conduct)
2. [Getting Started](#getting-started)
3. [Development Workflow](#development-workflow)
4. [Commit Message Convention](#commit-message-convention)
5. [Pull Request Process](#pull-request-process)
6. [Coding Standards](#coding-standards)
7. [Testing Requirements](#testing-requirements)
8. [Architecture Rules](#architecture-rules)
9. [Prohibited Changes](#prohibited-changes)
10. [Review Process](#review-process)
11. [Release Cycle](#release-cycle)

---

## Code of Conduct

All contributors must adhere to the following principles:

- **Respect** — Treat all community members with professionalism and courtesy.
- **Transparency** — Document your changes clearly. Undocumented code will be rejected.
- **Quality** — Do not submit "quick fixes" that compromise long-term maintainability.
- **Compatibility** — Changes must not break the model-agnostic design principle.

Harassment, trolling, or discriminatory behavior of any kind will result in immediate removal from the project.

---

## Getting Started

### Prerequisites

- Python 3.9 or higher
- Git
- `duckdb` (optional, for DuckDB backend development)
- `faiss-cpu` (optional, for vector retrieval development)

### Setup

```bash
# Fork the repository
git clone https://github.com/YOUR_USERNAME/Nooht.git
cd Nooht

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\\Scripts\\activate

# Install in development mode
pip install -e ".[dev]"

# Run tests
pytest tests/ -v
```

---

## Development Workflow

### 1. Branch Naming

All branches must follow this convention:

```
type/short-description
```

| Type | Purpose | Example |
|------|---------|---------|
| `feat/` | New feature | `feat/hmc-archive-policy` |
| `fix/` | Bug fix | `fix/duckdb-thread-safety` |
| `docs/` | Documentation | `docs/api-reference-update` |
| `refactor/` | Code refactoring | `refactor/symbol-memory-index` |
| `test/` | Test additions | `test/backend-stress-suite` |
| `perf/` | Performance improvement | `perf/faiss-batch-search` |

### 2. Before You Start

1. Check existing [Issues](https://github.com/MG-feng/Nooht/issues) and [Discussions](https://github.com/MG-feng/Nooht/discussions).
2. If your change is significant, open an **Issue** first to discuss the design.
3. Wait for at least one maintainer approval before beginning implementation.

### 3. Development Rules

- **Never modify frozen modules without Architecture Review approval.**
- All new features must include:
  - Unit tests (`tests/test_*.py`)
  - Type hints
  - Docstrings (Google style)
  - Updated API documentation (`api-document/`)

---

## Commit Message Convention

We follow [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): subject

[optional body]

[optional footer]
```

### Types

| Type | Description |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Code style (formatting, no logic change) |
| `refactor` | Code refactoring |
| `perf` | Performance improvement |
| `test` | Adding or fixing tests |
| `chore` | Build process, dependencies, etc. |

### Examples

```
feat(hmc): add L3 archive compression policy

Implements automatic archiving for memories older than 90 days.
Archive stores only semantic_hash and name to minimize footprint.

Closes #42
```

```
fix(duckdb): resolve thread-local connection leak

Connections were not properly closed when threads exited,
leading to file descriptor exhaustion under high load.

Fixes #38
```

---

## Pull Request Process

### 1. PR Title Format

```
[type] scope: Brief description
```

Example: `[feat] hmc: Add configurable compression thresholds`

### 2. PR Description Template

```markdown
## Summary
<!-- One-line description of the change -->

## Motivation
<!-- Why is this change needed? Link to related issues. -->

## Changes
<!-- Bullet list of what changed -->

## Testing
<!-- How was this tested? Include test commands and results. -->

## Backward Compatibility
<!-- Does this break existing APIs? If yes, list migration steps. -->

## Checklist
- [ ] Unit tests pass (`pytest tests/`)
- [ ] Type hints added
- [ ] Docstrings updated
- [ ] API documentation updated (`api-document/`)
- [ ] No prohibited changes (see below)
- [ ] Architecture Review approved (if touching frozen modules)
```

### 3. PR Requirements

- **Minimum 2 approving reviews** before merge
- **All CI checks must pass**
- **No merge conflicts**
- **Squash commits** on merge

---

## Coding Standards

### Python Style

- Follow [PEP 8](https://pep8.org/)
- Maximum line length: **100 characters**
- Use **black** for formatting: `black nooht/ tests/`
- Use **isort** for imports: `isort nooht/ tests/`

### Type Hints

All public functions must have type hints:

```python
def query(
    self,
    name: Optional[str] = None,
    type: Optional[SymbolType] = None,
    limit: int = 100,
) -> List[SymbolEntity]:
    """Multi-condition query with pagination support."""
    ...
```

### Docstrings

Use Google-style docstrings:

```python
def compress(self, symbol: SymbolEntity, target_level: MemoryLevel) -> CompressedMemory:
    """Compress a symbol to the target memory level.
    
    Args:
        symbol: The symbol entity to compress.
        target_level: Target compression level (L1-L3).
    
    Returns:
        CompressedMemory: The compressed memory unit.
    
    Raises:
        ValueError: If target_level is unknown.
    """
```

---

## Testing Requirements

### Minimum Coverage

- **Core modules** (`memory/`, `compression/`, `semantic/`): **≥ 90%**
- **Adapter layer** (`adapters/`): **≥ 80%**
- **Context & retrieval**: **≥ 75%**

### Test Categories

| Category | Purpose | Location |
|----------|---------|----------|
| Unit tests | Individual function correctness | `tests/unit/` |
| Integration tests | Module interaction | `tests/integration/` |
| Stress tests | Large-scale performance | `tests/stress/` |
| Fidelity tests | HMC/SCM quality preservation | `tests/fidelity/` |

### Running Tests

```bash
# All tests
pytest tests/ -v --cov=nooht

# Specific module
pytest tests/unit/test_memory.py -v

# With coverage report
pytest tests/ --cov=nooht --cov-report=html
```

---

## Architecture Rules

### 1. Model-Agnostic Principle

**Nooht must never bind to a specific LLM.**

- ❌ Hard-coding model names (e.g., "Qwen", "DeepSeek", "Llama")
- ❌ Importing `transformers` or `torch` in core modules
- ❌ Model-specific tokenization logic in `memory/` or `compression/`

✅ All model interaction goes through `adapters/`

### 2. Storage Abstraction

- Core logic must work with any `MemoryBackend` implementation
- Never assume `InMemoryBackend` or `DuckDBBackend` specifically
- New backends must implement the full `MemoryBackend` interface

### 3. Compression > Deletion

The **Memory Preservation Principle** is non-negotiable:

```
Compress → Re-compress → Archive → Delete (last resort)
```

Any change that violates this principle will be **rejected**.

---

## Prohibited Changes

The following changes are **strictly forbidden** without explicit Architecture Board approval:

| Category | Prohibition | Rationale |
|----------|-------------|-----------|
| **Model Binding** | Adding model-specific code to core modules | Breaks model-agnostic design |
| **Training Code** | LoRA, QLoRA, Trainer, Dataset classes | Nooht is infrastructure, not a training framework |
| **Tokenizer** | Custom tokenizer implementations | Out of scope for v0.1 |
| **Benchmark** | HumanEval, MBPP, LiveCodeBench runners | Separate benchmark repository |
| **Frozen Modules** | Modifying `symbol_entity.py`, `backend.py` architecture | Architecture stability |
| **HMC Principle** | Changing "Compression > Deletion" order | Core design invariant |

**Frozen Modules (v0.1):**
- `nooht/memory/symbol_entity.py`
- `nooht/memory/backend.py` (architecture, not bugfixes)
- `nooht/compression/hmc.py` (compression policy)
- `nooht/semantic/scm.py` (semantic structure)

---

## Review Process

### Reviewer Assignment

| Module | Primary Reviewer | Secondary Reviewer |
|--------|------------------|-------------------|
| `memory/` | DeepSeek V4 | Qwen 3.7 |
| `compression/` | DeepSeek V4 | Qwen 3.7 |
| `semantic/` | Qwen 3.7 | DeepSeek V4 |
| `retrieval/` | Qwen 3.7 | Kimi 2.6 |
| `context/` | DeepSeek V4 | ChatGPT 5.5 |
| `adapters/` | ChatGPT 5.5 | DeepSeek V4 |
| `docs/` | Qwen-Coder | Kimi 2.6 |

### Review Criteria

1. **Correctness** — Does the code do what it claims?
2. **Performance** — Any O(N²) or worse algorithms in hot paths?
3. **Security** — SQL injection, path traversal, unsafe deserialization?
4. **Compatibility** — Does it break existing APIs?
5. **Documentation** — Are docstrings and API docs updated?

---

## Release Cycle

| Phase | Trigger | Actions |
|-------|---------|---------|
| **Alpha** | Architecture approved | Feature development, API may change |
| **Beta** | All core modules frozen | Bug fixes only, API stable |
| **RC** | All tests pass | Final validation, no new features |
| **Release** | RC approved + 1 week | Tag, publish to PyPI |

### Versioning

We follow [Semantic Versioning](https://semver.org/):

```
MAJOR.MINOR.PATCH
```

- **MAJOR**: Breaking API changes
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes only

---

## Questions?

- **General questions:** [Discussions](https://github.com/MG-feng/Nooht/discussions)
- **Bug reports:** [Issues](https://github.com/MG-feng/Nooht/issues)
- **Security issues:** See [SECURITY.md](SECURITY.md)

---

<p align="center"><i>Every contribution makes Nooht smarter. Thank you.</i></p>
'''
