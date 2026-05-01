---
name: reparar
description: Corregir un bug puntual respetando blueprints y testing-strategy. Si el bug no esta cubierto por tests, OBLIGA a escribir primero un test que lo reproduzca (rojo) antes de implementar el fix (verde). Convierte cada bug en una garantia de no-regresion.
disable-model-invocation: true
---

# /reparar — Reparacion rapida

Voy a reportar un bug. Sigue el protocolo Fix.

Pasos:

1. Pregunta al usuario: "¿En que modulo esta el bug y cual es el comportamiento erroneo? Si puedes, incluye pasos para reproducirlo."

2. Consulta obligatoriamente antes de tocar codigo:
   - `docs/blueprints.md` — estilo de codigo del proyecto.
   - `docs/testing-strategy.md` — convenciones de tests.
   - Si el bug toca BD: `docs/database-strategy.md` y carga `@.cursor/skills/database-skill/SKILL.md`.

3. **Paso critico**: si el bug NO esta cubierto por ningun test actual:
   - Escribe PRIMERO un test que reproduzca el bug.
   - Ejecutalo: debe ponerse en rojo (confirmando que el bug es reproducible).
   - SOLO despues, implementa el fix.
   - Vuelve a ejecutar el test: debe ponerse en verde.
   - Ejecuta toda la suite: todos los tests deben seguir en verde.

4. Si el bug SI estaba cubierto por tests existentes pero estos estaban mal escritos:
   - Documenta en el commit por que el test anterior no detectaba el bug.
   - Ajusta el test para cubrir el caso correctamente.

5. **Si la reparacion implica una migracion destructiva en BD** (DROP COLUMN, ALTER TYPE con perdida, RENAME):
   - APLICA OBLIGATORIAMENTE el patron Expand-Contract:
     1. Expand: anadir columna nueva (nullable), escribir en ambas.
     2. Backfill: rellenar la nueva con datos de la vieja.
     3. Migrate reads: cambiar lecturas a la nueva.
     4. Contract: DROP de la vieja en una migracion separada.
   - Nunca aplicar el cambio destructivo en una sola migracion en produccion.

6. Al cerrar, ejecuta OBLIGATORIAMENTE `On(task_complete)` con el checklist completo:
   - Normalmente un bugfix no cambia user-stories, pero puede anadir un ticket retroactivo al roadmap si el bug revela una laguna.
   - Si el bug surgio de una decision previa (ej. eleccion de libreria con limitacion conocida), registra una ADR explicando el workaround.

No cierres la tarea si el checklist no se ha mostrado ni si algun test esta en rojo.
