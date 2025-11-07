# FastAPI Development Guidelines

Complete guide for developing FastAPI applications following industry best practices, clean code principles, and proven architectural patterns.

## Coding Foundations

- Focus on writing **clean and maintainable FastAPI code**
- Apply **Python best practices** and use **type hints** everywhere to improve IDE support and validation
- Leverage Python 3.12+ features for modern, efficient code
- Prioritize readability and maintainability over clever solutions

## Project Structure & Architecture

### Recommended Folder Structure

```bash
app/
├── api/
│   ├── v1/
│   │   ├── endpoints/
│   │   │   ├── users.py
│   │   │   └── items.py
│   │   └── dependencies.py
│   └── v2/
├── core/
│   ├── config.py
│   ├── security.py
│   └── database.py
├── models/
│   ├── user.py
│   └── item.py
├── services/
│   ├── user_service.py
│   └── item_service.py
└── main.py
```

### Architecture Principles

- **Separation of Concerns**: Clear boundaries between layers (API, business logic, data access)
- **Maintainability**: Organize code into logical modules that are easy to understand and modify
- **Scalability**: Design structure to support application growth without major refactoring
- **Reusability**: Create shared components (dependencies, utilities, services) to avoid duplication

### Directory Purposes

- **`app/api/`**: API endpoints and versioning (`v1/`, `v2/`)
- **`app/api/dependencies/`**: Reusable dependencies (authentication, database sessions)
- **`app/core/`**: Core modules (configuration, security, database sessions)
- **`app/models/`**: SQLModel definitions for ORM and Pydantic validation
- **`app/services/`**: Business logic layer
- **`main.py`**: Application entry point using the application factory pattern (`create_app()`)

## FastAPI Framework Rules

### Models and Schemas

#### SQLModel Usage

- Use SQLModel as the **single source of truth** for: database models, data validation, serialization, and API documentation
- Unify the domain model (database) and API schema in a single SQLModel class to avoid duplication (DRY principle)
- Create separate **DTO schemas** for input (Create/Update) and output (Read)
  - These can inherit from the base model or be Pydantic `BaseModel`
  - Prevents data leaks (e.g., hiding `password_hash`)
  - Use in `response_model` for proper serialization

#### Model Configuration

```python
import sqlmodel
import typing
import uuid

class UserBase(sqlmodel.SQLModel):
    """
    Campos base para usuario, compartidos entre modelo de tabla y DTOs.
    No usar Annotated aquí para evitar conflictos en migraciones.
    """
    email: str
    name: str
    is_active: bool = True

class User(UserBase, table=True):
    """
    User database model.
    Almacena datos persistentes del usuario. El campo password_hash nunca debe exponerse en respuestas API.
    """
    id: uuid.UUID | None = sqlmodel.Field(default_factory=uuid.uuid4, primary_key=True, description="Identificador único de usuario (UUID, PK lógica)")
    password_hash: str  # Hash de la contraseña, nunca exponer en API

class UserRead(UserBase):
    """
    User response schema - excludes sensitive data.
    Utilizado como response_model para evitar fugas de datos sensibles.
    """
    id: typing.Annotated[
        uuid.UUID,
        sqlmodel.Field(description="Identificador único de usuario (UUID)")
    ]

class UserCreate(UserBase):
    """
    User creation schema.
    Utilizado para validar datos de entrada al crear usuarios. Usa Annotated para validación extra.
    """
    password: typing.Annotated[
        str,
        sqlmodel.Field(min_length=8, description="Contraseña en texto plano, mínimo 8 caracteres")
    ]
```

### Path Operations & Routers

#### RESTful Principles

- Follow **RESTful design patterns** for all endpoints
- Always specify `response_model` for type validation and documentation
- Set appropriate `status_code` (200, 201, 204, etc.)
- Include `tags`, `summary`, and `description` for OpenAPI documentation

#### Good Example - Proper Endpoint Definition

