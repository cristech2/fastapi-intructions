---
description: 'FastAPI core configuration, security, exceptions, and middleware patterns'
applyTo: 'app/core/**/*.py'
---


# Estructura del Módulo Core

El módulo `app/core` debe contener la configuración global, gestión de base de datos, seguridad y manejadores de excepciones a nivel de aplicación.

## 1. Configuración Global (`app/core/config.py`)

Utiliza **`BaseSettings` de Pydantic** (v2+: `pydantic-settings`) para cargar variables de entorno y separar configuraciones por entorno (`dev`, `test`, `prod`).

**Normas:**
- Importa siempre con rutas absolutas.
- Usa tipado explícito y docstrings tipo Google.
- Los atributos de configuración deben estar en MAYÚSCULAS (PEP8).
- Implementa validación estricta de tipos y valores.
- Usa `model_config = ConfigDict(env_file=".env")` para Pydantic 2.x.

**Ejemplo correcto:**
```python
"""Configuración principal de la aplicación."""
import pydantic_settings
from pydantic import ConfigDict

class Settings(pydantic_settings.BaseSettings):
    """Ajustes cargados desde variables de entorno."""
    APP_NAME: str = "MyFastAPI"
    API_V1_STR: str = "/api/v1"
    DEBUG: bool = False
    DATABASE_URL: str
    SECRET_KEY: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    ALGORITHM: str = "HS256"

    model_config = ConfigDict(env_file=".env")

settings = Settings()
```

## 2. Configuración Asíncrona de Base de Datos (`app/core/db.py`)

Implementa conexión asíncrona con SQLAlchemy usando `create_async_engine` y `sessionmaker`.

**Normas:**
- Importa desde `sqlalchemy` y `sqlalchemy.ext.asyncio` con rutas absolutas.
- Configura el pool: `pool_size=5`, `max_overflow=10`, `pool_timeout=30`, `pool_pre_ping=True`.
- Usa `async_sessionmaker` y tipado explícito.
- No uses variables globales innecesarias.

**Ejemplo correcto:**
```python
"""Configuración de la base de datos asíncrona."""
import sqlalchemy
import sqlalchemy.ext.asyncio
import sqlalchemy.orm
from app.core.config import settings

engine = sqlalchemy.ext.asyncio.create_async_engine(
    settings.DATABASE_URL,
    echo=False,
    pool_size=5,
    max_overflow=10,
    pool_timeout=30,
    pool_recycle=1800,
    pool_pre_ping=True,
)

async_session_maker: sqlalchemy.orm.sessionmaker = sqlalchemy.orm.sessionmaker(
    bind=engine,
    class_=sqlalchemy.ext.asyncio.AsyncSession,
    expire_on_commit=False,
    autoflush=False,
)
```

### 2. Asynchronous Database Configuration (`app/core/db.py`)

Implement asynchronous SQLAlchemy connection with `create_async_engine` and `sessionmaker` for session factory.

**Requirements:**
- Import from `sqlalchemy` and `sqlalchemy.ext.asyncio` using absolute imports
- Configure connection pool: `pool_size=5`, `max_overflow=10`, `pool_timeout=30`
- Use `pool_pre_ping=True` to validate connections
- Use `async_sessionmaker` for creating `AsyncSessionLocal` factory
- Set `expire_on_commit=False` and `autoflush=False`

**Example structure:**
```python
import sqlalchemy
import sqlalchemy.ext.asyncio
import sqlalchemy.orm
from app.core import config

engine = sqlalchemy.ext.asyncio.create_async_engine(
    config.settings.database_url,
    echo=False,
    pool_size=5,
    max_overflow=10,
    pool_timeout=30,
    pool_recycle=1800,
    pool_pre_ping=True,
)

async_session_maker = sqlalchemy.orm.sessionmaker(
    engine,
    class_=sqlalchemy.ext.asyncio.AsyncSession,
    expire_on_commit=False,
    autoflush=False,
)
```


## 3. Excepciones Globales (`app/core/exceptions.py`)

Define clases de excepción personalizadas y modelos de respuesta de error consistentes.

**Normas:**
- Usa modelos Pydantic para respuestas de error.
- Incluye: `status_code`, `message`, `details`, `request_id`.
- Registra los handlers en `app/main.py`.
- Docstrings obligatorios y tipado explícito.

