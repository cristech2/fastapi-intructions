# Instrucciones Optimizadas de GitHub Copilot para Python, FastAPI y Seguridad

Este repositorio tiene como objetivo **construir y mantener un conjunto de instrucciones personalizadas** para GitHub Copilot, enfocadas en el desarrollo de alto rendimiento y seguro con Python y FastAPI, a su vez optimizar el uso de tokens

## 🎯 Propósito del Proyecto

El objetivo principal es **generar** un conjunto de archivos de instrucciones curados y listos para usar, que se encuentran en el directorio `instructions/`.

Este proyecto *no* es solo una colección de notas. Sigue un proceso claro:

1. **Fuente de Conocimiento (`/references`):** El directorio `references/` actúa como nuestra base de conocimiento central. Contiene una colección detallada de mejores prácticas, convenciones, patrones de código y principios de seguridad. **Este material es la fuente, no el producto final.**
2. **Producto Final (`/instructions`):** El directorio `instructions/` contiene los archivos `.md` **optimizados para GitHub Copilot**. Estos archivos son versiones curadas, resumidas y formateadas del material de `references/`, diseñadas para ser eficientes en tokens y proporcionar directrices claras y accionables a la IA.

## 📜 Convenciones de las Instrucciones

Todos los archivos generados en el directorio `/instructions` siguen un conjunto estricto de convenciones para asegurar la compatibilidad y eficacia con GitHub Copilot.

### Frontmatter Requerido

Cada archivo de instrucción **debe** comenzar con un bloque YAML frontmatter que define su alcance.

```yaml
---
description: 'Descripción breve del propósito y alcance de la instrucción.'
applyTo: 'patrón glob para los archivos objetivo (ej. **/*.py)'
---
```

* **description**: Debe ser una cadena clara y concisa (1-500 caracteres) que explique el propósito del archivo.
* **applyTo**: Define un patrón glob que especifica a qué archivos se aplican estas instrucciones (ej. `**/*.py`, `'src/api/**/*.ts'`, o `**` para todos).

### Nomenclatura y Estructura

* **Nomenclatura de Archivos**: Los archivos deben usar `minúsculas-con-guiones`, terminando en `.instructions.md` (ej. `fastapi-rutas-api.instructions.md`).
* **Estructura Interna**: Los archivos deben seguir una estructura lógica:
    1. **Título (`#`)**: Un título claro y descriptivo.
    2. **Introducción**: Un breve párrafo explicando el propósito.
    3. **Secciones Centrales**: Contenido organizado por temas (ej. `Mejores Prácticas`, `Estándares de Código`, `Patrones Comunes`).
    4. **Ejemplos**: Uso de bloques de código claros, preferiblemente con etiquetas `### Good Example` y `### Bad Example`.

### Estilo de Redacción

Para maximizar la comprensión de Copilot, el contenido debe:

* **Ser claro y conciso**: Evitar explicaciones demasiado extensas.
* **Usar el modo imperativo**: Emplear comandos directos ("Usar", "Implementar", "Evitar").
* **Ser específico y accionable**: Proporcionar ejemplos concretos en lugar de conceptos abstractos.
* **Evitar la ambigüedad**: No usar términos como "debería", "podría" o "posiblemente".
* **Usar tablas y listas**: Para comparar opciones o enumerar reglas de forma estructurada.