```python
import fastapi
import sqlmodel
import typing
import uuid

router = fastapi.APIRouter(
    prefix="/users",
    tags=["users"],
    summary="Operaciones de usuario",
    description="Endpoints para gestión de usuarios. Incluye creación y listado paginado."
)

@router.get(
    "/",
    response_model=list[UserRead],
    status_code=fastapi.status.HTTP_200_OK,
    summary="Listar usuarios",
    description="Recupera todos los usuarios con paginación."
)
async def list_users(
    session: sqlmodel.ext.asyncio.AsyncSession = fastapi.Depends(get_session),
    skip: typing.Annotated[int, fastapi.Query(0, ge=0, description="Número de registros a omitir para paginación")],
    limit: typing.Annotated[int, fastapi.Query(10, ge=1, le=100, description="Cantidad máxima de registros a devolver")]
) -> list[UserRead]:
    """
    Recupera todos los usuarios con paginación.
    La lógica de acceso a datos se delega al servicio UserService.
    Args:
        session: Sesión de base de datos asíncrona.
        skip: Número de registros a omitir.
        limit: Número máximo de registros a devolver.
    Returns:
        Lista de usuarios.
    """
    users = await UserService.get_users(session, skip, limit)
    return users

@router.post(
    "/",
    response_model=UserRead,
    status_code=fastapi.status.HTTP_201_CREATED,
    summary="Crear usuario",
    description="Crea un nuevo usuario en la base de datos."
)
async def create_user(
    user_data: UserCreate,
    session: sqlmodel.ext.asyncio.AsyncSession = fastapi.Depends(get_session)
) -> UserRead:
    """
    Crea un nuevo usuario.
    La validación y persistencia se delega al servicio UserService.
    Args:
        user_data: Datos de entrada para el usuario.
        session: Sesión de base de datos asíncrona.
    Returns:
        Usuario creado.
    """
    user = await UserService.create_user(session, user_data)
    return user
```

#### Bad Example - Avoid These Patterns

```python
# ❌ Missing response_model - loses type validation
@router.get("/users")
async def get_users():
    return users

# ❌ Missing status_code
@router.post("/users", response_model=UserRead)
async def create_user(data: UserCreate):
    pass

# ❌ No documentation
@router.get("/users/{user_id}")
async def get(u: int):
    pass
```

#### Router Organization

- Organize large applications into **modular routers**
- Use `prefix` for URL grouping (e.g., `/api/v1/users`)
- Use `tags` for API documentation organization
- Include `include_in_schema` to hide internal endpoints if needed

```python
router = APIRouter(
    prefix="/items",
    tags=["items"],
    responses={404: {"description": "Not found"}}
)
```

### Dependencies and Asynchrony

#### Dependency Injection

- Use `Depends()` for database sessions, configuration, authentication, and reusable logic
- Make dependencies **async** for I/O-bound operations
- Design dependencies to be **testable** and easily replaceable during testing

#### Good Example - Dependency Pattern

```python
import fastapi
import sqlmodel

async def get_session() -> sqlmodel.ext.asyncio.AsyncSession:
    """
    Provee una sesión de base de datos asíncrona para los handlers de rutas.
    Esta función es fácilmente reemplazable en tests.
    Returns:
        Sesión asíncrona de base de datos.
    """
    async with AsyncSessionLocal() as session:
        yield session

async def get_current_user(
    token: str = fastapi.Depends(oauth2_scheme),
    session: sqlmodel.ext.asyncio.AsyncSession = fastapi.Depends(get_session)
) -> User:
    """
    Autentica y retorna el usuario actual según el token.
    Args:
        token: Token JWT extraído del header Authorization.
        session: Sesión de base de datos asíncrona.
    Returns:
        Instancia de usuario autenticado.
    Raises:
        fastapi.HTTPException: Si el token es inválido o el usuario no existe.
    """
    user = await verify_token(token, session)
    if not user:
        raise fastapi.HTTPException(status_code=401, detail="Invalid token")
    return user

@router.get("/profile", response_model=UserRead, summary="Perfil de usuario", description="Devuelve el perfil del usuario autenticado.")
async def get_profile(current_user: User = fastapi.Depends(get_current_user)) -> UserRead:
    """
    Devuelve el perfil del usuario autenticado.
    Args:
        current_user: Usuario autenticado inyectado por la dependencia.
    Returns:
        Datos del usuario para respuesta API.
    """
    return current_user
```

#### Asynchronous Functions

- Use `async def` for route handlers and I/O-bound dependencies
- Ensure proper use of `await` for async operations
- Avoid blocking the event loop with CPU-intensive tasks
- Offload heavy computations to background tasks or worker processes

#### Bad Example - Blocking Operations

```python
# ❌ Blocks the event loop
@router.post("/process")
async def process_data(data: DataModel):
    for i in range(1000000):  # CPU-intensive
        result = expensive_calculation(i)
    return result

# ✅ Better: Offload to background task
from fastapi import BackgroundTasks

@router.post("/process")
async def process_data(data: DataModel, background_tasks: BackgroundTasks):
    background_tasks.add_task(expensive_operation, data)
    return {"status": "Processing"}
```

