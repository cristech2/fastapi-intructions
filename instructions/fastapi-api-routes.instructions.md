---
description: 'FastAPI API routes and endpoints best practices, including router organization, RESTful design, path operations, and error handling patterns'
applyTo: '**/*router*.py'
---

# FastAPI API Routes and Endpoints

Guidelines for creating well-organized, maintainable API endpoints using FastAPI's `APIRouter` with proper RESTful design patterns, error handling, and async best practices.

## General Instructions

- Organize endpoints into modular routers using `APIRouter` for large applications
- Always use descriptive names for resources following RESTful conventions (plural nouns for collections)
- Delegate business logic to service layer; endpoints should focus on HTTP concerns
- Use `async def` for all route handlers with I/O-bound operations
- Implement proper error handling using `HTTPException` and custom exception handlers
- Document endpoints with `summary`, `description`, and `tags` for OpenAPI clarity

## Router Organization

### Modular Structure

Organize routers into a clean hierarchy that reflects API versioning and resource grouping:

```python
# app/api/v1/api.py - Main router aggregator
import fastapi

import app.api.v1.endpoints.users as users
import app.api.v1.endpoints.items as items
import app.api.v1.endpoints.products as products

api_router = fastapi.APIRouter()

api_router.include_router(users.router, prefix="/users", tags=["Users"])
api_router.include_router(items.router, prefix="/items", tags=["Items"])
api_router.include_router(products.router, prefix="/products", tags=["Products"])
```

### Router Configuration

```python
# app/api/v1/endpoints/users.py
import fastapi

router = fastapi.APIRouter(
    prefix="/users",           # URL prefix for all routes
    tags=["Users"],            # Group in OpenAPI documentation
    responses={404: {"description": "Resource not found"}}  # Common responses
)
```

## Path Operations (RESTful Design)

### Core RESTful Principles

| Principle | Implementation |
| --------- | --------------- |
| **Resource Names** | Use plural nouns (e.g., `/users`, `/items`, not `/user`, `/item`) |
| **HTTP Methods** | GET (retrieve), POST (create), PUT (replace), PATCH (update), DELETE (remove) |
| **Status Codes** | Use appropriate codes: 200 OK, 201 Created, 204 No Content, 404 Not Found, 409 Conflict |
| **Response Models** | Always specify `response_model` for validation and preventing data leaks |
| **Consistency** | Follow the same patterns across all endpoints |

### Standard CRUD Operations

