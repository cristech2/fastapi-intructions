---
description: 'FastAPI services layer with business logic separation, data access patterns, and error handling'
applyTo: 'app/services/**/*.py'
---

# FastAPI Business Logic & Services

Guidelines for implementing business logic and data access operations (CRUD)
in the services layer, applying the separation of concerns principle.

This module demonstrates the recommended patterns and practices for:
- Organizing service modules (`app/services/`) with pure business logic
- Implementing asynchronous database operations with proper transaction
  management
- Handling errors in framework-agnostic ways (raising custom exceptions)
- Preventing common issues like N+1 queries through eager loading
- Separating API concerns (HTTP handling) from business logic concerns

## General Instructions

- All business logic and data access resides in `app/services/`
- Services implement pure business logic and database operations (CRUD)
- Services are framework-agnostic: never import FastAPI components
  (`HTTPException`, `Request`, `status`, etc.)
- Services do not manage transactions: only prepare changes in session
  (`db.add()`, `db.delete()`)
- The `get_db_session` dependency solely handles transaction confirmation
  (commit) or rollback
- All database interactions must be asynchronous using `AsyncSession`
- Use framework-agnostic custom exceptions in `app/core/exceptions.py`

## Best Practices

- Validate business logic before database operations
- Use `await asyncio.to_thread()` for CPU-intensive operations to avoid
  blocking the event loop
- Use `await db.flush()` when you need generated IDs before transaction
- Use `await db.refresh()` to reload instances with generated values
- Implement pagination on all collection queries using
  `.offset(skip).limit(limit)`
- Use Eager Loading (`selectinload()`) to prevent N+1 query problems
- Keep exceptions framework-agnostic for complete portability

## Code Standards

### Naming Conventions

| Element | Convention | Example |
| --- | --- | --- |
| Service modules | snake_case | `user_service.py`, `product_service.py` |
| Async functions | verb_noun | `get_user_by_id()`, `create_product()` |
| Variables (database objects) | descriptive snake_case | `existing_user`, `db_prod` |
| Boolean checks | is_/has_ prefix | `is_active`, `has_permission` |

### File Organization

```
app/services/
├── __init__.py
├── user_service.py
├── product_service.py
└── order_service.py
```

- One primary service per main domain entity
- Keep related functions in the same module
- Import models and dependencies at the top
- Separate read (query) and write (create/update/delete) operations logically

## Common Patterns

### Write Operation (Create/Update/Delete)

Use this pattern for operations that modify database state:

```python
# app/services/user_service.py
import asyncio
import app.models.user
import app.core.security
import app.core.exceptions
from sqlalchemy.ext.asyncio import AsyncSession
from sqlmodel import select

async def create_user(
    *, db: AsyncSession, user_in: app.models.user.UserCreate
) -> app.models.user.User:
    """Create a new user with hashed password.

    Args:
        db: Async database session for persistence.
        user_in: User creation data containing email and password.

    Returns:
        The newly created user object.

    Raises:
        EmailAlreadyExistsError: If email already exists in database.
    """
    # 1. Validate business logic (check for duplicates, etc.)
    existing_user = await get_user_by_email(db, email=user_in.email)
    if existing_user:
        # Prevent duplicate emails by raising before DB operation
        raise app.core.exceptions.EmailAlreadyExistsError(
            email=user_in.email
        )

    # 2. Apply business logic (hashing, transformations)
    # Use to_thread for CPU-intensive operations to avoid event loop
    # blocking (password hashing is computationally expensive)
    hashed_password = await asyncio.to_thread(
        app.core.security.hash_password, user_in.password
    )

    # 3. Create ORM model
    db_user = app.models.user.User(
        **user_in.model_dump(exclude={"password"}),
        hashed_password=hashed_password
    )

    # 4. Prepare database operation (no commit)
    # Only db.add() stages the object; transaction control happens in
    # the dependency (get_db_session)
    db.add(db_user)

    # 5. Flush to get generated ID if needed
    # flush() sends commands to DB and gets the generated ID,
    # but does not commit the transaction
    await db.flush()
    await db.refresh(db_user)

    return db_user

async def update_user(
    *, db: AsyncSession, user_id: int, user_in: app.models.user.UserCreate
) -> app.models.user.User:
    """Update an existing user with new data.

    Args:
        db: Async database session for persistence.
        user_id: ID of user to update.
        user_in: User update data.

    Returns:
        The updated user object with new values.

    Raises:
        NotFoundError: If user with given ID does not exist.
    """
    user = await get_user_by_id(db=db, user_id=user_id)

    # Update fields
    update_data = user_in.model_dump(exclude_unset=True)
    user.sqlmodel_update(update_data)

    db.add(user)
    await db.flush()
    await db.refresh(user)

    return user

async def delete_user(*, db: AsyncSession, user_id: int) -> None:
    """Delete a user by ID.

    Args:
        db: Async database session for persistence.
        user_id: ID of user to delete.

    Raises:
        NotFoundError: If user with given ID does not exist.
    """
    user = await get_user_by_id(db=db, user_id=user_id)
    await db.delete(user)
    # No flush or commit needed
```

