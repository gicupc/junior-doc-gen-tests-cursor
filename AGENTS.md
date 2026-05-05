# Project Instructions for AI Agents

Este proyecto usa el sistema **Architect-Brain** para garantizar calidad en codigo, documentacion, tests y base de datos.

## Documentacion principal

Antes de hacer cualquier cambio, consulta:
- `.cursor/rules/architect-brain.mdc` — reglas globales (cargadas automaticamente).
- `prompts/architect-brain.md` — protocolos completos del sistema.
- `inicio-chat.txt` — manual operativo con todos los comandos.

## Comandos disponibles

Escribe `/` en chat para ver los slash commands:
- `/inicio` — proyecto nuevo (entrevista BMADT + setup completo).
- `/onboarding` — codebase ajeno: mapa rapido en 6 secciones sin generar docs.
- `/regularizar` — proyecto existente sin docs.
- `/hoy` — briefing al retomar.
- `/nueva` — anadir funcionalidad con interrupcion de seguridad.
- `/reparar` — bugfix con test-rojo primero.
- `/decidir` — revisar decision con propagacion controlada.
- `/sync` — auditar drift entre codigo y docs.
- `/cerrar` — forzar checklist de sincronizacion.
- `/cuestionar` — auditar calidad de tests con mutation thinking.
- `/revisar-bd` — auditar base de datos (4 categorias).

Skills auxiliares (auto-invocables, sin slash):
- `tests-skill` — convenciones para generar tests por stack.
- `legacy-testing-skill` — characterization tests para legacy.
- `database-skill` — diseno y auditoria de BD.
- `prompt-skill` — manual del usuario para redactar prompts profesionales (patron RACEO).

## Reglas de oro

1. Toda la documentacion vive en `/docs/`. No crear subcarpetas `docs/` internas.
2. Los docs ganan sobre la memoria auto-generada del IDE.
3. Ningun ticket se cierra sin tests en verde + checklist visible.
4. Las decisiones no triviales se registran en `decisions-log.md` como ADRs inmutables.
5. En codigo legacy: primero congelar comportamiento con characterization tests, luego cambiar.
6. En BD: primero auditar, luego cambiar. Migraciones destructivas solo con patron Expand-Contract.
7. **Implementacion paso a paso:** si un plan toca mas de 2 archivos, aplicar `On(implementation_phase)` con paradas obligatorias entre archivos.
8. **Prompts profesionales:** consultar `prompt-skill` para redactar prompts complejos siguiendo el patron RACEO.

## SSOT (Single Source of Truth)

Los archivos `.md` de `/docs/` son la fuente de verdad del proyecto:
- `prd.md` — que es el producto y para quien.
- `architecture.md` — como esta montado tecnicamente.
- `blueprints.md` — patrones de codigo del proyecto.
- `user-stories.md` — funcionalidades con criterios testeables.
- `roadmap.md` — tickets y progreso.
- `testing-strategy.md` — framework, convenciones, cobertura de tests.
- `database-strategy.md` — motor, ORM, PII, migraciones (si hay BD).
- `decisions-log.md` — ADRs inmutables.