```python
import typing

import fastapi
import sqlalchemy.ext.asyncio

import app.api.dependencies.db as db_dependencies
import app.models.schemas.user as user_schemas
import app.services.user_service as user_service

router = fastapi.APIRouter()

# CREATE - Post single resource
@router.post(
    "/",
    response_model=user_schemas.UserRead,
    status_code=fastapi.status.HTTP_201_CREATED,
    summary="Create a new user",
    description="Register a new user in the system.",
)
async def create_user(
    user_in: user_schemas.UserCreate,
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(db_dependencies.get_db_session)
    ]
) -> user_schemas.UserRead:
    """Create a new user with validation and error handling."""
    existing_user = await user_service.get_user_by_email(
        db, email=user_in.email
    )
    if existing_user:
        raise fastapi.HTTPException(
            status_code=fastapi.status.HTTP_409_CONFLICT,
            detail="A user with this email already exists.",
        )
    return await user_service.create_user(db=db, user_in=user_in)

# READ - Get all resources with pagination
@router.get(
    "/",
    response_model=list[user_schemas.UserRead],
    status_code=fastapi.status.HTTP_200_OK,
    summary="List all users",
    description="Retrieve a paginated list of all users."
)
async def list_users(
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(db_dependencies.get_db_session)
    ],
    skip: typing.Annotated[
        int, fastapi.Query(ge=0, description="Items to skip")
    ] = 0,
    limit: typing.Annotated[
        int, fastapi.Query(ge=1, le=1000, description="Items per page")
    ] = 100
) -> list[user_schemas.UserRead]:
    """Get paginated list of users."""
    return await user_service.get_users(db, skip=skip, limit=limit)

# READ - Get single resource by ID
@router.get(
    "/{user_id}",
    response_model=user_schemas.UserRead,
    status_code=fastapi.status.HTTP_200_OK,
    summary="Get user by ID",
    description="Retrieve a specific user by their ID."
)
async def get_user(
    user_id: int,
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(db_dependencies.get_db_session)
    ]
) -> user_schemas.UserRead:
    """Get a single user by ID."""
    user = await user_service.get_user_by_id(db, user_id=user_id)
    if not user:
        raise fastapi.HTTPException(
            status_code=fastapi.status.HTTP_404_NOT_FOUND,
            detail=f"User with ID {user_id} not found."
        )
    return user

# UPDATE - Full update with PUT
@router.put(
    "/{user_id}",
    response_model=user_schemas.UserRead,
    status_code=fastapi.status.HTTP_200_OK,
    summary="Update entire user",
    description="Replace all fields of an existing user."
)
async def update_user_full(
    user_id: int,
    user_update: user_schemas.UserUpdate,
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(db_dependencies.get_db_session)
    ]
) -> user_schemas.UserRead:
    """Fully update a user (replace all fields)."""
    user = await user_service.get_user_by_id(db, user_id=user_id)
    if not user:
        raise fastapi.HTTPException(
            status_code=fastapi.status.HTTP_404_NOT_FOUND,
            detail=f"User with ID {user_id} not found."
        )
    return await user_service.update_user(
        db, user=user, user_update=user_update
    )

# UPDATE - Partial update with PATCH
@router.patch(
    "/{user_id}",
    response_model=user_schemas.UserRead,
    status_code=fastapi.status.HTTP_200_OK,
    summary="Partially update user",
    description="Update specific fields of an existing user."
)
async def update_user_partial(
    user_id: int,
    user_update: user_schemas.UserUpdate,
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(db_dependencies.get_db_session)
    ]
) -> user_schemas.UserRead:
    """Partially update a user (only provided fields)."""
    user = await user_service.get_user_by_id(db, user_id=user_id)
    if not user:
        raise fastapi.HTTPException(
            status_code=fastapi.status.HTTP_404_NOT_FOUND,
            detail=f"User with ID {user_id} not found."
        )
    return await user_service.update_user(
        db, user=user, user_update=user_update, partial=True
    )

# DELETE - Delete resource
@router.delete(
    "/{user_id}",
    status_code=fastapi.status.HTTP_204_NO_CONTENT,
    summary="Delete a user",
    description="Permanently delete a user from the system."
)
async def delete_user(
    user_id: int,
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(db_dependencies.get_db_session)
    ]
) -> None:
    """Delete a user by ID."""
    user = await user_service.get_user_by_id(db, user_id=user_id)
    if not user:
        raise fastapi.HTTPException(
            status_code=fastapi.status.HTTP_404_NOT_FOUND,
            detail=f"User with ID {user_id} not found."
        )
    await user_service.delete_user(db, user=user)
```

## Dependency Injection and Annotated

Use `Annotated` with `Depends()` for clean, modern dependency injection:

```python
import typing

import fastapi
import sqlalchemy.ext.asyncio

# Database dependency
async def get_db_session() -> typing.AsyncGenerator[
    sqlalchemy.ext.asyncio.AsyncSession, None
]:
    """
    Provide database session for route handlers.

    Yields:
        AsyncSession: Database session for query execution.
    """
    async with AsyncSessionLocal() as session:
        yield session

# Authentication dependency
async def get_current_user(
    token: str = fastapi.Depends(oauth2_scheme),
    db: sqlalchemy.ext.asyncio.AsyncSession = fastapi.Depends(
        get_db_session
    )
) -> User:
    """
    Authenticate and return current user.

    Args:
        token: OAuth2 token from request header.
        db: Database session for verification.

    Returns:
        User: Authenticated user object.

    Raises:
        HTTPException: If token is invalid or user not found.
    """
    user = await verify_token(token, db)
    if not user:
        raise fastapi.HTTPException(
            status_code=401, detail="Invalid credentials"
        )
    return user

# Using Annotated in route handlers
@router.get("/{item_id}", response_model=ItemRead)
async def get_item(
    item_id: int,
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(get_db_session)
    ],
    current_user: typing.Annotated[User, fastapi.Depends(get_current_user)],
    include_related: typing.Annotated[
        bool, fastapi.Query(description="Include related items")
    ] = False
) -> ItemRead:
    """
    Get item with optional related data.

    Args:
        item_id: ID of the item to retrieve.
        db: Database session.
        current_user: Authenticated user performing the request.
        include_related: Whether to include related items.

    Returns:
        ItemRead: Item object, visible only to owner.

    Raises:
        HTTPException: If item not found or user lacks permission.
    """
    item = await item_service.get_item(db, item_id)
    if not item or item.owner_id != current_user.id:
        raise fastapi.HTTPException(status_code=404, detail="Item not found")
    return item
```

## Query Parameters and Pagination

Implement advanced filtering and pagination using proper validation:

