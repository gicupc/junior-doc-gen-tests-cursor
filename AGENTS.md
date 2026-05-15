# Project Instructions for AI Agents

Este proyecto usa el sistema **Architect-Brain v4.6** para garantizar calidad en codigo, documentacion, tests, base de datos, frontend (accesibilidad + Core Web Vitals + seguridad) y cadena de suministro (supply chain).

Este archivo sigue el estandar `AGENTS.md` (donado al Linux Foundation en diciembre 2025, soportado nativamente por Cursor, Windsurf, Copilot, Codex, Zed, Warp, Aider, Devin y otros). Para Cursor, las reglas activas estan en `.cursor/rules/architect-brain.mdc` con `alwaysApply: true`.

## Documentacion principal

Antes de hacer cualquier cambio, consulta:
- `.cursor/rules/architect-brain.mdc` — reglas globales (cargadas automaticamente por Cursor en cada chat).
- `prompts/architect-brain.md` — protocolos completos del sistema.
- `inicio-chat.txt` — manual operativo con todos los comandos.

## Comandos disponibles

Escribe `/` en el chat de Agent para ver los slash commands:

- `/inicio` — proyecto nuevo (entrevista BMADT + setup completo).
- `/onboarding` — codebase ajeno: mapa rapido en 6 secciones sin generar docs.
- `/regularizar` — proyecto existente sin docs. Opcion [4] incluye auditoria BD y frontend.
- `/hoy` — briefing al retomar.
- `/nueva` — anadir funcionalidad con interrupcion de seguridad.
- `/reparar` — bugfix con test-rojo primero.
- `/decidir` — revisar decision con propagacion controlada.
- `/sync` — auditar drift entre codigo y docs.
- `/cerrar` — forzar checklist de sincronizacion.
- `/cuestionar` — auditar calidad de tests con mutation thinking.
- `/revisar-bd` — auditar base de datos (4 categorias).
- `/revisar-frontend` — auditar frontend (stack, seguridad, CWV, a11y WCAG 2.2 AA).
- `/bdd` — adoptar Behavior-Driven Development (verifica aplicabilidad, instala herramienta segun stack 2026, primera sesion Three Amigos asistida, valida `.feature` antes de step definitions). *(nuevo en v4.5)*
- `/revisar-supply-chain` — auditar cadena de suministro: dependencias, lockfile, scripts, GitHub Actions, publish path. Cross-referencia con incidentes publicos (TanStack, Shai-Hulud, axios, Trivy). *(nuevo en v4.6)*

Skills auxiliares (auto-invocables, sin slash):
- `tests-skill` — convenciones para generar tests unitarios por stack.
- `integration-testing-skill` — tests de integracion con Supertest, MSW, Testcontainers, Pact. *(nuevo en v4.5)*
- `legacy-testing-skill` — characterization tests para legacy.
- `database-skill` — diseno y auditoria de BD.
- `frontend-skill` — diseno y auditoria de frontend.
- `visual-testing-skill` — tests de componente con axe + E2E con Playwright + visual diff.
- `bdd-skill` — Gherkin, herramientas 2026 (playwright-bdd, @cucumber/cucumber, Reqnroll, Karate DSL), Three Amigos, anti-patrones de Gherkin por IA. *(nuevo en v4.5)*
- `ai-testing-skill` — testing asistido por IA (Playwright MCP/CLI, prompting, anti-patrones, GDPR + EU AI Act). *(nuevo en v4.5)*
- `supply-chain-skill` — defensas en 4 capas (minimumReleaseAge, allowBuilds, GitHub Actions hardening, OIDC pinning), respuesta a incidentes con orden critico (TanStack, Shai-Hulud). *(nuevo en v4.6)*
- `prompt-skill` — manual del usuario para redactar prompts profesionales (patron RACEO).

## Stack canonico para proyectos NUEVOS con frontend (2026)

Salvo desviacion documentada en ADR:

- **Framework**: Next.js 15/16 (App Router, RSC, Server Actions).
- **Lenguaje**: TypeScript estricto (`strict: true`, `noUncheckedIndexedAccess: true`).
- **Estilado**: Tailwind v4 (CSS-first, container queries en core).
- **Componentes**: shadcn/ui (Radix + Tailwind).
- **Forms y validacion**: react-hook-form + Zod (schemas compartidos cliente/servidor).
- **i18n**: next-intl o paraglide.
- **Linter**: Biome v2 (recomendado) o ESLint 9 flat config + Prettier.
- **Testing**: Vitest + RTL + vitest-axe + Playwright + @axe-core/playwright.
- **RUM**: Vercel Speed Insights, Cloudflare Web Analytics o Sentry.