### Read Operation (Query/Retrieve)

Use this pattern for operations that retrieve data:

```python
import app.models.user
import app.core.exceptions
from sqlalchemy.ext.asyncio import AsyncSession
from sqlmodel import select

async def get_user_by_id(
    *, db: AsyncSession, user_id: int
) -> app.models.user.User:
    """Retrieve a user by primary key ID.

    Args:
        db: Async database session for queries.
        user_id: ID of user to retrieve.

    Returns:
        The user object with matching ID.

    Raises:
        NotFoundError: If no user exists with given ID.
    """
    # Use .get() for primary key queries - most efficient
    user = await db.get(app.models.user.User, user_id)

    if not user:
        raise app.core.exceptions.NotFoundError(
            item_type="User", item_id=str(user_id)
        )

    return user

async def get_user_by_email(
    *, db: AsyncSession, email: str
) -> app.models.user.User | None:
    """Retrieve a user by email address.

    Args:
        db: Async database session for queries.
        email: Email address to search for.

    Returns:
        The user object if found, None otherwise.
    """
    statement = select(app.models.user.User).where(
        app.models.user.User.email == email
    )
    result = await db.execute(statement)
    return result.scalars().first()

async def list_users(
    *, db: AsyncSession, skip: int = 0, limit: int = 10
) -> list[app.models.user.User]:
    """Retrieve a paginated list of users.

    Args:
        db: Async database session for queries.
        skip: Number of records to skip (pagination offset).
        limit: Maximum number of records to return.

    Returns:
        List of user objects within pagination bounds.
    """
    # Always implement pagination
    statement = select(app.models.user.User).offset(skip).limit(limit)
    result = await db.execute(statement)
    return result.scalars().all()

async def get_active_users(
    *, db: AsyncSession, skip: int = 0, limit: int = 10
) -> list[app.models.user.User]:
    """Retrieve a paginated list of active users.

    Args:
        db: Async database session for queries.
        skip: Number of records to skip (pagination offset).
        limit: Maximum number of records to return.

    Returns:
        List of active user objects within pagination bounds.
    """
    statement = (
        select(app.models.user.User)
        .where(app.models.user.User.is_active)
        .offset(skip)
        .limit(limit)
    )
    result = await db.execute(statement)
    return result.scalars().all()
```

### Preventing N+1 Queries with Eager Loading

```python
from sqlalchemy.orm import selectinload
from sqlmodel import select
import app.models.user
import app.core.exceptions
from sqlalchemy.ext.asyncio import AsyncSession

async def get_user_with_orders(
    *, db: AsyncSession, user_id: int
) -> app.models.user.User:
    """Retrieve a user with their orders eagerly loaded.

    Uses selectinload to prevent N+1 query problem when accessing
    the user's orders relationship.

    Args:
        db: Async database session for queries.
        user_id: ID of user to retrieve with orders.

    Returns:
        The user object with orders pre-loaded in one query.

    Raises:
        NotFoundError: If no user exists with given ID.
    """
    # Use selectinload to avoid N+1 queries
    # Instead of loading user then querying orders separately for each
    # user, selectinload fetches orders in a second query but for all
    # users at once (2 queries total vs N+1)
    statement = (
        select(app.models.user.User)
        .where(app.models.user.User.id == user_id)
        .options(selectinload(app.models.user.User.orders))
    )
    result = await db.execute(statement)
    # unique() prevents duplicate rows when joins are used
    user = result.scalars().unique().one_or_none()

    if not user:
        raise app.core.exceptions.NotFoundError(
            item_type="User", item_id=str(user_id)
        )

    return user
```

