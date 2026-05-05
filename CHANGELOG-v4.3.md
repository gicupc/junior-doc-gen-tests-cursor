# Changelog v4.2 → v4.3 (Cursor Edition)

Sesión de actualización: 5 de mayo de 2026

Origen: paridad con la versión Windsurf v4.3 (`junior-doc-gen-tests`). Se portan a Cursor todas las mejoras nacidas de la práctica del módulo 9 (AI4Devs) sobre endpoints LTI con asistente IA.

---

## Cambios

### 1. Nuevo skill: `.cursor/skills/prompt-skill/SKILL.md`

**Estado:** archivo nuevo (skill auxiliar auto-invocable, sin `disable-model-invocation`).

Manual operativo para el **usuario** sobre cómo redactar prompts profesionales siguiendo el patrón **RACEO** (Role + Objective + Context + Constraints + Expected Output). Incluye:

- Por qué cada bloque del patrón.
- Plantilla base copy-paste.
- 3 ejemplos de prompts completos (onboarding, planificación, implementación paso a paso).
- 5 antipatrones del usuario.
- Cuándo NO usar RACEO (tareas triviales).

Origen: el patrón observado en las PRs destacadas del módulo 8 (Arnau, Alberto, Fran).

### 2. Skill nuevo: `.cursor/skills/onboarding/SKILL.md`

**Estado:** archivo nuevo (slash command con `disable-model-invocation: true`).

Comando `/onboarding` para entrar a un codebase ajeno sin crear docs ni adoptar el proyecto a largo plazo. Diferencia explícita con `/regularizar`:

- `/regularizar` → adopción del proyecto, blueprints, ADRs, plan de cobertura.
- `/onboarding` → mapa puntual en 6 secciones: stack, capas, traza de un endpoint de referencia, modelo de datos, tests, convenciones. Cierra con TODOs verificables por el usuario.

Detecta automáticamente antipatrones existentes (controllers huérfanos, rutas que bypasean capas) y los reporta — esto condiciona la planificación posterior.

### 3. Modificado: `.cursor/skills/tests-skill/SKILL.md`

**Estado:** sección de mock de Prisma reemplazada.

El ejemplo anterior usaba `mockImplementation(() => ({...}))` que **falla con código de producción Active Record** (cuando cada modelo de dominio instancia su propio `PrismaClient`). El ejemplo nuevo usa `jest.fn(() => mPrisma)` que garantiza que todas las invocaciones a `new PrismaClient()` devuelven la misma instancia, y por tanto el mock alcanza al prisma interno del modelo.

Se añade la regla: **ante la duda, usar el Patrón A** (con `mockReturnValue` / `jest.fn(() => obj)`). Funciona tanto con repositorios separados como con Active Record.

Razón del cambio: durante la implementación del módulo 9 se detectó que el patrón anterior no permitía mockear Prisma cuando `Position.ts` instanciaba su propio cliente (Active Record).

### 4. Modificado: `prompts/architect-brain.md`

**Estado:** versión actualizada de v4.2 a v4.3 (Cursor Edition + Implementation Phase) + nuevo protocolo.

Añadido `On(implementation_phase)` justo antes de `On(task_complete)`. Este protocolo se activa cuando un ticket pasa de planificación a implementación con plan previo. Exige:

- Anunciar el plan de pasos al usuario antes de tocar nada.
- Detenerse **tras cada paso** (1 archivo = 1 paso) y esperar confirmación explícita "ok, sigue".
- Marcador visible al final de cada paso: `Paso K/N completado. Esperando "ok, sigue".`
- Si encuentra un hueco no previsto, **parar y preguntar**, no improvisar.
- Paso final siempre es `tests + ejecución de la suite` con output completo.

Aplicabilidad: si el plan implica más de 2 archivos, aplica obligatoriamente. Bugfixes triviales no aplican.

### 5. Modificado: `.cursor/rules/architect-brain.mdc`

**Estado:** dos filas nuevas en la tabla de comandos del usuario + dos reglas de oro nuevas.

- Disparador para `/onboarding` ("dame un mapa rapido", "no conozco este codigo").
- Disparador para `On(implementation_phase)` ("implementa el plan", "vamos al codigo").
- Regla de oro **Implementacion paso a paso**: si el plan toca >2 archivos, aplica el protocolo.
- Regla de oro **Prompts profesionales**: el usuario consulta `prompt-skill` para tareas complejas.

### 6. Modificado: `AGENTS.md`

**Estado:** lista de slash commands actualizada con `/onboarding`, sección nueva de skills auxiliares con `prompt-skill`, dos reglas de oro nuevas (paso a paso + prompts profesionales).

### 7. Modificado: `README.md` e `inicio-chat.txt`