**Ejemplo correcto:**
```python
"""Modelos y excepciones globales."""
from pydantic import BaseModel
from typing import Any, Optional

class ErrorResponse(BaseModel):
    status_code: int
    message: str
    details: Optional[Any] = None
    request_id: Optional[str] = None

class NotFoundError(Exception):
    """Excepción para recursos no encontrados."""
    def __init__(self, message: str = "No encontrado"):
        super().__init__(message)
```


## 4. Seguridad y Hashing (`app/core/security.py`)

Gestiona operaciones criptográficas: hash de contraseñas (bcrypt) y creación/verificación de JWT.

**Normas:**
- Usa `bcrypt` para el hash de contraseñas.
- Las operaciones bcrypt son bloqueantes: usa `asyncio.to_thread()` en contextos async.
- Usa `python-jose` para JWT, incluyendo `exp` e `iat` en los claims.
- Usa `datetime.now(timezone.utc)` para marcas de tiempo.
- Maneja excepciones `JWTError` correctamente.
- Tipado explícito y docstrings tipo Google.

**Ejemplo correcto:**
```python
"""Funciones de seguridad y JWT."""
import bcrypt
import asyncio
from jose import jwt, JWTError
from datetime import datetime, timedelta, timezone
from app.core.config import settings

def hash_password(password: str) -> str:
    """Genera el hash de una contraseña usando bcrypt."""
    salt = bcrypt.gensalt()
    return bcrypt.hashpw(password.encode(), salt).decode()

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verifica una contraseña contra su hash."""
    return bcrypt.checkpw(plain_password.encode(), hashed_password.encode())

def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    """Crea un token JWT de acceso."""
    to_encode = data.copy()
    now = datetime.now(timezone.utc)
    expire = now + (expires_delta or timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES))
    to_encode.update({"exp": expire, "iat": now})
    return jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

def decode_access_token(token: str) -> dict:
    """Decodifica y valida un token JWT."""
    try:
        return jwt.decode(token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM])
    except JWTError as exc:
        raise ValueError("Token inválido") from exc
```


---

## Inicio de la Aplicación y Middleware (`app/main.py` y `app/middleware/`)

### Patrón Factory para la Aplicación

Utiliza el patrón factory para inicializar la app, incluir routers, middleware y handlers.

**Estructura recomendada en `app/main.py`:**
```python
"""Inicialización principal de la aplicación FastAPI."""
from fastapi import FastAPI
from app.core.config import settings
from app.middleware.request_id import RequestIDMiddleware
from fastapi.middleware.cors import CORSMiddleware

def create_application() -> FastAPI:
    """Crea y configura la instancia principal de FastAPI."""
    app = FastAPI(title=settings.APP_NAME, debug=settings.DEBUG)
    app.add_middleware(RequestIDMiddleware)
    app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])
    # Registrar handlers, routers, eventos, etc.
    return app

app = create_application()
```

### Middleware Personalizado

Implementa middleware para preocupaciones transversales.

**Normas:**
- Hereda de `starlette.middleware.base.BaseHTTPMiddleware`.
- Implementa `async def dispatch(self, request, call_next)`.
- Usa `request.state` para almacenar datos (ej. `request_id`).
- Docstrings obligatorios y tipado explícito.

**Ejemplo correcto (`app/middleware/request_id.py`):**
```python
"""Middleware para agregar un ID único a cada request."""
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response, JSONResponse
from starlette.types import ASGIApp

class RequestIDMiddleware(BaseHTTPMiddleware):
    """Agrega un ID único a cada request y lo incluye en la respuesta."""
    async def dispatch(self, request: Request, call_next: ASGIApp) -> Response:
        request_id = str(uuid.uuid4())
        request.state.request_id = request_id
        try:
            response = await call_next(request)
        except Exception:
            response = JSONResponse(
                status_code=500,
                content={"message": "Internal Server Error", "request_id": request_id},
            )
        response.headers["X-Request-ID"] = request_id
        return response
```

---


## Dependencias y Configuración

**Paquetes requeridos:**
- `fastapi`
- `sqlalchemy` (async)
- `pydantic-settings` (Pydantic v2)
- `bcrypt`
- `python-jose[cryptography]`

**Variables de entorno (.env):**
- `DATABASE_URL`: Cadena de conexión a la base de datos
- `SECRET_KEY`: Clave para firmar JWT
- `DEBUG`: Flag de modo debug
- `ACCESS_TOKEN_EXPIRE_MINUTES`: Minutos de expiración del token (por defecto: 30)