## Error Handling

### Service Layer (Throws Custom Exceptions)

Services throw only framework-agnostic custom exceptions:

```python
# app/services/user_service.py
import app.models.user
import app.core.exceptions
from sqlalchemy.ext.asyncio import AsyncSession

async def get_user_by_id(
    *, db: AsyncSession, user_id: int
) -> app.models.user.User:
    """Retrieve a user by primary key ID.

    Args:
        db: Async database session for queries.
        user_id: ID of user to retrieve.

    Returns:
        The user object with matching ID.

    Raises:
        NotFoundError: If no user exists with given ID.
    """
    user = await db.get(app.models.user.User, user_id)
    if not user:
        # Correct: Throws custom business exception
        raise app.core.exceptions.NotFoundError(
            item_type="User", item_id=str(user_id)
        )
    return user
```

### API Layer (Catches and Translates)

Endpoints translate business exceptions to HTTP responses:

#### Pattern 1: Global Exception Handler (Recommended)

For common exceptions that always map to the same HTTP status, use the
global exception handler:

```python
# app/api/v1/endpoints/users.py
from typing import Annotated
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
import app.services.user_service
from app.api.dependencies.db import get_db_session

@router.get("/{user_id}", response_model=UserResponse)
async def read_user(
    user_id: int,
    db: Annotated[AsyncSession, Depends(get_db_session)]
):
    """Get user by ID. No try/except needed - global handler catches exceptions."""
    user = await app.services.user_service.get_user_by_id(
        db=db, user_id=user_id
    )
    return user
```

#### Pattern 2: Local Exception Handling

Use when specific exceptions need custom HTTP mappings:

```python
# app/api/v1/endpoints/users.py
from typing import Annotated
from fastapi import APIRouter, Depends, status, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
import app.services.user_service
import app.core.exceptions
from app.api.dependencies.db import get_db_session
import app.models.user

@router.post("/", status_code=status.HTTP_201_CREATED)
async def create_new_user(
    user_in: app.models.user.UserCreate,
    db: Annotated[AsyncSession, Depends(get_db_session)]
):
    """Create new user with custom conflict handling."""
    try:
        user = await app.services.user_service.create_user(
            db=db, user_in=user_in
        )
        return user
    except app.core.exceptions.EmailAlreadyExistsError as e:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail=f"User with email {e.email} already exists."
        )
```

### Custom Exceptions

Define all business exceptions in `app/core/exceptions.py`:

```python
# app/core/exceptions.py

class NotFoundError(LookupError):
    """Raised when a resource is not found.

    Inherits from LookupError (same hierarchy as KeyError, IndexError)
    to indicate that a lookup operation failed.
    """
    def __init__(self, item_type: str, item_id: str):
        """Initialize NotFoundError with resource details.

        Args:
            item_type: Type of resource (e.g., 'User', 'Product').
            item_id: ID of the resource that was not found.
        """
        self.item_type = item_type
        self.item_id = item_id
        super().__init__(f"{item_type} with id {item_id} not found")

class EmailAlreadyExistsError(ValueError):
    """Raised when email already exists.

    Inherits from ValueError to indicate invalid/duplicate value
    passed to operation.
    """
    def __init__(self, email: str):
        """Initialize EmailAlreadyExistsError with duplicate email.

        Args:
            email: Email address that already exists in database.
        """
        self.email = email
        super().__init__(f"User with email {email} already exists")

class InsufficientPermissionError(PermissionError):
    """Raised when user lacks required permission.

    Inherits from PermissionError to indicate access denial based
    on insufficient permissions.
    """
    def __init__(self, permission: str):
        """Initialize InsufficientPermissionError with permission name.

        Args:
            permission: Name of required permission that is missing.
        """
        self.permission = permission
        super().__init__(f"Insufficient permission: {permission}")
```

