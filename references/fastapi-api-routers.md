# FastAPI API Routers & Endpoints Instructions

Este documento establece las convenciones para organizar y definir las rutas de la API en FastAPI.

## 🛣️ Organización de Routers (`APIRouter`)

La organización del proyecto debe seguir un enfoque modular utilizando `APIRouter`.

### 1\. Estructura y Modularidad

* **Modularización**: Organiza las aplicaciones grandes en *routers* modulares.
* **Agrupación Lógica**: Utiliza prefijos (`prefix`) y etiquetas (`tags`) de *router* para una agrupación lógica clara.
* **Versión de API**: La versión principal (`/api/v1`) debe ser el prefijo de nivel superior, y dentro de ella, los *routers* específicos para recursos (e.g., `/users`, `/items`).

**Ejemplo de Organización de Routers:**

```python
# app/api/v1/api.py (Router principal de la versión)
# (Cambiado de __init__.py a api.py por claridad, aunque __init__ también es válido)
from fastapi import APIRouter
from app.api.v1.endpoints import users, items # Asumiendo endpoints/users.py

api_router = APIRouter()
# Cada recurso obtiene su propio prefijo y tags
api_router.include_router(users.router, prefix="/users", tags=["Users"])
api_router.include_router(items.router, prefix="/items", tags=["Items"])
```

### 2\. Definición del Endpoint (Controlador)

Cada archivo dentro de `app/api/v1/endpoints/` debe definir un *router* y sus *path operations*.

```python
# app/api/v1/endpoints/users.py
from typing import Annotated
from fastapi import APIRouter, Depends, status, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from app.api.dependencies.db import get_db_session
from app.models.schemas.user import UserCreate, UserResponse
from app.services import user_service # Lógica de negocio
from app.core.exceptions import NotFoundError # Excepción de negocio personalizada

router = APIRouter()

@router.post(
    "/",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED, # Código para el "happy path"
    summary="Create a new user",
    description="Register a new user in the system.",
)
async def create_new_user(
    user_in: UserCreate,
    # Usar Annotated para inyectar dependencias (moderno y más claro)
    db: Annotated[AsyncSession, Depends(get_db_session)] 
):
    """
    Crea un nuevo usuario.
    Lanza excepciones en caso de error.
    """
    # La lógica de negocio se delega al módulo de servicio
    try:
        # El servicio puede lanzar excepciones de negocio personalizadas
        # o podemos detectar el error aquí.
        existing_user = await user_service.get_user_by_email(db, email=user_in.email)
        if existing_user:
            # 1. Lanzar HTTPException: FastAPI la maneja.
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="A user with this email already exists.",
            )
        
        return await user_service.create_user(db=db, user_in=user_in)

    except NotFoundError as e:
        # 2. Lanzar Excepción Personalizada: El handler global la maneja.
        # (Ejemplo: si create_user dependiera de otro recurso que no se encontró)
        raise e 
    except Exception as e:
        # 3. Errores inesperados: El handler global de 500 los maneja.
        raise e
```

-----

## 💻 Path Operations (Reglas de RESTful Design)

Todas las operaciones de ruta deben seguir las siguientes convenciones:

| Regla                  | Descripción                                                                                                                                                                                                                   |
| :--------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Principios RESTful** | Usar nombres (sustantivos) para las URLs de recursos y aplicar métodos HTTP estándar (GET, POST, PUT, DELETE, PATCH).                                                                                                         |
| **`response_model`**   | **Siempre especificar** el `response_model` (Pydantic/SQLModel) para **validación** y, crucialmente, para **prevenir la fuga de datos** sensibles en las respuestas.                                                          |
| **Manejo de Errores**  | **Nunca retornar `JSONResponse` de error** desde un endpoint. **Siempre `raise`** `HTTPException` (para errores de API) o excepciones de negocio personalizadas (que serán capturadas por los *exception handlers* globales). |
| **Códigos de Estado**  | Usar códigos de estado HTTP apropiados (`200 OK`, `201 Created`, `404 Not Found`, etc.) en el decorador para el *happy path*.                                                                                                 |
| **Documentación**      | Incluir **`summary`**, **`description`** y **`tags`** en el decorador para una documentación OpenAPI clara.                                                                                                                   |
| **Asincronía**         | Usar **`async def`** para todas las operaciones de ruta que son *I/O-bound* (lectura de DB, llamadas externas), asegurando el uso de `await`.                                                                                 |
| **Paginación**         | **Siempre implementar paginación** para colecciones grandes utilizando parámetros de consulta (e.g., `skip` y `limit`).                                                                                                       |

**Ejemplo de Paginación y Filtrado Avanzado (Moderno):**

```python
# app/api/v1/endpoints/products.py
from typing import Annotated
from fastapi import Query, Depends, APIRouter
from sqlalchemy.ext.asyncio import AsyncSession
from app.api.dependencies.db import get_db_session
from app.models.schemas.product import ProductSchema
# Asumiendo que existe un servicio de paginación o lógica de repositorio
from app.services import product_service 

router = APIRouter()

@router.get(
    "/products/", 
    response_model=list[ProductSchema], # Usar list[T] en lugar de List[T]
    summary="List products",
    tags=["Products"]
)
async def list_products(
    # Usar Annotated para combinar validación (Query) y tipo
    db: Annotated[AsyncSession, Depends(get_db_session)],
    category: Annotated[str | None, Query(description="Filter by category")] = None,
    sort_by: Annotated[str, Query(description="Field to sort by")] = "id",
    
    # Usar Annotated para validación de parámetros de paginación
    skip: Annotated[int, Query(ge=0, description="Items to skip")] = 0,
    limit: Annotated[int, Query(ge=1, le=1000, description="Items per page")] = 100
):
    """
    Obtiene una lista paginada de productos, con filtros y ordenamiento.
    """
    # La lógica de filtrado y acceso a DB se delega al servicio
    products = await product_service.get_filtered_products(
        db, 
        category=category, 
        sort_by=sort_by, 
        skip=skip, 
        limit=limit
    )
    return products
```

-----