**Estado:** tablas de comandos actualizadas con `/onboarding`. Estructura de archivos actualizada (14 skills total: 11 commands + 3 auxiliares + 1 manual usuario). Nuevo pilar 7 "Implementación disciplinada paso a paso" en README. Reglas de oro 11 y 12 añadidas.

### 8. Modificado: `docs/architecture.md`

**Estado:** reescrito como meta-documentación del scaffolding v4.3 (Cursor Edition).

Antes era un documento mixto (mitad meta-doc, mitad plantilla). Ahora describe explícitamente la arquitectura del propio sistema — las 4 capas (Intent / Implementation / Memory / Lifecycle), los 9 protocolos centrales (incluyendo `On(implementation_phase)` nuevo), las garantías del sistema, y la distinción entre **skills tipo slash command**, **skills auxiliares auto-invocables** y **skills dinámicas** (las que crea el Agent durante el proyecto).

Añade el estado `[ONBOARDING]` y formaliza `[IMPLEMENTING]` en el ciclo de vida.

### 9. Modificado: `docs/testing-strategy.md`

**Estado:** sección "Patrón de mock para BD" ampliada con dos snippets de referencia (Prisma+Jest con el patrón Active-Record-friendly, y PDO+PHPUnit). El placeholder original del proyecto se mantiene; los snippets aparecen como referencia bajo el placeholder.

Razón: que los proyectos que usen el sistema tengan ya el patrón correcto a la vista cuando rellenen su propia testing-strategy.

### 10. Modificado: `docs/decisions-log.md`

**Estado:** sección nueva "Decisiones de comportamiento ad-hoc" justo antes de la plantilla ADR-000.

Documenta el patrón aprendido: cuando una feature compleja tiene preguntas semánticas no cubiertas por el enunciado, se etiquetan como C1/C2/C3 durante la planificación, se confirman con el usuario antes de implementar y se registran como ADR. Esto evita el clásico "porque la IA lo decidió así" en code reviews.

---

## Migración

Para integrar v4.3 en tu repo local:

1. Reemplaza la carpeta entera `junior-doc-gen-tests-cursor/` con la del repo actualizado.
2. O bien, añade/reemplaza solo estos archivos individualmente:
   - `.cursor/skills/onboarding/SKILL.md` (nuevo)
   - `.cursor/skills/prompt-skill/SKILL.md` (nuevo)
   - `.cursor/skills/tests-skill/SKILL.md` (modificado)
   - `.cursor/rules/architect-brain.mdc` (modificado)
   - `prompts/architect-brain.md` (modificado)
   - `AGENTS.md` (modificado)
   - `README.md` (modificado)
   - `inicio-chat.txt` (modificado)
   - `docs/architecture.md` (reescrito)
   - `docs/testing-strategy.md` (modificado)
   - `docs/decisions-log.md` (modificado)

No hay cambios destructivos. Las skills existentes (`/inicio`, `/regularizar`, `/cuestionar`, etc.) siguen funcionando igual. El sistema es retrocompatible.

---

## Verificación post-migración

Para confirmar que todo está bien:

1. Abre Cursor en un proyecto cualquiera con esta estructura.
2. En el chat de Agent escribe `/` — deberían aparecer **11 slash commands** en el desplegable, incluido `/onboarding`.
3. Pide al Agent: "Lee `@.cursor/skills/prompt-skill/SKILL.md` y dime qué cubre."
   - Si te resume RACEO + ejemplos → integrado correctamente.
4. Pide: "Si te paso un plan de implementación de 5 archivos, ¿qué protocolo activas?"
   - Debería responder: `On(implementation_phase)` con paradas obligatorias entre pasos.

---

## Paridad con la versión Windsurf

Esta versión iguala funcionalmente a `junior-doc-gen-tests` v4.3 (Windsurf). Las únicas diferencias son las propias de cada IDE:

| Concepto | Windsurf v4.3 | Cursor v4.3 |
|---|---|---|
| Reglas globales | `.windsurfrules` | `.cursor/rules/architect-brain.mdc` con `alwaysApply: true` |
| Slash commands | `.windsurf/workflows/*.md` con frontmatter | `.cursor/skills/*/SKILL.md` con `disable-model-invocation: true` |
| Skills auxiliares | `prompts/skills/*.md` (carga manual con `@`) | `.cursor/skills/*/SKILL.md` (auto-invocables) |
| MCP servers | `~/.codeium/windsurf/mcp_config.json` | `.cursor/mcp.json` o `~/.cursor/mcp.json` |
| Resumen para agentes | (no existe) | `AGENTS.md` (estilo Cursor) |

Todo lo demás (cerebro `architect-brain.md`, docs SSOT, protocolos `On(...)`, comandos disponibles, reglas de oro) es idéntico entre ambas versiones.
