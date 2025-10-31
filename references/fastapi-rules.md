# FastAPI General & Core Rules

## 🧠 Coding Foundations

* Enfocarse en escribir **código FastAPI limpio y mantenible**.
* Aplicar **mejores prácticas de Python** y utilizar **type hints** (*sugerencias de tipo*) en todas partes para mejorar el soporte del IDE y la validación.

---

## 🏗️ Project Structure & Architecture

* **Principios Fundamentales**: Garantizar una clara **separación de preocupaciones** y una estructura de carpetas **mantenible**.
* **Estructura Recomendada**:
  * **`app/api/`**: Contiene los *endpoints* de la API y las versiones (`v1/`, `v2/`).
  * **`app/api/dependencies/`**: Dependencias reutilizables (e.g., autenticación, sesiones de DB).
  * **`app/core/`**: Módulos centrales (configuración, seguridad, sesiones de DB).
  * **`app/models/`**: Definiciones de modelos SQLMModel lo que permite modelos ORM y contratos I/O Pydantic.
  * **`app/services/`**: **Lógica de negocio**.
  * **`main.py`**: Punto de entrada de la aplicación, utilizando un patrón de **fábrica de aplicaciones** (`create_application()`).

---

## 🧰 Framework-Specific Rules (Reglas Centrales de FastAPI)

### Modelos y Esquemas

* **SQLModel Models**:
  * Usar como la **fuente única de verdad** para: **modelos de BD** (mapeo ORM), **validación** de datos, **serialización** y **documentación** de API.
  * **Unificar** el modelo de dominio (BD) y el esquema de API en una clase base (`SQLModel`) para **evitar duplicación** (principio DRY).
  * Crear **esquemas DTO (Data Transfer Objects)** separados para **entrada (Create/Update)** y **salida (Read)**. Estos pueden heredar del modelo base (o ser `BaseModel` de Pydantic) para **prevenir fugas de datos** (e.g., ocultar `password_hash`) y usar en `response_model`.
  * Usar `ConfigDict` y `Field` para configurar el modelo y sus campos (e.g., `table=True` en la clase base de la tabla, `default=None`, `foreign_key`, `index=True`).

### Path Operations & Routers

* **Operaciones de Ruta**:
  * Seguir **principios RESTful**.
  * Especificar siempre `response_model`.
  * Establecer `status_code` apropiado.
  * Incluir `tags`, `summary` y `description` para la **documentación**.
* **Routers (`APIRouter`)**:
  * Organizar aplicaciones grandes en **routers modulares**.
  * Usar prefijos (`prefix`) y etiquetas (`tags`) de router para agrupación lógica.

### Dependencias y Asincronía

* **Inyección de Dependencias (`Depends`)**:
  * Usar para sesiones de base de datos, configuración, autenticación y lógica reutilizable.
  * Hacer que las dependencias sean **`async`** para operaciones I/O-bound.
  * Diseñar para **testabilidad** con posibles reemplazos durante las pruebas.
* **Funciones Asíncronas**:
  * Usar `async def` para operaciones de ruta y dependencias I/O-bound.
  * Asegurar el uso adecuado de `await`.
  * **Evitar bloquear el *event loop*** con tareas intensivas de CPU.

### Ciclo de Vida y Concurrencia

* **Eventos del Ciclo de Vida**:
  * Usar eventos `startup`/`shutdown` para **inicialización y limpieza de recursos** (e.g., conexiones a DB, tareas en segundo plano).
* **Tareas en Segundo Plano (`BackgroundTasks`)**:
  * Usar para operaciones **no bloqueantes** después de enviar la respuesta (e.g., envío de correos, procesamiento de datos).
  * Evitar tareas de larga duración; usar colas de trabajo (job queues) adecuadas en su lugar.

---
