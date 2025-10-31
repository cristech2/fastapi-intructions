# FastAPI Core, Config & Middleware Instructions

## ⚙️ Configuración (app/core)

El módulo `app/core` debe contener las configuraciones globales, la gestión de la base de datos y los manejadores de excepciones a nivel de aplicación y la seguridad.

### 1\. Configuración Global (`app/core/config.py`)

Utiliza **Pydantic's `BaseSettings`** para cargar variables de entorno y separar las configuraciones por entorno (dev, test, prod).

```python
# app/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Core
    APP_NAME: str = "MyFastAPI"
    API_V1_STR: str = "/api/v1"
    DEBUG: bool = False
    
    # Database (Se recomienda usar variables de entorno)
    DATABASE_URL: str = "postgresql+asyncpg://user:password@localhost/dbname"
    
    # Security (Mantener sensibles en variables de entorno)
    SECRET_KEY: str 
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    ALGORITHM: str = "HS256" # Algoritmo para JWT
    
    class Config:
        # Pydantic 1.x: env_file = ".env"
        # Pydantic 2.x: model_config = ConfigDict(env_file=".env")
        pass

settings = Settings()
```

### 2\. Configuración de Base de Datos Asíncrona (`app/core/db.py`)

Implementa la conexión asíncrona de SQLAlchemy utilizando `create_async_engine` y `sessionmaker` para la fábrica de sesiones, configurando el **pool de conexiones**.

```python
# app/core/db.py
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import QueuePool
from .config import settings

# Crear engine con configuración del pool
engine = create_async_engine(
    settings.DATABASE_URL,
    echo=False,  # True para logging SQL en desarrollo
    pool_size=5,
    max_overflow=10,
    pool_timeout=30,
    pool_recycle=1800,
    pool_pre_ping=True,
    poolclass=QueuePool
)

# Fábrica de sesiones (Session Local)
# La dependencia get_db() debe usar esta fábrica y manejar commit/rollback en app/api/dependencies/db.py
AsyncSessionLocal = sessionmaker(
    engine, 
    class_=AsyncSession, 
    expire_on_commit=False,
    autoflush=False
)
```

### 3\. Excepciones Globales (`app/core/exceptions.py`)

Define clases de excepción personalizadas y manejadores para garantizar **respuestas de error consistentes**.

```python
# app/core/exceptions.py
from fastapi import status
from pydantic import BaseModel
from typing import Dict, Any, Optional

# Modelo de respuesta de error consistente
class ErrorResponse(BaseModel):
    status_code: int
    message: str
    details: Dict[str, Any] | None = None
    request_id: Optional[str] = None
    
# Clase de excepción personalizada (ejemplo)
class NotFoundError(Exception):
    def __init__(self, item_type: str, item_id: str):
        self.item_type = item_type
        self.item_id = item_id

# Este manejador debe registrarse en main.py
async def not_found_exception_handler(request, exc: NotFoundError):
    from fastapi.responses import JSONResponse
    return JSONResponse(
        status_code=status.HTTP_404_NOT_FOUND,
        content=ErrorResponse(
            status_code=status.HTTP_404_NOT_FOUND,
            message=f"{exc.item_type} with id {exc.item_id} not found",
            request_id=getattr(request.state, "request_id", None)
        ).dict(),
    )
```

### 4\. Seguridad y Hashing (`app/core/security.py`)

Este módulo maneja las operaciones criptográficas, como el hashing de contraseñas (usando `bcrypt`) y la creación/verificación de tokens JWT.

*(Requiere: `pip install bcrypt python-jose[cryptography]`)*

