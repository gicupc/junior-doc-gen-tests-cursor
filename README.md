# junior-doc-gen-tests-cursor

Sistema de scaffolding para [Cursor](https://cursor.com) que convierte al Agent en un arquitecto senior disciplinado. Mantiene **documentación**, **código**, **tests**, **decisiones**, **base de datos** y **frontend** (accesibilidad + Core Web Vitals + seguridad) sincronizados durante todo el ciclo de vida del proyecto, evitando el clásico *documentation drift*.

> **Versión Cursor** del sistema. Existe también la [versión Windsurf](https://github.com/gicupc/junior-doc-gen-tests).

---

## Qué resuelve

Cuando trabajas con una IA generando código día tras día, aparecen ocho problemas silenciosos:

1. **Los docs se quedan atrás del código** — nadie los actualiza porque no duele inmediatamente.
2. **Las decisiones técnicas se olvidan** — ¿por qué elegimos esta librería? Nadie se acuerda tres meses después.
3. **Los tests son un "luego" eterno** — se añaden al final, si sobra tiempo, con calidad baja.
4. **El código legacy se toca sin red de seguridad** — cambiar primero y rezar después es receta para bugs nuevos.
5. **Cobertura ≠ calidad** — tests que pasan no garantizan detectar bugs reales.
6. **Las bases de datos crecen sin auditoría** — índices faltantes, datos personales sin proteger, queries lentas que nadie detecta.
7. **El frontend acumula deuda invisible** — accesibilidad rota, Core Web Vitals en rojo, secretos expuestos en variables públicas, código IA sin revisar.
8. **Los tests generados por IA son test theater** — cobertura alta, capacidad de detectar regresiones baja, sin trazabilidad de qué prompt los generó.

Este sistema impone protocolos que obligan a la IA a resolver los ocho problemas por diseño.

---

## Cómo se usa en 30 segundos

1. Copia esta estructura completa (`.cursor/`, `docs/`, `prompts/`, `AGENTS.md`, `inicio-chat.txt`) en la raíz de tu proyecto.
2. Abre Cursor apuntando a la raíz del proyecto.
3. En el chat de Agent, escribe `/` y te aparecerá un menú con los comandos disponibles.
4. Elige el comando que toque según la situación.

La rule `architect-brain.mdc` se carga automáticamente con `alwaysApply: true`, así que las reglas globales del sistema están presentes en cada chat sin necesidad de invocarlas.

---

## Slash commands disponibles (13)

| Comando             | Cuándo usarlo                                                                 |
|---------------------|-------------------------------------------------------------------------------|
| `/inicio`           | Proyecto vacío. Entrevista BMADT + docs + setup tests + setup BD si aplica + setup FRONTEND si aplica. |
| `/onboarding`       | Codebase ajeno (ejercicio bootcamp, PR de un compañero, debug urgente). Mapa rápido en 6 secciones, sin generar docs. |
| `/regularizar`      | Proyecto con código pero sin docs. Menú [1][2][3][4]. La opción [4] incluye auditoría de BD y frontend. |
| `/hoy`              | "¿Qué hacemos hoy?" Briefing con sugerencia concreta para el día.             |
| `/nueva`            | Añadir funcionalidad. Activa la interrupción de seguridad antes de tocar código. |
| `/reparar`          | Corregir un bug. Si no está cubierto, primero escribe el test que lo reproduce. |
| `/decidir`          | Cambiar una decisión pasada. Propaga cambios sin perder histórico.            |
| `/sync`             | Detectar drift entre código y documentación. Solo reporta.                    |
| `/cerrar`           | Forzar checklist de sincronización si sospechas que se saltó.                 |
| `/cuestionar`       | Auditar la calidad real de los tests con mutation thinking.                   |
| `/revisar-bd`       | Auditar la BD: modelo, seguridad, rendimiento, calidad de datos.              |
| `/revisar-frontend` | Auditar el frontend: stack, seguridad, Core Web Vitals, accesibilidad WCAG 2.2 AA. |
| `/bdd`              | Adoptar Behavior-Driven Development (verifica aplicabilidad, instala herramienta según stack 2026, primera sesión Three Amigos asistida). *(nuevo en v4.5)* |

---

## Cómo funciona en Cursor

### Rules y skills (la diferencia con Windsurf)

Cursor unifica `rules` y `commands` bajo el concepto de **skills**. Esta versión aprovecha esa arquitectura:

- **`.cursor/rules/architect-brain.mdc`** con `alwaysApply: true` → reglas globales cargadas en cada chat.
- **`.cursor/skills/<nombre>/SKILL.md`** con `disable-model-invocation: true` → comportan como slash commands clásicos. Solo se invocan manualmente con `/`.
- **`.cursor/skills/<auxiliar>/SKILL.md`** sin esa línea → la IA las carga automáticamente cuando detecta que aplican (por ejemplo, `database-skill` cuando trabajas con BD, `frontend-skill` cuando trabajas con UI).

### MCP servers

El archivo `.cursor/mcp.json` incluye plantillas comentadas para:

- **Postgres** (BD local de desarrollo).
- **Figma Dev Mode MCP** *(nuevo v4.4)* — tokens y jerarquía estructurados desde Figma.
- **shadcn MCP** *(nuevo v4.4)* — acceso a >6.000 bloques de shadcn/ui.
- **Chrome DevTools MCP** *(nuevo v4.4)* — auditoría runtime de DOM/CSS/performance/networking.
- **Playwright MCP** *(nuevo v4.4)* — automatización sobre accessibility tree para tests visuales y E2E.

Para activar uno: renombra la clave del bloque (de `_figma-dev-mode` a `figma-dev-mode`), rellena los placeholders y reinicia Cursor.

**Regla crítica de seguridad**: en producción, los MCPs de BD deben ir en modo **read-only**. Para desarrollo local con datos de prueba, full-access es aceptable.

---

## Qué hace el sistema (los 9 pilares)

### 1. Entrevista inicial estructurada (BMADT)

Al arrancar un proyecto nuevo con `/inicio`, la IA te hace 5 preguntas:
- **B (Beneficio):** objetivo y valor del proyecto.
- **M (Mecanismo):** stack técnico e integraciones.
- **A (Alcance):** módulos del MVP.
- **D (Datos):** entidades y flujos críticos.
- **T (Testing):** nivel de cobertura deseado (básico/estándar/exhaustivo).

Con las respuestas genera automáticamente los docs del SSOT en `/docs/` y **no escribe código** hasta que apruebes.

### 2. Setup automático del entorno de tests

Según el stack detectado, instala el framework adecuado con su soporte de linter (PHPUnit + phpstan-phpunit, Jest + @types/jest, Vitest, pytest, etc.). Si hay frontend con nivel T 2 o 3, añade además RTL + vitest-axe + Playwright + @axe-core/playwright.

### 3. Setup automático de base de datos

Si la respuesta M o D del BMADT implica BD, dispara automáticamente `On(database_setup)`: detecta motor + ORM, identifica campos PII, genera `database-strategy.md`, configura auditoría futura, prepara Testcontainers, registra ADRs.

### 4. Setup automático de frontend *(nuevo en v4.4)*

Si la respuesta M implica frontend (React, Vue, Svelte, Next, Nuxt, Astro, etc.), dispara automáticamente `On(frontend_setup)`:

- **Propone el stack canónico 2026**: Next.js 15/16 + TypeScript estricto + Tailwind v4 + shadcn/ui + Biome + Zod. Cualquier desviación se registra como ADR.
- **Pregunta por el flujo diseño-a-código** (Figma con Dev Mode, capturas, sin diseño) y **recomienda los MCP servers** apropiados (descomentando bloques en `.cursor/mcp.json`).
- **Configura headers de seguridad** (CSP, HSTS, X-Frame-Options, Referrer-Policy).
- **Establece reglas operativas** que entran en el ADR de stack: cero `any`, componentes <150 líneas, WCAG 2.2 AA como mínimo legal, Core Web Vitals como Definition of Done, validación cliente Y servidor con Zod, cero secretos en `NEXT_PUBLIC_*`/`VITE_*`/`PUBLIC_*`.
- **Genera `frontend-strategy.md`** con todas las decisiones documentadas.
- **Registra ADRs** por cada decisión (stack, a11y, MCP, RUM).

### 5. Trazabilidad de decisiones (ADRs)

Todas las decisiones no triviales se registran en `decisions-log.md` como **Architecture Decision Records** inmutables. Si cambias de opinión, se añade una nueva ADR y la antigua queda marcada como `Superseded por ADR-XXX`.

### 6. Antidrift documental

Al cerrar **cualquier tarea** de código, la IA muestra un checklist visible marcando cada uno de los docs del SSOT (incluido `frontend-strategy.md` cuando aplica) como actualizado o sin cambios con justificación. Una tarea no está cerrada si el checklist no aparece.

### 7. Auditorías de calidad (cuatro niveles)

- **`/sync`** — detecta drift entre código y docs.
- **`/cuestionar`** — audita calidad de los tests con mutation thinking.
- **`/revisar-bd`** — audita la BD en 4 categorías (modelo, seguridad, rendimiento, calidad de datos).
- **`/revisar-frontend`** *(nuevo en v4.4)* — audita el frontend en 4 categorías:
  - **Stack y mantenibilidad**: TS estricto, `any`, componentes gigantes, dependencias, mezcla de UI libs.
  - **Seguridad**: secretos en cliente, `dangerouslySetInnerHTML` sin sanitizar, validación solo en cliente, headers (CSP, HSTS), CORS, `npm audit`, código IA sin revisión.
  - **Rendimiento (Core Web Vitals)**: LCP, INP, CLS reales (CrUX) si hay despliegue; análisis estático si no.
  - **Accesibilidad WCAG 2.2 AA**: alt, labels, contraste, teclado, landmarks, headings, focus visible, `prefers-reduced-motion`.

Filosofía común: **la IA detecta, el humano decide, el humano repara**. Las auditorías nunca modifican código ni datos automáticamente.

### 8. Implementación disciplinada paso a paso

Cuando un plan acordado implica más de 2 archivos, se activa automáticamente `On(implementation_phase)`. El Agent anuncia los pasos atómicos antes de tocar nada y se detiene tras cada uno esperando "ok, sigue". Esto evita el clásico problema de que la IA encadene 5 archivos de un tirón y ya no haya forma de revisar a tiempo. También incluye un skill auxiliar (`prompt-skill`) con el patrón **RACEO** para que el usuario redacte prompts profesionales con plantillas y ejemplos.

### 9. Adopción controlada de BDD y testing con IA *(nuevo en v4.5)*

v4.5 añade dos capas nuevas al stack de testing con 4 garantías:

1. **BDD opt-in, no por defecto**: `/bdd` solo procede si hay stakeholders no técnicos y disposición a Three Amigos. Sin esas precondiciones, Gherkin es disfraz técnico y el sistema lo bloquea.
2. **Validación humana del `.feature` ANTES de step definitions**: cada `.feature` debe ser aprobado por humanos antes de que la IA genere código de test.
3. **Trazabilidad obligatoria de prompts**: cada test generado por LLM debe tener su `*.prompt.md` asociado y versionado en Git. PR sin auto-merge, code review humano explicito.
4. **Gates de cumplimiento normativo**: para apps de alto riesgo bajo EU AI Act (Annex III: empleo, crédito, educación, salud, justicia), los tests entran en el expediente de calidad junto a prompts y revisiones. AI literacy obligatoria desde febrero 2025.

Skills nuevas: `bdd-skill`, `integration-testing-skill`, `ai-testing-skill`. Workflow nuevo: `/bdd`.

---

## Stack canónico para frontend nuevo (2026)

Salvo desviación documentada en ADR:

| Capa | Recomendación | Razón |
|---|---|---|
| Framework | Next.js 15 / 16 (App Router, RSC, Server Actions) | El que mejor genera la IA; ecosistema sólido |
| Lenguaje | TypeScript estricto (`strict`, `noUncheckedIndexedAccess`) | Sin tipos, la IA degrada |
| Estilado | Tailwind v4 (CSS-first, container queries en core) | Convergencia 2026 |
| Componentes | shadcn/ui (Radix + Tailwind, copy-in) | Accesibles por construcción |
| Forms y validación | react-hook-form + Zod (schemas compartidos front/back) | Validación en cliente Y servidor |
| Linter | Biome v2 (todo-en-uno) | ~35× más rápido que ESLint+Prettier |
| Testing | Vitest + RTL + vitest-axe + Playwright + @axe-core/playwright | Cubre los 3 niveles |
| RUM | Vercel Speed Insights / Cloudflare Web Analytics / Sentry | CWV reales, no de laboratorio |

---

## MCP servers recomendados

| MCP server | Cuándo activarlo | Qué da |
|---|---|---|
| **Figma Dev Mode MCP** | Diseño en Figma con Dev Mode | tokens, jerarquía y component mappings estructurados (no captura) |
| **shadcn MCP** | Stack incluye shadcn/ui | búsqueda e instalación de >6.000 bloques |
| **Chrome DevTools MCP** | Auditoría runtime | DOM/CSS/performance/networking (oficial Google) |
| **Playwright MCP** | Validación visual y E2E | accessibility tree determinista (sin modelo de visión) |
| **Postgres MCP** | BD Postgres | inventario y queries auditables (read-only en producción) |

Plantillas comentadas en `.cursor/mcp.json` listas para activar.

---

## Estructura de archivos

```
/
├── AGENTS.md                          # Resumen para agentes (estándar Linux Foundation)
├── README.md                          # Este archivo
├── LICENSE                            # MIT
├── inicio-chat.txt                    # Manual operativo completo
├── .cursor/
│   ├── mcp.json                       # Plantillas MCP (Postgres + Figma + shadcn
│   │                                  #   + Chrome DevTools + Playwright)
│   ├── rules/
│   │   └── architect-brain.mdc        # Rule global con alwaysApply: true
│   └── skills/                        # 21 skills (13 commands + 8 auxiliares incluido RACEO)
│       ├── inicio/SKILL.md
│       ├── onboarding/SKILL.md
│       ├── regularizar/SKILL.md
│       ├── hoy/SKILL.md
│       ├── nueva/SKILL.md
│       ├── reparar/SKILL.md
│       ├── decidir/SKILL.md
│       ├── sync/SKILL.md
│       ├── cerrar/SKILL.md
│       ├── cuestionar/SKILL.md
│       ├── revisar-bd/SKILL.md
│       ├── revisar-frontend/SKILL.md         # v4.4
│       ├── bdd/SKILL.md                      # nuevo v4.5
│       ├── tests-skill/SKILL.md              # auto-invocable
│       ├── legacy-testing-skill/SKILL.md     # auto-invocable
│       ├── database-skill/SKILL.md           # auto-invocable
│       ├── frontend-skill/SKILL.md           # auto-invocable, v4.4
│       ├── visual-testing-skill/SKILL.md     # auto-invocable, v4.4
│       ├── bdd-skill/SKILL.md                # auto-invocable, nuevo v4.5
│       ├── integration-testing-skill/SKILL.md # auto-invocable, nuevo v4.5
│       ├── ai-testing-skill/SKILL.md         # auto-invocable, nuevo v4.5
│       └── prompt-skill/SKILL.md             # auto-invocable, manual usuario (RACEO)
├── docs/                              # SSOT - plantillas
│   ├── prd.md
│   ├── architecture.md
│   ├── blueprints.md
│   ├── user-stories.md
│   ├── roadmap.md
│   ├── testing-strategy.md
│   ├── database-strategy.md           # Si hay BD
│   ├── frontend-strategy.md           # Si hay frontend (nuevo v4.4)
│   └── decisions-log.md
└── prompts/
    └── architect-brain.md             # Cerebro completo del sistema
```

---

## Reglas de oro

1. **Centralización** — toda documentación vive en `/docs/`. Nada de subcarpetas `docs` internas.
2. **Fuente de verdad** — los docs ganan sobre la memoria auto-generada del IDE.
3. **Sincronización** — ticket terminado = `[x]` en roadmap + tests en verde + checklist visible.
4. **No proliferación** — no crear archivos nuevos para cambios individuales; editar los maestros.
5. **Tests como contrato** — un `.skip` durante más de una semana es un bug del roadmap.
6. **Decisiones trazables** — si no está en `decisions-log.md`, no existe como decisión oficial.
7. **Antidrift** — ninguna tarea se cierra sin el checklist de sincronización documental.
8. **Legacy primero congelar, luego cambiar** — en código legacy, nunca modificar sin haber capturado antes el comportamiento actual.
9. **Cobertura ≠ calidad** — un test que pasa no garantiza detectar bugs.
10. **BD primero auditar, luego cambiar** — en proyectos con BD, ejecutar `/revisar-bd` antes de cambios destructivos.
11. **Frontend primero medir, luego optimizar** — WCAG 2.2 AA y CWV son criterios objetivos. Datos reales (CrUX, axe, lector de pantalla) > intuición.
12. **Implementación paso a paso** — si un plan toca >2 archivos, aplicar `On(implementation_phase)` con paradas obligatorias.
13. **Código IA bajo revisión** — outputs de v0/Lovable/Bolt/Figma Make pasan obligatoriamente por revisión humana en Cursor/Windsurf/Claude Code antes de producción.
14. **BDD sin Three Amigos NO se adopta** *(nuevo en v4.5)* — Gherkin sin conversación previa es disfraz técnico. Si el equipo no está dispuesto, registrar ADR "BDD descartado" y volver más adelante.
15. **Tests IA trazables** *(nuevo en v4.5)* — cada test generado por LLM tiene su `*.prompt.md` asociado, pasa code review humano y NUNCA se mergea por auto-aprobación. Queries accesibles obligatorias.
16. **AI literacy obligatoria** *(nuevo en v4.5)* — EU AI Act exige formación en uso responsable de IA desde febrero 2025. Equipo formado antes de generalizar uso de agentes IA.
17. **GDPR no negociable** *(nuevo en v4.5)* — NO enviar PII a LLMs externos sin DPA válido. Para datos sensibles, LLMs locales (Ollama, vLLM) o proveedores con región europea y DPA firmado.

---

## Sobre este sistema

Construido como adaptación a Cursor del sistema [`junior-doc-gen-tests`](https://github.com/gicupc/junior-doc-gen-tests) (versión original para Windsurf).

Las dos versiones comparten la misma filosofía y los mismos protocolos. La diferencia es la sintaxis de configuración del IDE:

| Concepto | Windsurf | Cursor |
|---|---|---|
| Reglas globales | `.windsurfrules` | `.cursor/rules/*.mdc` con `alwaysApply: true` |
| Slash commands | `.windsurf/workflows/*.md` con frontmatter | `.cursor/skills/*/SKILL.md` con `disable-model-invocation: true` |
| Skills auxiliares | `prompts/skills/*.md` | `.cursor/skills/*/SKILL.md` (auto-invocables) |
| MCP servers | `~/.codeium/windsurf/mcp_config.json` | `.cursor/mcp.json` o `~/.cursor/mcp.json` |

Evolución:

- **v4.1**: testing integrado, ADRs inmutables, antidrift por checklist, slash commands, characterization tests, mutation thinking.
- **v4.2**: base de datos como ciudadano de primera clase — `On(database_setup)`, `On(database_audit)`, `/revisar-bd`, `database-skill`, `database-strategy.md`.
- **v4.3**: `/onboarding` para codebases ajenos, `On(implementation_phase)` con paradas obligatorias entre archivos, patrón Active-Record-friendly para mocks de Prisma, `prompt-skill` con patrón RACEO.
- **v4.4**: **Frontend & MCP Edition**: añade frontend como ciudadano de primera clase. `On(frontend_setup)` automático para proyectos nuevos con stack canónico 2026 (Next.js + TS estricto + Tailwind v4 + shadcn + Biome + Zod). `On(frontend_audit)` y `/revisar-frontend` para proyectos existentes con auditoría de stack/seguridad/CWV/a11y. Skills nuevas `frontend-skill` y `visual-testing-skill`. Doc nuevo `frontend-strategy.md` en SSOT. Plantillas MCP de Figma Dev Mode, shadcn, Chrome DevTools y Playwright añadidas a `.cursor/mcp.json`. Reglas de oro 11, 12 y 13 sobre frontend, MCP servers y código IA bajo revisión.
- **v4.5** *(actual)* — **Testing 2026 Edition (BDD + Integration + AI-Assisted)**: paridad con la versión Windsurf v4.5. Añade tres capas al stack de testing como ciudadanos de primera clase: BDD opt-in con Three Amigos como precondición (skill nueva `bdd-skill`, workflow nuevo `/bdd`), integración con Supertest/MSW/Testcontainers/Pact (skill nueva `integration-testing-skill`), testing asistido por IA con Playwright MCP/CLI, trazabilidad de prompts y gates GDPR + EU AI Act (skill nueva `ai-testing-skill`). Recomendación 2026: Vitest sobre Jest en proyectos nuevos JS/TS. Reglas de oro 14, 15, 16 y 17 sobre BDD sin Three Amigos, tests IA trazables, AI literacy y GDPR.

---

## Licencia

MIT


---

## Soporte para Claude Code (post-v4.4)

Además de Cursor, este sistema funciona también en **Claude Code** (la CLI de Anthropic) gracias al archivo `CLAUDE.md` añadido en la raíz del proyecto. Es un fichero delgado (~6 KB) que apunta a las fuentes de verdad existentes (`.cursor/rules/architect-brain.mdc`, `prompts/architect-brain.md`, las skills), por lo que no duplica contenido y no afecta al comportamiento de Cursor.

Si abres Claude Code en un proyecto que tenga esta estructura instalada, lee `CLAUDE.md` automáticamente y aplica el mismo sistema Architect-Brain v4.5 que aplica Cursor: mismas reglas de oro, mismos protocolos `On(...)`, mismos slash commands.
