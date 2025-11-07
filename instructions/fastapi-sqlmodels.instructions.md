---
description: 'SQLModel conventions and patterns for FastAPI schemas, models, and ORM structures with modern Python typing and Pydantic validation'
applyTo: '**/models/*.py'
---

# FastAPI SQLModel Development

Guidelines for creating effective SQLModel schemas and ORM models in FastAPI applications, combining Pydantic validation with SQLAlchemy ORM using modern Python typing (3.10+) and `Annotated` for metadata.

## General Instructions

- Always organize models in `app/models/` directory with one file per entity (e.g., `user.py`, `item.py`)
- Use SQLModel as the primary ORM framework combining Pydantic and SQLAlchemy
- Leverage `Annotated` from `typing` for explicit metadata and validation rules
- Use modern Python collection syntax (PEP 585) instead of `typing` module imports
- Implement clear separation between ORM, Pydantic, and response models
- Use UUID as default primary key type with automatic generation
- Never define `from_attributes = True` in Config when mapping ORM to response models. SQLModel does not require it.

## Model Architecture

### Class Hierarchy and Responsibilities

| Model Class | Inherits From | CRUD Operation | Key Characteristics |
|---|---|---|---|
| `[Entity]Base` | `SQLModel` | Base Config | Shared fields common to both creation and responses. Excludes `id` and timestamps. |
| `[Entity]` | `[Entity]Base` | Database (ORM) | ORM table model with `table=True`. Includes `id` (PK), timestamps, and relationships. Requires explicit table configuration. |
| `[Entity]Create` | `[Entity]Base` | CREATE (POST) | Client input schema. Generally identical to Base but validates incoming data. Never allows client-provided `id` or `created_at`. |
| `[Entity]Update` | `SQLModel` | UPDATE (PATCH) | Client input for updates. All fields must be optional with `\| None` syntax. Inherits from SQLModel directly to avoid default values. |
| `[Entity]Response` | `[Entity]Base` | READ (GET) | Server-generated fields output schema. Includes `id` and `created_at`. Used as `response_model` in endpoints. |
| `[Entity]ResponseWithRelations` | `[Entity]Response` | READ (GET) | Complex output with nested relationships. Includes related entity response schemas. Requires `from_attributes = True` configuration. |

## Code Standards

### Naming Conventions

- Model files: lowercase with underscores (`user.py`, `item.py`, `order.py`)
- Class names: PascalCase (`User`, `Item`, `Order`)
- Field names: snake_case (`full_name`, `created_at`, `is_active`)
- Foreign keys: explicit reference name (`user_id` not just `user`)
- Type aliases: PascalCase for custom types (`FullName`, `PasswordStr`)

### Type Annotations

- Use union syntax with `|` instead of `Union[Type1, Type2]`
- Use built-in generics: `list[str]` instead of `List[str]`
- Use `Annotated` for all fields requiring validation metadata
- Always specify examples in Field definitions for API documentation

### Field Organization

Within each model class, organize fields in this order:

1. Basic/scalar fields (strings, integers, booleans)
2. Complex fields (EmailStr, URLs, custom types)
3. Relationship fields (only in ORM and full response models)
4. Timestamps (created_at, updated_at, deleted_at)

## Best Practices

### Annotated and Validation

Use `Annotated` to combine field documentation, validation rules, and examples:

```python
from typing import Annotated
from pydantic import EmailStr, Field

FullName = Annotated[str, Field(min_length=1, max_length=100, example="Jane Doe")]
PasswordStr = Annotated[str, Field(min_length=8, example="secure_password")]
Email = Annotated[EmailStr, Field(example="user@example.com")]
```

Define type aliases at module level for reusability across multiple models.

### UUID Primary Keys

Always use UUID for primary keys with automatic generation:

```python
import uuid
from sqlmodel import Field

entity_id: uuid.UUID | None = Field(
    default_factory=uuid.uuid4,
    primary_key=True,
    index=True
)
```

Use `default_factory=uuid.uuid4` to generate UUIDs only when needed, not at class definition time.

### Relationships

Define relationships with explicit back_populates and use list syntax:

```python
from sqlmodel import Relationship

# In parent model
items: list["Item"] = Relationship(back_populates="owner")

# In child model
owner: "User" = Relationship(back_populates="items")
```

Always use forward references (quoted strings) for models not yet defined and specify `back_populates` for bidirectional relationships.

### Optional Fields in Updates

Update models require all fields to be optional to allow partial updates:

```python
class UserUpdate(SQLModel):
    email: str | None = None
    full_name: str | None = None
    is_active: bool | None = None
```

Do not inherit from Base to avoid inheriting non-optional fields with defaults.