```python
import typing

import fastapi
import sqlalchemy.ext.asyncio

import app.services.product_service as product_service

@router.get("/products/", response_model=list[ProductSchema])
async def list_products(
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(get_db_session)
    ],
    # Pagination
    skip: typing.Annotated[
        int,
        fastapi.Query(ge=0, description="Number of items to skip")
    ] = 0,
    limit: typing.Annotated[
        int,
        fastapi.Query(ge=1, le=1000, description="Maximum items per page")
    ] = 100,
    # Filtering
    category: typing.Annotated[
        str | None, fastapi.Query(description="Filter by category")
    ] = None,
    min_price: typing.Annotated[
        float | None, fastapi.Query(ge=0, description="Minimum price")
    ] = None,
    max_price: typing.Annotated[
        float | None, fastapi.Query(ge=0, description="Maximum price")
    ] = None,
    # Sorting
    sort_by: typing.Annotated[
        str, fastapi.Query(description="Field to sort by")
    ] = "id",
    sort_order: typing.Annotated[
        str, fastapi.Query(regex="^(asc|desc)$", description="Sort order")
    ] = "asc",
    # Search
    search: typing.Annotated[
        str | None,
        fastapi.Query(
            min_length=1, max_length=100, description="Search term"
        )
    ] = None,
) -> list[ProductSchema]:
    """
    List products with advanced filtering, sorting, and pagination.

    Query Parameters:
    - `skip`: Number of items to skip (default: 0)
    - `limit`: Items per page, max 1000 (default: 100)
    - `category`: Filter by product category
    - `min_price`, `max_price`: Filter by price range
    - `sort_by`: Field to sort results
    - `sort_order`: Sort direction (asc or desc)
    - `search`: Search term for product name/description

    Returns:
        list[ProductSchema]: Filtered and paginated products.
    """
    products = await product_service.get_filtered_products(
        db,
        category=category,
        min_price=min_price,
        max_price=max_price,
        sort_by=sort_by,
        sort_order=sort_order,
        search=search,
        skip=skip,
        limit=limit
    )
    return products
```

## Error Handling

Raise `HTTPException` for API errors instead of returning error responses:

```python
import typing

import fastapi
import sqlalchemy.ext.asyncio

import app.core.exceptions as core_exceptions
import app.services.user_service as user_service

# Good Example - Proper error handling
@router.post(
    "/users/",
    response_model=UserRead,
    status_code=fastapi.status.HTTP_201_CREATED
)
async def create_user(
    user_in: UserCreate,
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(get_db_session)
    ]
) -> UserRead:
    """
    Create user with proper validation.

    Args:
        user_in: User creation data.
        db: Database session.

    Returns:
        UserRead: Created user object.

    Raises:
        HTTPException: If email already registered (409) or
                      database error occurs (500).
    """
    try:
        existing_user = await user_service.get_user_by_email(
            db, user_in.email
        )
        if existing_user:
            raise fastapi.HTTPException(
                status_code=fastapi.status.HTTP_409_CONFLICT,
                detail="Email already registered",
                headers={"X-Error": "duplicate_email"}
            )
        return await user_service.create_user(db, user_in)
    except core_exceptions.DatabaseError:
        raise fastapi.HTTPException(
            status_code=fastapi.status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Database error occurred"
        )

# Bad Example - Do NOT do this
@router.post("/users/")
async def create_user_bad(user_in: UserCreate) -> dict[str, typing.Any]:
    """
    Bad example - avoid these patterns.

    ❌ Missing response_model
    ❌ Catching generic Exception
    ❌ Returning error response instead of raising
    """
    # This is intentionally bad to show what NOT to do
    pass
```

## Documentation Standards

### Endpoint Documentation

Include complete documentation in route decorators:

```python
import typing

import fastapi
import sqlalchemy.ext.asyncio

@router.post(
    "/users/{user_id}/assign-role",
    response_model=UserRead,
    status_code=fastapi.status.HTTP_200_OK,
    summary="Assign role to user",
    description=(
        "Assign a specific role to an existing user. "
        "Only administrators can perform this action."
    ),
    tags=["User Management"],
    responses={
        200: {"description": "Role successfully assigned"},
        403: {"description": "Insufficient permissions"},
        404: {"description": "User not found"},
        409: {"description": "User already has this role"}
    }
)
async def assign_user_role(
    user_id: int,
    role: RoleAssignment,
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(get_db_session)
    ],
    current_user: typing.Annotated[User, fastapi.Depends(get_admin_user)]
) -> UserRead:
    """
    Assign a role to an existing user.

    This endpoint allows administrators to assign specific roles to users.
    Users can have multiple roles. Assigning a role that already exists
    will raise a 409 Conflict error.

    Args:
        user_id: ID of the user to assign role to
        role: Role assignment details
        db: Database session
        current_user: Current authenticated user (must be admin)

    Returns:
        Updated user object with new role assignment

    Raises:
        HTTPException: If user not found (404), insufficient
                      permissions (403), or role already
                      assigned (409)
    """
    # Implementation would go here
    pass
```

