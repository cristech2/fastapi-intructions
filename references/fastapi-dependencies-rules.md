# FastAPI Dependency Rules

El sistema de **Inyección de Dependencias (`Depends`)** es fundamental para estructurar aplicaciones FastAPI, permitiendo la reutilización de lógica, la gestión de recursos y la separación de preocupaciones.

## ⚙️ Principios de `Depends`

* **Propósito**: Utilizar `Depends` para inyectar recursos como sesiones de base de datos, configuraciones, autenticación, y cualquier lógica reutilizable.
* **Asincronía**: Las dependencias para operaciones **I/O-bound** (como el acceso a la base de datos) deben ser **asíncronas** (`async def` o `AsyncGenerator`).
* **Testabilidad**: Las dependencias deben diseñarse para ser fácilmente reemplazables durante las pruebas.

-----

## 💾 Convención: Dependencia de Sesión de Base de Datos Asíncrona

La dependencia de la base de datos (`get_db_session`) es el ejemplo más importante de gestión de recursos a través de la inyección de dependencias. Debe residir en **`app/api/dependencies/db.py`**.

### 1\. Definición de la Dependencia (`get_db_session`)

Se debe usar un `AsyncGenerator` para garantizar que la sesión se abra (con `yield`) y se cierre correctamente, manejando la transacción (commit/rollback) dentro del ciclo de vida de la solicitud.

```python
# app/api/dependencies/db.py
from typing import AsyncGenerator, Annotated
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.db import AsyncSessionLocal # Fábrica de sesiones definida en core/db.py
from fastapi import Depends

async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    """Proporciona una sesión de base de datos asíncrona con manejo de transacción."""
    # Uso de la fábrica de sesiones asíncronas
    async with AsyncSessionLocal() as session:
        try:
            yield session # La sesión se inyecta aquí
            await session.commit() # Commit si el controlador termina sin excepciones
        except Exception:
            await session.rollback() # Rollback si ocurre una excepción
            raise
        finally:
            await session.close()

# Alias de dependencia para un uso limpio en routers y servicios
DBSession = Annotated[AsyncSession, Depends(get_db_session)]
```

### 2\. Uso de la Dependencia en Routers y Servicios

* **Aplicación**: La dependencia se inyecta en la firma de las funciones del *path operation* o en otras dependencias.
* **Notación Moderna**: Se recomienda usar la sintaxis `Annotated` (`DBSession` en el ejemplo anterior) para un código más limpio y tipado.

<!-- end list -->

```python
# app/api/v1/endpoints/users.py (Fragmento de uso)
from fastapi import APIRouter
from app.api.dependencies.db import DBSession # Importa el alias Annoted
from app.models.schemas.user import UserResponse
from app.services import user_service

router = APIRouter()

@router.get("/users/{user_id}", response_model=UserResponse)
async def read_user(
    user_id: uuid.UUID, 
    db: DBSession # Uso del alias Annoted/Depends
):
    # El router pasa la sesión al servicio de lógica de negocio
    return await user_service.get_user_by_id(db=db, user_id=user_id)
```

-----