## Transaction Management

The `get_db_session` dependency in `app/api/dependencies/db.py` must handle
all transaction logic:

```python
# app/api/dependencies/db.py
from typing import AsyncGenerator
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
import app.core.database

async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    """Provide database session with automatic transaction management.

    This dependency handles all ACID transaction semantics. It commits
    successful operations and rolls back on any exception. Services
    should never call commit/rollback directly.

    Yields:
        AsyncSession: Database session for path operation.
    """
    async with app.core.database.async_session_maker() as db:
        try:
            yield db
        except Exception:
            await db.rollback()
            raise
        else:
            await db.commit()
        finally:
            await db.close()
```

**Rules:**

- The dependency is the ONLY place where `await db.commit()` is called
- Services must NEVER call `commit()` or `rollback()`
- Services only call `db.add()`, `db.delete()`, `db.flush()`, `db.refresh()`
- Any exception from services triggers automatic rollback via the dependency
  (keeps database state consistent even when service logic fails)

## Performance Considerations

### Async/Await for I/O Operations

```python
# Correct: Async for database operations
import app.models.user
from sqlalchemy.ext.asyncio import AsyncSession

async def get_user_by_id(
    *, db: AsyncSession, user_id: int
) -> app.models.user.User:
    user = await db.get(app.models.user.User, user_id)
    return user
```

### CPU-Intensive Operations

```python
# Correct: Use to_thread for CPU operations
import asyncio
import app.models.user
import app.core.security
from sqlalchemy.ext.asyncio import AsyncSession

async def create_user(
    *, db: AsyncSession, user_in: app.models.user.UserCreate
) -> app.models.user.User:
    hashed_password = await asyncio.to_thread(
        app.core.security.hash_password, user_in.password
    )
    # Continue with user creation
```

### Pagination on Collections

```python
# Correct: Always paginate collection queries
import app.models.user
from sqlalchemy.ext.asyncio import AsyncSession
from sqlmodel import select

async def list_users(
    *, db: AsyncSession, skip: int = 0, limit: int = 100
) -> list[app.models.user.User]:
    statement = select(app.models.user.User).offset(skip).limit(limit)
    result = await db.execute(statement)
    return result.scalars().all()

# Avoid: Fetching all records without pagination
# Bad: statement = select(app.models.user.User)
```

## Testing

Services should be tested independently of FastAPI:

- Mock the `AsyncSession` using `AsyncMock`
- Test business logic and error conditions
- Verify database operations are prepared (not committed)
- Test exception raising for invalid inputs

```python
# tests/services/test_user_service.py
import pytest
from unittest.mock import AsyncMock
import app.services.user_service
import app.core.exceptions
import app.models.user

@pytest.mark.asyncio
async def test_get_user_by_id_not_found():
    """Test that NotFoundError is raised for non-existent user ID."""
    db = AsyncMock()
    db.get.return_value = None

    with pytest.raises(app.core.exceptions.NotFoundError):
        await app.services.user_service.get_user_by_id(
            db=db, user_id=999
        )

@pytest.mark.asyncio
async def test_create_user_success(mock_db):
    """Test successful user creation with database persistence."""
    user_in = app.models.user.UserCreate(
        email="test@example.com", password="secure123"
    )
    user = await app.services.user_service.create_user(
        db=mock_db, user_in=user_in
    )

    assert mock_db.add.called
    assert mock_db.flush.called
```

## Validation

- Ensure all functions in services are async
- Services import only from `app.models`, `app.core`, `sqlalchemy`, `sqlmodel`
- No FastAPI imports in service files
- All exceptions inherit from `AppException`
- All collection queries implement pagination
- No hardcoded pagination limits without defaults