```python
# app/core/security.py
import bcrypt
from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError
from fastapi import HTTPException, status
from .config import settings

# --- Hashing de Contraseñas (Bcrypt) ---

def hash_password(password: str) -> str:
    """
    Genera un hash de la contraseña usando bcrypt.
    Nota: Bcrypt es una operación síncrona y bloqueante (CPU-bound).
    En un endpoint async, debe ser envuelta con:
    await asyncio.to_thread(hash_password, password)
    """
    password_bytes = password.encode('utf-8')
    salt = bcrypt.gensalt()
    hashed_password = bcrypt.hashpw(password_bytes, salt)
    return hashed_password.decode('utf-8')

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """
    Verifica una contraseña plana contra un hash almacenado.
    Nota: También es bloqueante y debe usarse con asyncio.to_thread.
    """
    plain_password_bytes = plain_password.encode('utf-8')
    hashed_password_bytes = hashed_password.encode('utf-8')
    
    return bcrypt.checkpw(plain_password_bytes, hashed_password_bytes)

# --- JSON Web Tokens (JWT) ---

def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    """
    Crea un token de acceso JWT.
    """
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        # Usa el tiempo de expiración de la configuración
        expire = datetime.now(timezone.utc) + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
        )
        
    to_encode.update({"exp": expire, "iat": datetime.now(timezone.utc)})
    
    encoded_jwt = jwt.encode(
        to_encode, 
        settings.SECRET_KEY, 
        algorithm=settings.ALGORITHM
    )
    return encoded_jwt

def decode_access_token(token: str) -> dict:
    """
    Decodifica el token. Maneja la expiración y errores de firma.
    Usado comúnmente en dependencias de autenticación.
    """
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(
            token, 
            settings.SECRET_KEY, 
            algorithms=[settings.ALGORITHM]
        )
        # Aquí se podrían validar claims específicos (ej. 'sub')
        return payload
    except JWTError:
        raise credentials_exception
    except Exception:
        raise credentials_exception

```

-----

## 🛡️ Middleware & Arranque (app/main.py)

El archivo `app/main.py` debe usar el patrón de **fábrica de aplicaciones** para incluir *routers*, **middleware** y manejadores de excepciones.

### 1\. Arranque y Organización de Routers

Utiliza el patrón de fábrica para inicializar la aplicación e incluir los *routers* de versión.

```python
# app/main.py
from fastapi import FastAPI
from app.api.v1 import api_router
from app.core.config import settings
from app.core.exceptions import NotFoundError, not_found_exception_handler
# Importar Middleware si está definido en un archivo separado
from app.middleware.request_id import RequestIDMiddleware 
from fastapi.middleware.cors import CORSMiddleware 

def create_application() -> FastAPI:
    application = FastAPI(
        title=settings.APP_NAME,
        openapi_url=f"{settings.API_V1_STR}/openapi.json",
        debug=settings.DEBUG,
    )
    
    # 1. Registrar Middleware (El orden es crucial)
    application.add_middleware(RequestIDMiddleware)  # Primero para trazar todo
    application.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    
    # 2. Registrar Manejadores de Excepciones
    application.add_exception_handler(NotFoundError, not_found_exception_handler)
    # Registrar catch-all handler para errores inesperados
    # application.add_exception_handler(Exception, global_exception_handler) 
    
    # 3. Incluir Routers
    application.include_router(
        api_router, 
        prefix=settings.API_V1_STR
    )
    
    # 4. Registrar Eventos de Ciclo de Vida (startup/shutdown)
    # @application.on_event("startup")
    # async def startup_event():
    #     await initialize_resources() 
        
    return application

app = create_application()
```

### 2\. Implementación de Middleware

Implementa middleware para **preocupaciones transversales** (*cross-cutting concerns*) como logging, CORS o seguimiento de Request ID.

```python
# app/middleware/request_id.py (Ejemplo de Middleware custom)
import uuid
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = str(uuid.uuid4())
        # Almacenar en el estado de la solicitud para logging y excepciones
        request.state.request_id = request_id
        
        try:
            response = await call_next(request)
        except Exception as exc:
            # Asegurar que el request_id esté en la respuesta de error 500
            response = JSONResponse(
                status_code=500,
                content={"message": "Internal Server Error", "request_id": request_id}
            )
        
        # Añadir al encabezado de la respuesta
        response.headers["X-Request-ID"] = request_id
        return response
```

-----
