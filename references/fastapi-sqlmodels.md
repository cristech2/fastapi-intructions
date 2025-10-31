# FastAPI Schemas & Models Instructions

Este documento define las convenciones para modelos FastAPI, integrando **SQLModel** con la sintaxis de *typing* moderna (Python 3.10+) y el uso de **`Annotated`** para la declaración de metadatos y validación.

## 🧱 Estructura y Convención de Modelos

Los modelos combinan definiciones Pydantic y ORM utilizando SQLModel, residiendo en **`app/models/`** con un archivo por entidad (`user.py`, `item.py`).

### 1\. Principios de Diseño de Clases (SQLModel)

|           Modelo / Esquema           |      Hereda de      |    Operación CRUD    |                                                                                 Contenido y Reglas Clave                                                                                 |
| :----------------------------------: | :-----------------: | :------------------: | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|         **`[Entidad]Base`**          |     `SQLModel`      |  **Base (Config)**   |    **Campos compartidos** (ej: `name`, `description`).<br>Define los atributos que son comunes tanto para la creación como para la respuesta. **No incluye** el `id` ni *timestamps*.    |
|           **`[Entidad]`**            |   `[Entidad]Base`   |    **BD (Tabla)**    | **Campos de la tabla/ORM**. <br>Añade `id` (PK, ej: `UUID`), *timestamps* y las **relaciones** (`relationship`). <br> **Regla:** Requiere `table=True` en `ConfigDict` o `class Config`. |
|        **`[Entidad]Create`**         |   `[Entidad]Base`   | **`CREATE` (POST)**  |                                **Input del cliente.** <br>Generalmente idéntico a `Base`. **No debe** permitir que el cliente envíe `id` o `created_at`.                                 |
|        **`[Entidad]Update`**         |     `SQLModel`      | **`UPDATE` (PATCH)** |      **Input del cliente (Opcional).** <br> **Regla:** Todos los campos deben ser opcionales (ej: `name: str \| None = None`). No hereda de `Base` para evitar valores por defecto.      |
|       **`[Entidad]Response`**        |   `[Entidad]Base`   |   **`READ` (GET)**   |                **Output simple (sin relaciones).** <br>Añade los campos generados por el servidor (ej: `id: UUID`, `created_at: datetime`). Se usa como `response_model`.                |
| **`[Entidad]ResponseConRelaciones`** | `[Entidad]Response` |   **`READ` (GET)**   |  **Output complejo (con relaciones).** <br>Añade los esquemas de respuesta de las relaciones (ej: `items: list[ItemResponse]`). <br> **Regla:** Requiere `Config.from_attributes=True`.  |

## 📐 Reglas de Pydantic, Typing y Annotations

### 1\. Uso de Annotations y Metadata

Utiliza **`Annotated`** para añadir metadatos de validación (Pydantic `Field`) y documentación (`Query`, `Path`) de forma explícita, mejorando el soporte de herramientas.

### 2\. Sintaxis Moderna de Typing

* Utiliza la sintaxis de **colección nativa** (PEP 585) en lugar de importar de `typing`.
  * **Ejemplo**: Usar `list[str]` en lugar de `List[str]`.

<!-- end list -->

```python
# app/models/user.py (Fragmento de Pydantic/SQLModel)
import uuid
from sqlmodel import SQLModel, Field, Relationship
from pydantic import EmailStr, Field as PydanticField
from typing import Annotated # Necesario para Annotated

# Definición explícita de campos Pydantic con metadatos
FullName = Annotated[str, PydanticField(min_length=1, max_length=100, example="Jane Doe")]
PasswordStr = Annotated[str, PydanticField(min_length=8, example="secure_password")]

class UserBase(SQLModel):
    email: EmailStr = Field(index=True, unique=True, example="user@example.com")
    full_name: FullName # Uso de Annotated para metadatos

# Esquema de Creación
class UserCreate(UserBase):
    password: PasswordStr 
```

## 💾 Reglas de ORM y SQLModel (UUID Predeterminado)

El modelo de tabla debe utilizar `UUID` para la clave primaria con generación, siempre que sea necesario, automática y nombres de campo explícitos.

```python
# app/models/user.py (Fragmento de ORM/SQLModel - Actualizado)
from datetime import datetime

# 4. Modelo de la Tabla (ORM)
class User(UserBase, table=True):
    # Clave primaria: UUID y nombre explícito
    user_id: uuid.UUID | None = Field(
        default_factory=uuid.uuid4, 
        primary_key=True,
        index=True
    )
    # Almacenar el hash
    hashed_password: str
    is_active: bool = Field(default=True)
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    
    # Relaciones: Usar list[...] y una clave explicativa (user_id)
    items: list["Item"] = Relationship(back_populates="owner")
    
# 5. Esquema de Respuesta (Output)
class UserResponse(UserBase):
    # Incluye el ID con el nombre específico
    user_id: uuid.UUID
    is_active: bool
    created_at: datetime

    class Config:
        from_attributes = True
```

-----
