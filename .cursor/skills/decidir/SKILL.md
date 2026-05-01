---
name: decidir
description: Cambiar o revisar una decision pasada (stack, testing, alcance, motor de BD, etc.) con propagacion controlada a los docs afectados. Crea nueva ADR y marca la antigua como Superseded sin perder historico.
disable-model-invocation: true
---

# /decidir — Revisar una decision

Ejecuta el protocolo `On(revisit_decision)` definido en `@prompts/architect-brain.md`.

Pasos:

1. Pregunta al usuario: "¿Que decision quieres revisar? Ejemplos:
   - El nivel de testing (T del BMADT).
   - El stack tecnico (M del BMADT).
   - El alcance del MVP (A del BMADT).
   - Los datos/entidades (D del BMADT).
   - El motor de BD o el ORM.
   - Otra decision concreta (dame el ADR-XXX si lo conoces)."

2. Lee `docs/decisions-log.md` y localiza la ADR afectada.

3. Pregunta al usuario SOLO lo que cambia. No rehagas la entrevista BMADT entera.

4. Calcula el impacto antes de tocar nada. Muestra al usuario una tabla:
   ```
   Archivos afectados por este cambio:
   - docs/[archivo].md — [que se modificaria]
   - docs/[archivo].md — [que se modificaria]
   - [codigo]/[archivo] — [si el cambio requiere modificar codigo]
   ```

5. Pide confirmacion explicita: "¿Procedo con estos cambios?"

6. Tras confirmar:
   - Anade una nueva ADR al final de `decisions-log.md` con el cambio.
   - Edita la ADR antigua marcando su campo "Estado" como `Superseded por ADR-XXX` (el contenido original se conserva intacto).
   - Propaga los cambios a los docs afectados.
   - Si el cambio genera trabajo pendiente, crea tickets en `roadmap.md`.

7. Cierra con el checklist de `On(task_complete)`.
