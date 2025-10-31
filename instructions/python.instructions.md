---
description: 'Unified Python coding conventions, quality, and performance guidelines for Python 3.12+'
applyTo: '**/*.py'
---

# Unified Python Guidelines (Python 3.12+)

This document unifies core principles, performance optimizations, quality practices, and general coding conventions for Python 3.12+ development.

---

## 1. Core Principles & General Guidelines

* **Readability**: Always prioritize readability and clarity. Write concise, efficient, and idiomatic code that is easily understandable.
* **DRY (Don't Repeat Yourself)**: Maintain a single source of truth for knowledge.
* **KISS (Keep It Simple, Stupid)**: Avoid needless complexity.
* **YAGNI (You Ain't Gonna Need It)**: Implement only features that are truly necessary.
* **Maintainability**: Write code with good maintainability practices.
    * Include comments on *why* certain design decisions were made.
    * For algorithm-related code, include explanations of the approach used.
* **SOLID Principles**:
    * **SRP (Single Responsibility)**: Modules, classes, and functions should do one thing.
    * **OCP (Open/Closed)**: Software entities should be extendable, not modifiable.
    * **LSP (Liskov Substitution)**: Subtypes must be replaceable for their base types.
    * **ISP (Interface Segregation)**: Use small, specific interfaces.
    * **DIP (Dependency Inversion)**: Depend on abstractions, not concretions.

---

## 2. Code Style & Structure

* **Formatting**: Follow the **PEP 8** style guide.
    * Use 4 spaces for each level of indentation.
    * Ensure lines do not exceed 79 characters.
    * Use blank lines appropriately to separate functions, classes, and code blocks.
* **Naming**: Adhere to PEP 8 naming conventions.
    * `snake_case` for variables and functions.
    * `PascalCase` for classes.
    * `UPPER_CASE` for constants.
* **Structure**:
    * Organize related functionality into logical modules and packages.
    * Keep files reasonably sized (e.g., < 500 lines as a guideline).
    * Follow a logical import order: standard library, third-party, then local modules.
    * Use `__init__.py` files to define clear public APIs for your packages.

---

## 3. Documentation, Comments & Typing

* **Type Hints**:
    * Always use type hints (PEP 484/585/604) for all function parameters and return values.
    * Use modern union types (e.g., `str | int`) and the `typing` module (e.g., `List[str]`, `Dict[str, int]`).
    * Leverage `TypedDict` for dictionary structures and `Protocols` for duck typing.
* **Docstrings**:
    * Provide docstrings following **PEP 257** conventions for all public modules, functions, classes, and methods.
    * Docstrings must document the purpose, parameters (`Args`), return values (`Returns`), and any exceptions raised (`Raises`).
    * Follow a consistent style (e.g., Google, NumPy, or reStructuredText).
    * Place docstrings immediately after the `def` or `class` keyword.
* **Comments**:
    * Write clear and concise comments.
    * Explain the *why*, not the *what*, for complex or non-obvious logic.
    * Keep comments current when changing code.
    * Format `TODO` comments clearly: `# TODO: Brief description [JIRA-123]`.
    * For external dependencies, mention their usage and purpose in comments.

---

## 4. Error Handling & Testing

* **Error Handling**:
    * Handle edge cases and write clear exception handling. Account for empty inputs, invalid data types, and large datasets.
    * Catch *specific* exceptions. Do not use bare `except:`.
    * Use `exception groups` (Python 3.11+) to add context when handling multiple exceptions.
* **Testable Code Design**:
    * Write small, focused functions that are easy to test.
    * Break down complex functions into smaller, more manageable functions.
    * Use dependency injection and isolate side effects to make components testable and easier to mock.
* **Testing**:
    * Always include test cases for the critical paths of the application.
    * Write unit tests for functions and document them with docstrings explaining the test cases.
    * Include comments for edge cases and the expected behavior.

---

## 5. Performance, Quality & Tooling

* **Algorithms & Data Structures**:
    * Choose appropriate data structures for the task (e.g., `dict` for fast lookups, `set` for membership testing).
    * Prefer list/dict/set comprehensions and generators for concise, efficient, and memory-friendly iteration.
    * Use `__slots__` for classes that will be instantiated many times to reduce memory usage.
* **Concurrency**:
    * Use `asyncio` for I/O-bound operations (e.g., network calls, file I/O).
    * Use `concurrent.futures` (especially `ProcessPoolExecutor`) for CPU-bound tasks to leverage multiple cores.
* **Optimization**:
    * Use caching (e.g., `functools.lru_cache`) for expensive, repeated computations.
    * Profile code first (`cProfile`, `timeit`) to identify bottlenecks before optimizing.
* **Static Analysis**:
    * Use **MyPy** for static type checking.
    * Use **Ruff** or Flake8 for linting.
    * Use **Black** for automatic code formatting.
    * Use **isort** for automatic import sorting.
* **Dependency Management**:
    * **Always** use a virtual environment (e.g., uv, venv, poetry).
    * Use **UV** for fast dependency installation and resolution.
    * Keep dependencies explicit in `pyproject.toml` and keep them minimal.