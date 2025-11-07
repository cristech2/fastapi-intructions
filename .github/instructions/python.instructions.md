---
description: 'Python coding conventions, quality, and performance guidelines for Python 3.12+'
applyTo: '**/*.py'
---

# Python Guidelines (Python 3.12+)

This document outlines core principles, performance optimizations, quality practices, and general coding conventions for Python 3.12+ development.

---

## 1. Core Principles & General Guidelines
* **Guidelines Principles(PEP 20)**: Follow PEP 20 (The Zen of Python). Prioritize 'Explicit is better than implicit' and 'Readability counts'.
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
    * Ensure lines do not exceed 88 characters.
    * Use blank lines appropriately to separate functions, classes, and code blocks.
* **Imports**:
    * **Import only modules and packages** (e.g., `import os`, `import collections.abc`). NEVER use `from my_pkg import MyClass` or `from my_pkg.my_mod import my_func`. This ensures clarity of namespace and avoids ambiguity.
    * NEVER use relative imports (e.g., `from . import my_sibling_module`). Use absolute imports from the project's top-level directory for clarity and portability.
    * Follow a logical import order: standard library, third-party, then local modules.
    * Use groups separated by blank lines to organize imports visually.
* **Naming**: Adhere to PEP 8 naming conventions.
    * `snake_case` for variables and functions.
    * `PascalCase` for classes.
    * `UPPER_CASE` for constants.
    * Custom exception classes must end with the suffix `Error` (e.g., `class MySpecificError(ValueError):`). This clarifies intent and improves readability.
    * Package directory names must be short, all-lowercase, and should NOT use underscores (e.g., `mypackage`).
    * Module file names (`*.py`) MAY use underscores for clarity (e.g., `my_module.py`).
* **Classes**:
    * Avoid getter and setter methods (e.g., `get_foo()`, `set_foo()`). Use public attributes directly.
    * If logic is required for getting or setting an attribute, use the `@property` and `@foo.setter` decorators for a clean, Pythonic API.
    * Place property definitions and setters immediately after the `__init__` method.
* **Structure**:
    * Organize related functionality into logical modules and packages.
    * Keep files reasonably sized (e.g., < 500 lines as a guideline).
    * Use `__init__.py` files to define clear public APIs for your packages.

---

## 3. Documentation, Comments & Typing

**Type Hints:**
- Always use type hints for all function parameters, variables, and return values (PEP 484).
- Use modern union types with the `|` operator (PEP 604), not `typing.Union`.
    - ✅ Correct: `str | int`
    - ❌ Incorrect: `Union[str, int]`
- Use `T | None` for optional types, not `typing.Optional`.
    - ✅ Correct: `str | None`
    - ❌ Incorrect: `Optional[str]`
- Prefer native generics for collections (PEP 585, Python 3.9+) instead of importing from `typing`.
    - ✅ Correct: `list[str]`, `dict[str, int]`, `set[int]`
    - ❌ Incorrect: `List[str]`, `Dict[str]`, `Set[int]`
- For fixed-shape dictionaries, use `TypedDict` (PEP 589).
- For duck typing, use `typing.Protocol` (PEP 544) to define structural interfaces.
- Avoid using `typing.Any` unless strictly necessary and justified with a comment.

**Docstrings:**
**Note:**
Use docstrings and comments for documentation intended for humans.
`Annotated` should only be used when metadata is consumed by tools (e.g., Pydantic, FastAPI) for validation, documentation, or autocompletion.
Do not use `Annotated` for descriptive comments—prefer docstrings or inline comments instead.
- Provide docstrings following **PEP 257** conventions for all public modules, functions, classes, and methods.
- Docstrings must document the purpose, parameters (`Args`), return values (`Returns`), and any exceptions raised (`Raises`).
- Follow a consistent style (e.g., Google, NumPy, or reStructuredText).
- Place docstrings immediately after the `def` or `class` keyword.

**Comments:**
- Write clear and concise comments.
- Explain the *why*, not the *what*, for complex or non-obvious logic.
- Keep comments current when changing code.
- Format `TODO` comments as `# TODO(username): Brief description` to enable accountability and easy searching.
- For external dependencies, mention their usage and purpose in comments.

