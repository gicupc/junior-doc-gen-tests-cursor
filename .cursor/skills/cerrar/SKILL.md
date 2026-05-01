---
name: cerrar
description: Forzar el checklist de sincronizacion documental al cerrar tarea. Util cuando se sospecha que una tarea anterior se cerro sin el checklist correcto. Ejecuta On(task_complete) explicitamente.
disable-model-invocation: true
---

# /cerrar — Cerrar tarea con checklist

Ejecuta el protocolo `On(task_complete)` de forma explicita.

Normalmente este protocolo se dispara solo al finalizar cualquier tarea, pero puedes invocarlo manualmente cuando sospeches que una tarea se cerro sin el checklist correcto.

Pasos:

1. Lista los archivos de codigo modificados en la sesion actual (o en el ultimo commit si la sesion es nueva).

2. Para cada uno de los siguientes docs, indica explicitamente que cambio o si no aplica:
   - `user-stories.md`
   - `roadmap.md`
   - `blueprints.md`
   - `architecture.md`
   - `testing-strategy.md`
   - `database-strategy.md` (si existe)
   - `decisions-log.md`

3. Sobre tests:
   - Lista los tests anadidos o modificados.
   - Ejecuta la suite y reporta passing/failing/skipped.

4. Presenta el bloque completo con este formato:

```
📋 Sincronizacion documental

Archivos de codigo modificados:
- [lista]

Documentacion:
- [x] user-stories.md — [descripcion del cambio, o "sin cambios"]
- [x] roadmap.md — [descripcion del cambio, o "sin cambios"]
- [x] blueprints.md — [descripcion del cambio, o "sin cambios"]
- [x] architecture.md — [descripcion del cambio, o "sin cambios"]
- [x] testing-strategy.md — [descripcion del cambio, o "sin cambios"]
- [x] database-strategy.md — [descripcion del cambio, o "sin cambios" / "no aplica"]
- [x] decisions-log.md — [descripcion del cambio, o "sin cambios"]

Tests:
- [x] Anadidos/modificados: [lista o "ninguno aplicable"]
- [x] Suite ejecutada: [N passing / M failing / K skipped]

Warnings:
- [si algo quedo a medias; si no hay warnings: "ninguno"]
```

5. Si detectas que hay docs que DEBERIAN haberse actualizado pero no lo hicieron, NO declares la tarea cerrada. Pide permiso para anadir las actualizaciones faltantes.
