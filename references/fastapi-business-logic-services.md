# FastAPI Business Logic & Services Instructions

Este documento define las convenciones para implementar la **lógica de negocio** (módulos `Services`) y las **operaciones de acceso a datos** (CRUD), aplicando el principio de separación de preocupaciones (SoC).

-----

## 📂 Organización y Principios de Servicio

* **Ubicación**: Toda la lógica de negocio y acceso a datos reside en **`app/services/`**.
* **Separación de Preocupaciones**: Los servicios contienen la **lógica de negocio pura** y las operaciones de base de datos (CRUD).
* **Agnóstico a la API**: Los módulos de servicio son "agnósticos al framework". **No deben importar** nada de `fastapi` (como `HTTPException`, `Request`, `status`). Solo deben manejar tipos de Python, modelos (Pydantic/SQLModel) y excepciones personalizadas.
* **Manejo de Transacciones**: Los servicios **no gestionan la transacción** (no llaman a `db.commit()` o `db.rollback()`). Simplemente preparan los cambios en la sesión (`db.add()`, `db.delete()`). La dependencia `get_db_session` es la única responsable de confirmar (commit) o revertir (rollback) la transacción.

### Ejemplo de Estructura de Servicio (Correcto)

```python
# app/services/user_service.py
import asyncio
from sqlalchemy.ext.asyncio import AsyncSession
from sqlmodel import select
from app.models.user import User, UserCreate
from app.core.security import hash_password # Importa la función de hashing
from app.core.exceptions import NotFoundError, EmailAlreadyExistsError

# --- Operación de Escritura (Write) ---
async def create_user(*, db: AsyncSession, user_in: UserCreate) -> User:
    
    # 0. Validar lógica de negocio (ej. duplicados)
    existing_user = await get_user_by_email(db, email=user_in.email)
    if existing_user:
        # Lanza una excepción personalizada pura
        raise EmailAlreadyExistsError(email=user_in.email)

    # 1. Aplicar lógica de negocio (Hashing de password)
    # ¡CRÍTICO! Usar to_thread para no bloquear el event loop
    hashed_password = await asyncio.to_thread(
        hash_password, user_in.password
    )
    
    # 2. Crear modelo ORM
    db_user = User(
        **user_in.model_dump(exclude={"password"}), 
        hashed_password=hashed_password
    )
    
    # 3. Preparar operación de DB (SIN commit)
    db.add(db_user)
    
    # 4. Refrescar la instancia (opcional, si se necesita el ID generado)
    await db.flush() # Envía la orden a la BD, pero no cierra la transacción
    await db.refresh(db_user)
    
    return db_user

# --- Operación de Lectura (Read) ---
async def get_user_by_id(*, db: AsyncSession, user_id: int) -> User:
    
    # Usar .get() para PK es lo más eficiente
    user = await db.get(User, user_id)
    
    if not user:
        # Lanza una excepción de Python que NO es HTTP
        raise NotFoundError(item_type="User", item_id=str(user_id))
        
    return user

async def get_user_by_email(db: AsyncSession, email: str) -> User | None:
    statement = select(User).where(User.email == email)
    result = await db.execute(statement)
    return result.scalars().first()
```

-----

## 🛠️ Reglas de Acceso a Datos y Transacciones

Todas las interacciones con la base de datos deben ser **asíncronas** y gestionadas a través de sesiones inyectadas.

### 1\. Manejo Transaccional (Dependencia `get_db`)

* **Dependencia Asíncrona**: La dependencia (`app/api/dependencies/db.py`) debe usar `AsyncGenerator` para inyectar la `AsyncSession`.
* **Manejo Transaccional (ACID)**: Esta dependencia **DEBE** ser la única responsable de la transacción. Debe implementar el patrón `try...yield...finally`:
  * `try`: Inicia la sesión.
  * `yield session`: Entrega la sesión al *path operation*.
  * `except`: Llama a `await db.rollback()` si ocurre CUALQUIER excepción (ya sea una `HTTPException` o una `NotFoundError` personalizada).
  * `else`: Llama a `await db.commit()` si no hubo excepciones.
  * `finally`: Llama a `await db.close()`.

### 2\. Eficiencia y Seguridad de Consultas

* **Async ORM**: Utilizar siempre `await db.execute(query)` o `await db.get()`.
* **Prevención N+1**: Evitar el problema N+1 usando **Eager Loading** (`selectinload()`) en consultas que involucren relaciones que se sabe que se van a usar.
* **Paginación Obligatoria**: **Siempre implementar paginación** (`.offset(skip).limit(limit)`) en consultas que devuelven colecciones.

-----

## 🚨 Manejo de Errores: Servicio (Lanza) vs. Endpoint (Captura)

Esta es la regla más importante de la separación de capas.

### 1\. El Servicio (Capa de Negocio)

El servicio **solo lanza excepciones de Python puras** (definidas en `app/core/exceptions.py`). *Nunca* lanza `HTTPException`.

```python
# app/services/user_service.py (Reiteración del ejemplo)

async def get_user_by_id(*, db: AsyncSession, user_id: int) -> User:
    user = await db.get(User, user_id)
    if not user:
        # CORRECTO: Lanza excepción de negocio.
        raise NotFoundError(item_type="User", item_id=str(user_id))
    return user
```

### 2\. El Endpoint (Capa de API)

El endpoint es responsable de *traducir* esos errores en respuestas HTTP. Tenemos dos patrones:

#### Patrón 1 (Recomendado): Confiar en el Manejador Global

Para excepciones comunes que siempre devuelven el mismo error HTTP (como `NotFoundError` -\> 404), el endpoint **no necesita código `try...except`**.

El manejador de excepciones global (registrado en `main.py`) interceptará la `NotFoundError` y la convertirá automáticamente en la respuesta JSON 404 formateada.

```python
# app/api/v1/endpoints/users.py (Fragmento)
from typing import Annotated
from fastapi import APIRouter, Depends
from app.core.exceptions import NotFoundError # (Aunque no la capturemos, es bueno saberla)

router = APIRouter()

@router.get("/{user_id}", response_model=UserResponse)
async def read_user(
    user_id: int, 
    db: Annotated[AsyncSession, Depends(get_db_session)]
):
    """
    Obtiene un usuario por ID.
    No necesita try/except para NotFoundError.
    El manejador global se encarga si el servicio lanza la excepción.
    """
    # Si user_service.get_user_by_id lanza NotFoundError,
    # el manejador global lo captura.
    user = await user_service.get_user_by_id(db=db, user_id=user_id)
    return user
```

#### Patrón 2 (Específico): Capturar y Lanzar `HTTPException`

Se usa cuando el endpoint necesita tomar una **decisión local** o cuando una excepción de negocio específica debe ser mapeada a un error HTTP *diferente* (ej. `EmailAlreadyExistsError` -\> 409 Conflict).

```python
# app/api/v1/endpoints/users.py (Fragmento)
from fastapi import status, HTTPException
from app.core.exceptions import EmailAlreadyExistsError

@router.post("/", status_code=status.HTTP_201_CREATED, response_model=UserResponse)
async def create_new_user(
    user_in: UserCreate,
    db: Annotated[AsyncSession, Depends(get_db_session)]
):
    try:
        user = await user_service.create_user(db=db, user_in=user_in)
        return user
    except EmailAlreadyExistsError as e:
        # CORRECTO: Captura la excepción de negocio específica...
        # ...y la traduce a una excepción HTTP en la capa de API.
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail=f"User with email {e.email} already exists."
        )
    # Nota: No necesitamos capturar NotFoundError, el global lo haría.
```

-----