### Configuration and from_attributes

Enable attribute mapping for ORM-to-Pydantic conversion. This is not needed for SQLModel. But if you need it for complex relationships on Pydantic models, use:

```python
class Config:
    from_attributes = True  # Allows .dict() to work with ORM instances
```

This is required for all response models that include ORM relationships or server-generated fields.

## Common Patterns

### Complete User Model Example

```python
import uuid
from datetime import datetime, timezone
from typing import Annotated
from sqlmodel import SQLModel, Field, Relationship
from pydantic import EmailStr

# Type aliases for reusabilidad
Email: EmailStr = Field(example="user@example.com")
FullName: str = Field(min_length=1, max_length=100, example="Jane Doe")
PasswordStr: str = Field(min_length=8, example="secure_password")


# Base model with shared fields
class UserBase(SQLModel):
    """
    Base model for users.

    Includes fields shared between creation and response schemas.
    """
    email: Email
    full_name: FullName
    is_active: bool = Field(default=True)

# ORM model for database
class User(UserBase, table=True):
    """
    ORM model for the users table in the database.

    Attributes:
        user_id (uuid.UUID | None): Unique identifier for the user.
        hashed_password (str): Hashed password for authentication.
        created_at (datetime): UTC creation timestamp.
        updated_at (datetime): UTC update timestamp.
        items (list["Item"]): Related items owned by the user.
    """
    __tablename__ = "users"
    
    user_id: uuid.UUID | None = Field(
        default_factory=uuid.uuid4,
        primary_key=True,
        index=True
    )
    hashed_password: str
    # Always use UTC for timestamps
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    # Bidirectional relationship with Item
    items: list["Item"] = Relationship(back_populates="owner")

# Client input for creation
class UserCreate(UserBase):
    """
    Input schema for user creation.

    Includes plain text password field.

    Attributes:
        password (PasswordStr): Plain text password for user creation.
    """
    password: PasswordStr

# Client input for updates
class UserUpdate(SQLModel):
    """
    Input schema for partial user updates.

    All fields are optional to allow partial updates.

    Attributes:
        email (Email | None): Optional email address.
        full_name (FullName | None): Optional full name.
        is_active (bool | None): Optional active status.
    """
    email: Email | None = None
    full_name: FullName | None = None
    is_active: bool | None = None

# Server response
class UserResponse(UserBase):
    """
    Response schema for users.

    Includes server-generated fields such as ID and timestamps.

    Attributes:
        user_id (uuid.UUID): Unique identifier for the user.
        created_at (datetime): UTC creation timestamp.
        updated_at (datetime): UTC update timestamp.
    """
    user_id: uuid.UUID
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True  # Required for ORM-to-Pydantic conversion with relationships

# Server response with nested relationships
class UserResponseWithItems(UserResponse):
    """
    Response schema for users including related items.

    Contains the list of items associated with the user.

    Attributes:
        items (list["ItemResponse"]): List of items associated with the user.
    """
    items: list["ItemResponse"] = []  # Nested relationship response
```

### Partial Update Pattern

For PATCH endpoints, use the update model without base inheritance:

```python
class ItemUpdate(SQLModel):
    title: str | None = None
    description: str | None = None
    price: float | None = None
    
    class Config:
        from_attributes = True
```

This allows clients to update only the fields they provide without affecting unchanged fields.

### Timestamps with Timezone

Always use UTC timezone for consistency:

```python
from datetime import datetime, timezone

created_at: datetime = Field(
    default_factory=lambda: datetime.now(timezone.utc)
)
```

Never use `datetime.now()` without timezone information.

## Security Considerations

- Never expose hashed passwords in response models
- Store sensitive data like password hashes in separate fields (e.g., `hashed_password`)
- Use `EmailStr` type for automatic email validation
- Implement field validators for complex validation logic using Pydantic `field_validator`
- Exclude sensitive fields from response models using `exclude` parameter
- Use Annotated constraints for field length limits and patterns

## Performance Guidelines

- Add database indexes to frequently queried fields: `Field(index=True)`
- Use `unique=True` for fields that should be unique in the database
- Define relationships with lazy loading considerations
- Use `response_model_exclude_unset=True` to omit unset optional fields
- Consider pagination for large relationship lists

## Validation

- Build command: Check models can be imported without errors
- Linting: Use `pylint` or `ruff` to verify type hints
- Type checking: Use `mypy` or Pylance with `python.analysis.typeCheckingMode` set to strict
- Testing: Verify model instantiation, serialization, and ORM operations

### Verification Steps

```bash
# Type checking
mypy app/models/

# Linting
ruff check app/models/

# Import verification
python -c "from app.models import *; print('All models imported successfully')"
```