Razon: este stack es el que las herramientas IA generan mejor por defecto en 2026. Mantenerse en el reduce drasticamente el rate de errores de generacion.

## MCP servers recomendados (segun escenario)

Configuracion en `.cursor/mcp.json` (ya incluye plantillas comentadas).

| MCP server | Cuando | Que aporta |
|---|---|---|
| Figma Dev Mode MCP | Diseno en Figma con Dev Mode | tokens, jerarquia y component mappings estructurados |
| shadcn MCP | Stack incluye shadcn/ui | acceso a >6.000 bloques |
| Chrome DevTools MCP | Auditoria runtime | DOM/CSS/performance/networking |
| Playwright MCP | Validacion visual + E2E | accessibility tree determinista |
| Postgres MCP (read-only en prod) | BD Postgres | inventario y queries auditables |

## Reglas de oro

1. Toda la documentacion vive en `/docs/`. No crear subcarpetas `docs/` internas.
2. Los docs ganan sobre la memoria auto-generada del IDE.
3. Ningun ticket se cierra sin tests en verde + checklist visible.
4. Las decisiones no triviales se registran en `decisions-log.md` como ADRs inmutables.
5. En codigo legacy: primero congelar comportamiento con characterization tests, luego cambiar.
6. En BD: primero auditar, luego cambiar. Migraciones destructivas solo con patron Expand-Contract.
7. **En frontend: primero medir, luego optimizar.** WCAG 2.2 AA como minimo, CWV como Definition of Done. Cero `any`, cero secretos en variables publicas, validacion cliente Y servidor con Zod.
8. **Implementacion paso a paso:** si un plan toca mas de 2 archivos, aplicar `On(implementation_phase)` con paradas obligatorias entre archivos.
9. **Prompts profesionales:** consultar `prompt-skill` para redactar prompts complejos siguiendo el patron RACEO.
10. **Codigo IA bajo revision:** outputs de v0/Lovable/Bolt/Figma Make pasan obligatoriamente por revision humana en Cursor/Windsurf/Claude Code antes de produccion.
11. **BDD sin Three Amigos NO se adopta** *(nuevo en v4.5)*: Gherkin sin conversacion previa con stakeholders es disfraz tecnico. Si no se cumplen las precondiciones, registrar ADR "BDD descartado" y volver mas adelante.
12. **Tests IA trazables** *(nuevo en v4.5)*: cada test generado por LLM tiene su `*.prompt.md` asociado, pasa code review humano y NUNCA se mergea por auto-aprobacion. Queries accesibles obligatorias (cero selectores fragiles).
13. **AI literacy + GDPR** *(nuevo en v4.5)*: EU AI Act exige formacion en uso responsable de IA desde febrero 2025. NO enviar PII a LLMs externos sin DPA valido. Para apps de alto riesgo (Annex III), los tests forman parte del expediente de calidad.
14. **Supply chain en 4 capas** *(nuevo en v4.6)*: `minimumReleaseAge` (cooldown 1+ dia) + `allowBuilds` (deny by default) + GitHub Actions hardening (NO `pull_request_target` con checkout de fork, acciones pineadas a SHA) + OIDC pinning a workflow+branch. Provenance SLSA valido NO es garantia (lo demostro el ataque TanStack del 11 mayo 2026).
15. **Respuesta a incidente con orden critico** *(nuevo en v4.6)*: si se detecta paquete comprometido, NO revocar token de GitHub antes de limpiar daemon de persistencia. Algunos payloads ejecutan `rm -rf ~/` al detectar revocacion.

## SSOT (Single Source of Truth)

Los archivos `.md` de `/docs/` son la fuente de verdad del proyecto:
- `prd.md` — que es el producto y para quien.
- `architecture.md` — como esta montado tecnicamente.
- `blueprints.md` — patrones de codigo del proyecto.
- `user-stories.md` — funcionalidades con criterios testeables.
- `roadmap.md` — tickets y progreso.
- `testing-strategy.md` — framework, convenciones, cobertura de tests.
- `database-strategy.md` — motor, ORM, PII, migraciones (si hay BD).
- `frontend-strategy.md` — stack, MCP, a11y, CWV, visual testing (si hay frontend).
- `supply-chain-strategy.md` — package manager, defensas 4 capas, plan de respuesta a incidentes (si usa npm registry).
- `decisions-log.md` — ADRs inmutables.
