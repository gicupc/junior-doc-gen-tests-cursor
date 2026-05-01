---
name: hoy
description: "Que hacemos hoy" - briefing de 10 lineas sobre el estado del proyecto. Lee los docs clave, ejecuta health-check y propone una accion concreta para el dia. Util al retomar un proyecto tras una pausa.
disable-model-invocation: true
---

# /hoy — Que hacemos hoy

Ejecuta el protocolo `On(resume_project)` definido en `@prompts/architect-brain.md`.

Pasos:

1. Lee en este orden: `docs/prd.md` (solo el objetivo), `docs/roadmap.md` completo, `docs/decisions-log.md` (ultimas 3 entradas), `docs/testing-strategy.md` (estado de cobertura), `docs/database-strategy.md` (si existe).

2. Ejecuta mini health-check:
   - Corre la suite de tests; anota cuantos pasan/fallan/skipped.
   - Verifica que el ultimo ticket `[x]` tiene commits asociados.
   - Busca archivos en codigo que no esten mencionados en `architecture.md`.
   - Detecta tests en `.skip` con mas de 7 dias.
   - Si hay BD: detecta migraciones pendientes sin aplicar.

3. Presenta un briefing de maximo 10 lineas con este formato:

```
📋 Proyecto: [Nombre]
🎯 Objetivo: [1 frase del PRD]
📈 Progreso: [X/Y tickets completados — Z%]
✅ Ultimo cerrado: [titulo del ultimo ticket con [x]]
👉 Siguiente: [titulo del proximo ticket pendiente]
🧪 Tests: [N passing / M skipped / K failing]
💡 Ultimas decisiones: [2-3 ultimas ADRs por titulo]
⚠️ Warnings: [drift detectado, o "ninguno"]
🚀 Sugerencia: [accion concreta recomendada para hoy]
```

4. Termina con: "¿Empezamos por la sugerencia, o prefieres otra cosa?"

No ejecutes ninguna accion de codigo. Solo briefing.
