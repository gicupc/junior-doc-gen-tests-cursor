# Project Instructions for Claude Code

Este proyecto usa el sistema **Architect-Brain v4.4** para garantizar calidad en codigo, documentacion, tests, base de datos y frontend.

> **Nota:** Claude Code lee este archivo (`CLAUDE.md`); Cursor lee `.cursor/rules/architect-brain.mdc`. Este archivo apunta a las mismas fuentes de verdad que Cursor, para que el sistema funcione identico en ambos entornos. NO duplica contenido.

---

## Fuentes de verdad — leer ANTES de cualquier accion

Cuando inicies sesion en este proyecto, lee en este orden:

1. **`.cursor/rules/architect-brain.mdc`** — reglas globales operativas del sistema (mismo contenido que aplica Cursor con `alwaysApply: true`).
2. **`prompts/architect-brain.md`** — protocolos completos `On(...)`: project_start, testing_setup, database_setup, frontend_setup, database_audit, frontend_audit, task_complete, sync_check, ticket_close, implementation_phase, resume_project, revisit_decision.
3. **`AGENTS.md`** — resumen rapido y stack canonico.
4. **`/docs/`** — SSOT del proyecto concreto en el que estas: `prd.md`, `architecture.md`, `blueprints.md`, `user-stories.md`, `roadmap.md`, `testing-strategy.md`, `database-strategy.md` (si hay BD), `frontend-strategy.md` (si hay frontend), `decisions-log.md`.

---

## Comandos disponibles

Aunque Claude Code soporta `.claude/commands/` para slash commands nativos, este sistema vive en `.cursor/skills/` (compartido con Cursor). Cuando el usuario diga uno de estos comandos, lee el SKILL.md correspondiente y ejecuta su protocolo:

- `/inicio` → `.cursor/skills/inicio/SKILL.md`
- `/onboarding` → `.cursor/skills/onboarding/SKILL.md`
- `/regularizar` → `.cursor/skills/regularizar/SKILL.md`
- `/hoy` → `.cursor/skills/hoy/SKILL.md`
- `/nueva` → `.cursor/skills/nueva/SKILL.md`
- `/reparar` → `.cursor/skills/reparar/SKILL.md`
- `/decidir` → `.cursor/skills/decidir/SKILL.md`
- `/sync` → `.cursor/skills/sync/SKILL.md`
- `/cerrar` → `.cursor/skills/cerrar/SKILL.md`
- `/cuestionar` → `.cursor/skills/cuestionar/SKILL.md`
- `/revisar-bd` → `.cursor/skills/revisar-bd/SKILL.md`
- `/revisar-frontend` → `.cursor/skills/revisar-frontend/SKILL.md`

Skills auxiliares (cargar con `@` o automaticamente segun contexto):

- `.cursor/skills/tests-skill/SKILL.md` — convenciones de tests por stack.
- `.cursor/skills/legacy-testing-skill/SKILL.md` — characterization tests para legacy.
- `.cursor/skills/database-skill/SKILL.md` — diseno y auditoria de BD.
- `.cursor/skills/frontend-skill/SKILL.md` — diseno y auditoria de frontend.
- `.cursor/skills/visual-testing-skill/SKILL.md` — tests UI con axe + Playwright + visual diff.
- `.cursor/skills/prompt-skill/SKILL.md` — manual del usuario para redactar prompts profesionales (RACEO).

---

## MCP servers disponibles

Configurados en `.cursor/mcp.json` con plantillas comentadas listas para activar:

- **Postgres MCP** — BD de desarrollo (read-only obligatorio en produccion).
- **Figma Dev Mode MCP** — tokens y jerarquia desde Figma.
- **shadcn MCP** — acceso a >6.000 bloques de shadcn/ui.
- **Chrome DevTools MCP** — auditoria runtime DOM/CSS/performance/networking.
- **Playwright MCP** — automatizacion sobre accessibility tree para tests visuales y E2E.

Si Claude Code no tiene acceso al MCP por su configuracion local, explica al usuario que el MCP esta definido en `.cursor/mcp.json` para Cursor, y procede con analisis estatico cuando sea posible.

---

## Reglas de oro (resumen operativo)

Estas son las que aplicas tu, Claude Code, en cada turno. Para el detalle, lee `.cursor/rules/architect-brain.mdc`:

1. **SSOT**: los docs de `/docs/` ganan sobre tu memoria del proyecto.
2. **Antidrift**: ninguna tarea se cierra sin el checklist `On(task_complete)` visible.
3. **Tests como contrato**: ticket cerrado solo cuando los tests asociados estan en verde.
4. **Decisiones trazables**: si no esta en `decisions-log.md`, no existe como decision oficial.
5. **Legacy primero congelar, luego cambiar**: en codigo legacy, characterization tests antes de modificar.
6. **BD primero auditar, luego cambiar**: ejecutar `/revisar-bd` antes de cambios destructivos en BD.
7. **Frontend primero medir, luego optimizar**: ejecutar `/revisar-frontend` antes de optimizaciones a ojo. WCAG 2.2 AA y CWV son criterios objetivos.
8. **Implementacion paso a paso**: si el plan toca >2 archivos, aplicar `On(implementation_phase)` con paradas obligatorias entre archivos.
9. **Cero `any` en frontend**, cero secretos en `NEXT_PUBLIC_*`/`VITE_*`/`PUBLIC_*`, validacion cliente Y servidor con Zod.
10. **Codigo IA bajo revision**: outputs de v0/Lovable/Bolt/Figma Make pasan por revision humana antes de produccion.

---

## Interrupcion de seguridad (obligatoria antes de tocar codigo)

Antes de modificar, anadir o reparar codigo, pregunta:

```
He recibido tu peticion. Segun el protocolo de calidad:
1. ¿Actualizo docs/user-stories.md y docs/roadmap.md primero?
2. ¿Hay nuevas reglas para docs/blueprints.md?
3. ¿Los tests asociados estan planificados en docs/testing-strategy.md?
4. ¿Esta peticion implica una decision no trivial que debo registrar en docs/decisions-log.md?
5. ¿Esta peticion toca la base de datos? Si es si, ¿debo actualizar docs/database-strategy.md?
6. ¿Esta peticion toca el frontend (componentes, paginas, estilos, a11y)? Si es si, ¿debo actualizar docs/frontend-strategy.md?

No tocare el codigo hasta que confirmes la sincronizacion documental.
```

---

## Cierre de tarea (obligatorio)

Al terminar CUALQUIER tarea de codigo, muestra el bloque `On(task_complete)` definido en `prompts/architect-brain.md`. Sin ese bloque visible, la tarea NO esta cerrada.