## Common Patterns

### Nested Resources

Organize nested resources with clear hierarchies:

```python
import typing

import fastapi
import sqlalchemy.ext.asyncio

import app.services.user_service as user_service
import app.services.item_service as item_service

# app/api/v1/endpoints/user_items.py
@router.get(
    "/users/{user_id}/items",
    response_model=list[ItemRead],
    summary="List user's items"
)
async def list_user_items(
    user_id: int,
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(get_db_session)
    ],
    skip: typing.Annotated[int, fastapi.Query(ge=0)] = 0,
    limit: typing.Annotated[int, fastapi.Query(ge=1, le=100)] = 10
) -> list[ItemRead]:
    """
    Get all items belonging to a specific user.

    Args:
        user_id: ID of the user
        db: Database session
        skip: Number of items to skip
        limit: Maximum items to return

    Returns:
        list[ItemRead]: Items owned by the user

    Raises:
        HTTPException: If user not found (404)
    """
    user = await user_service.get_user_by_id(db, user_id)
    if not user:
        raise fastapi.HTTPException(
            status_code=404, detail="User not found"
        )
    return await item_service.get_user_items(db, user_id, skip, limit)

@router.post(
    "/users/{user_id}/items",
    response_model=ItemRead,
    status_code=fastapi.status.HTTP_201_CREATED,
    summary="Create item for user"
)
async def create_user_item(
    user_id: int,
    item_in: ItemCreate,
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(get_db_session)
    ]
) -> ItemRead:
    """
    Create a new item for a specific user.

    Args:
        user_id: ID of the user
        item_in: Item creation data
        db: Database session

    Returns:
        ItemRead: Created item object

    Raises:
        HTTPException: If user not found (404)
    """
    user = await user_service.get_user_by_id(db, user_id)
    if not user:
        raise fastapi.HTTPException(
            status_code=404, detail="User not found"
        )
    return await item_service.create_item(db, item_in, user_id)
```

### Bulk Operations

For bulk operations, return appropriate summary responses:

```python
import typing

import fastapi
import pydantic
import sqlalchemy.ext.asyncio

import app.core.exceptions as core_exceptions
import app.services.user_service as user_service

class BulkResult(pydantic.BaseModel):
    """Result summary for bulk operation."""

    total: int
    processed: int
    failed: int
    errors: list[dict[str, typing.Any]] = pydantic.Field(default_factory=list)

@router.post(
    "/users/bulk-import",
    response_model=BulkResult,
    status_code=fastapi.status.HTTP_200_OK,
    summary="Bulk import users"
)
async def bulk_import_users(
    users_data: list[UserCreate],
    db: typing.Annotated[
        sqlalchemy.ext.asyncio.AsyncSession,
        fastapi.Depends(get_db_session)
    ]
) -> BulkResult:
    """
    Import multiple users in a single request.

    Processes each user and collects errors for failed imports.
    Partial failures are acceptable; operation returns status
    for all items.

    Args:
        users_data: List of users to import
        db: Database session

    Returns:
        BulkResult: Summary with total, processed, failed counts
                   and detailed error list
    """
    result = BulkResult(total=len(users_data), processed=0, failed=0)
    for user_data in users_data:
        try:
            await user_service.create_user(db, user_data)
            result.processed += 1
        except core_exceptions.DuplicateUserError as exc:
            result.failed += 1
            result.errors.append({
                "email": user_data.email,
                "error": str(exc)
            })
        except core_exceptions.ValidationError as exc:
            result.failed += 1
            result.errors.append({
                "email": user_data.email,
                "error": str(exc)
            })
    return result
```

## Best Practices

- **Always use `async def`** for route handlers with I/O operations
- **Specify `response_model`** to validate responses and prevent data leaks
- **Set `status_code`** explicitly for the happy path
- **Use `Annotated` with `Query`, `Path`, `Body`** for modern parameter annotation
- **Validate at the boundary** using Pydantic models
- **Delegate business logic** to the service layer
- **Use consistent naming** across all endpoints (plurals, descriptive)
- **Document thoroughly** with `summary`, `description`, `tags`
- **Handle errors gracefully** by raising `HTTPException`
- **Implement pagination** for list endpoints to manage large datasets
- **Version your API** using URL prefixes (`/api/v1/`, `/api/v2/`)

## Validation and Verification

- Test endpoints: `pytest tests/api/`
- Format code: `black app/api/`
- Type checking: `mypy app/api/`
- Linting: `ruff check app/api/`
- API documentation: Visit `/docs` endpoint (Swagger UI) or `/redoc` (ReDoc)