### Lifecycle and Concurrency

#### Lifecycle Events

- Use `startup` and `shutdown` events for **resource initialization and cleanup**
- Examples: database connections, background task workers, external service connections

```python
import fastapi
import typing

@app.on_event("startup")
async def startup() -> None:
    """
    Inicializa recursos globales al iniciar la aplicación.
    Ejemplo: pool de base de datos y conexión a Redis.
    """
    # Inicialización de pool de base de datos
    app.db_pool = create_connection_pool()
    # Inicialización de Redis asíncrono
    app.redis = await aioredis.create_redis_pool()

@app.on_event("shutdown")
async def shutdown() -> None:
    """
    Libera y limpia recursos globales al apagar la aplicación.
    """
    await app.db_pool.close()
    app.redis.close()
```

#### Background Tasks

- Use `BackgroundTasks` for **non-blocking operations** after sending response
- Examples: email sending, data processing, logging
- Avoid long-running tasks; use job queues (Celery, RQ) for complex workflows

```python
import fastapi

@router.post(
    "/notify",
    summary="Enviar notificación",
    description="Envía una notificación por email en segundo plano."
)
async def send_notification(
    user_id: typing.Annotated[
        uuid.UUID,
        fastapi.Path(description="Identificador único del usuario destinatario (UUID)")
    ],
    background_tasks: fastapi.BackgroundTasks
) -> dict[str, str]:
    """
    Envía una notificación por email en segundo plano usando BackgroundTasks.
    Args:
        user_id: Identificador del usuario destinatario.
        background_tasks: Instancia de BackgroundTasks para ejecutar tareas asíncronas.
    Returns:
        Estado de la cola de notificación.
    """
    background_tasks.add_task(send_email, user_id)
    return {"status": "Notification queued"}
```

## Common Patterns

### Service Layer Pattern

Business logic in FastAPI can be implemented using either dedicated service classes or
standalone async functions grouped by module. Both approaches are valid and should be
chosen based on the complexity and needs of your project.

- **Standalone Functions:**
    Recommended for simple CRUD operations and when no internal state or grouping is
    required. Functions are grouped by domain entity in their respective modules
    (e.g., `ingredient_service.py`, `recipe_service.py`).

- **Service Classes:**
    Useful when you need to maintain internal state, share resources, or group related
    operations. Classes allow for inheritance, composition, and advanced design patterns.

### Example: Standalone Function

```python
# app/services/ingredient_service.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlmodel import select
import app.models.ingredient

async def get_ingredient_by_id(db: AsyncSession, ingredient_id: int) -> app.models.ingredient.Ingredient:
    result = await db.get(app.models.ingredient.Ingredient, ingredient_id)
    return result
```

### Example: Service Class

```python
# app/services/ingredient_service.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlmodel import select
import app.models.ingredient

class IngredientService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_ingredient_by_id(self, ingredient_id: int) -> app.models.ingredient.Ingredient:
        result = await self.db.get(app.models.ingredient.Ingredient, ingredient_id)
        return result
```

**Guidelines:**

- Use standalone functions for simple, stateless operations.

- Use service classes for complex logic, stateful operations, or when grouping related
    methods is beneficial.

- Always keep business logic separated from API and framework concerns.

### Error Handling

Implement consistent error handling with appropriate HTTP status codes:

```python
import fastapi
import typing
import uuid

@router.get(
    "/users/{user_id}",
    response_model=UserRead,
    summary="Obtener usuario por ID",
    description="Devuelve los datos de un usuario por su identificador único (UUID)."
)
async def get_user(
    user_id: typing.Annotated[
        uuid.UUID,
        fastapi.Path(description="Identificador único del usuario (UUID)")
    ],
    session: sqlmodel.ext.asyncio.AsyncSession = fastapi.Depends(get_session)
) -> UserRead:
    """
    Devuelve los datos de un usuario por su identificador único.
    Args:
        user_id: UUID del usuario a buscar.
        session: Sesión de base de datos asíncrona.
    Returns:
        Datos del usuario si existe.
    Raises:
        fastapi.HTTPException: Si el usuario no existe.
    """
    user = await UserService.get_by_id(session, user_id)
    if not user:
        raise fastapi.HTTPException(
            status_code=fastapi.status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    return user
```

## Validation and Verification

- Run tests: `pytest`
- Type checking: `mypy app/`
- Formatting, import sorting, and linting: `ruff check app/`