**Executable Scripts:**
- Scripts must define a `def main(argv: list[str] | None = None) -> int | None:` function.
- Parse command-line arguments inside `main()` from `sys.argv[1:]` if `argv` is `None`.
- Call `main()` only inside the `if __name__ == "__main__":` block.
- This pattern promotes code reusability and testability (you can import the module and call `main(test_args)` without executing the script).

### Annotated usage (PEP 593)

**When and How to Use Annotated:**
- Use `Annotated` in Pydantic models (not SQLModel with `table=True`) when metadata provides real value for validation, automatic documentation (e.g., FastAPI), or framework autocompletion.
- Purpose: Add validators, examples, query metadata, etc., that Pydantic, FastAPI, or other tools will consume.

**Examples:**

```python
from typing import Annotated
from pydantic import BaseModel, Field
from fastapi import Query

class Item(BaseModel):
    # 'Annotated' adds validation (max_length) used by Pydantic
    name: Annotated[str, Field(max_length=50)]
    
    # 'Annotated' adds metadata for API documentation
    price: Annotated[float, Field(gt=0, description="Price must be positive")]

# 'Annotated' defines how FastAPI should treat the parameter
async def read_items(q: Annotated[str | None, Query(max_length=30)] = None):
    ...
```

**Prohibited: Table Models (SQLModel / SQLAlchemy)**
- Do NOT use `Annotated` in fields of models representing database tables (e.g., SQLModel with `table=True` or SQLAlchemy base classes).
- Reason: Causes known conflicts with Alembic and automatic migration generation.

```python
# ❌ INCORRECT (Causes Alembic issues)
class User(SQLModel, table=True):
    id: Annotated[int | None, Field(default=None, primary_key=True)]
    name: str

# ✅ CORRECT (For table models)
class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
```

*Note: You may use Annotated in derived Pydantic models ("Read", "Create", etc.) that do not define the table.*

**Avoid Cosmetic Use**
- Do NOT use `Annotated` just to add descriptive comments not consumed by a tool.
- Reason: Adds visual noise without functional benefit. Use comments (`#`) or docstrings for human documentation.

```python
# ❌ Incorrect (Unnecessary)
user_id: Annotated[int, "User ID"]

# ✅ Preferable
user_id: int  # User ID
```

**Warning:** Using Annotated incorrectly in table models will break Alembic migrations and database schema generation.

---

## 4. Error Handling & Testing

* **Error Handling**:
    * Handle edge cases and write clear exception handling. Account for empty inputs, invalid data types, and large datasets.
    * Catch *specific* exceptions. Do not use bare `except:`.
    * Use `exception groups` (Python 3.11+) to add context when handling multiple exceptions.
    * NEVER use `assert` to validate public function arguments or API inputs. `assert` can be disabled globally with `python -O`, eliminating validation entirely.
    * Use `raise ValueError()` or `raise TypeError()` for invalid API inputs instead.
    * Use `assert` only for internal correctness checks that should not be disabled (e.g., invariants in private helper functions).
* **Testable Code Design**:
    * Write small, focused functions that are easy to test.
    * Break down complex functions into smaller, more manageable functions.
    * Use dependency injection and isolate side effects to make components testable and easier to mock.
    * NEVER use mutable objects as default parameter values (e.g., `my_list: list = []` or `my_dict: dict = {}`). Use `my_list: list[str] | None = None` and initialize inside the function: `if my_list is None: my_list = []`.
    * Use comprehensions (list, dict) and generator expressions for simple data transformations only.
    * If the logic requires more than one `if` condition, multiple lines, or side effects (like `print()`), use a standard `for` loop for readability and maintainability.
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
    * Use **Ruff** as the central linting and code formatting tool. Ruff consolidates:
        * Linting (replaces Flake8)
        * Automatic code formatting (replaces Black)
        * Automatic import sorting (replaces isort)
    * Configure Ruff in `pyproject.toml` to enforce all coding standards in one place.
    * Use **Bandit** for security analysis. Scan for common security issues (e.g., hardcoded credentials, insecure function calls, SQL injection vulnerabilities).
* **Dependency Management**:
    * **Always** use a virtual environment (e.g., uv, venv, poetry).
    * Use **UV** for fast dependency installation and resolution.
    * Keep dependencies explicit in `pyproject.toml` and keep them minimal.