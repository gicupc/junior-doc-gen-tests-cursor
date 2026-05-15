# Changelog v4.4 → v4.5 (Cursor Edition — Testing 2026: BDD + Integration + AI-Assisted)

Sesión de actualización: 15 de mayo de 2026

Origen: módulo 11 del curso AI4Devs (Testing 2026 — pruebas de integración + E2E, BDD, testing asistido por IA). Paridad con la versión Windsurf v4.5 (`junior-doc-gen-tests`). El cambio del panorama de testing 2024-2026 es estructural: Playwright MCP/CLI (Microsoft, marzo 2025), playwright-bdd como recomendación para BDD JS/TS con UI, EU AI Act vigente desde agosto 2024 con calendario escalonado (2 feb 2025 AI literacy + prohibiciones, 2 ago 2025 GPAI, 2 ago 2026 alto riesgo Annex III), SpecFlow discontinuado en diciembre 2024 (sucedido por Reqnroll).

---

## Resumen del cambio

El sistema v4.4 ya tenía testing integrado (unitario + visual + a11y) pero quedaban tres capas como puntos ciegos:

1. **Tests de integración** (capa intermedia entre unit y E2E) — sin patrones explícitos para Supertest, MSW o Testcontainers.
2. **BDD / aceptación** — sin protocolo para adoptarlo de forma controlada y sin guía contra los anti-patrones de Gherkin generado por IA.
3. **Testing asistido por IA** — sin reglas sobre Playwright MCP/CLI, trazabilidad de prompts, gates en CI ni cumplimiento normativo (GDPR, EU AI Act).

v4.5 convierte las tres en ciudadanos de primera clase, paralelas a cómo v4.2 hizo lo mismo con base de datos y v4.4 con frontend. La filosofía es idéntica: **detectar señales, configurar entorno automáticamente, registrar ADRs, validar antes de mergear**.

Adicionalmente, v4.5 hace **BDD opt-in con precondiciones explícitas** (stakeholders no técnicos + disposición a Three Amigos). Sin esas precondiciones, el sistema NO adopta BDD: Gherkin sin conversación es disfraz técnico.

---

## Cambios

### 1. Skill nueva: `.cursor/skills/bdd-skill/SKILL.md`

**Estado:** archivo nuevo (auto-invocable, sin `disable-model-invocation`).

Manual completo de Behavior-Driven Development:

- **Principio fundamental**: BDD no es Cucumber; es una conversación que produce ejemplos y se traduce a tests.
- **Cuándo activar / no activar**: criterios explícitos para evitar BDD ceremonial.
- **TDD vs BDD**: tabla operativa para entender que son capas distintas, no alternativas.
- **Sintaxis Gherkin**: referencia rápida + ejemplo canónico con `Background`, `Scenario Outline`, tags.
- **Herramientas 2026** con regla de elección por stack:
  - **playwright-bdd** (recomendado JS/TS con UI, Vitaly Slobodin).
  - **@cucumber/cucumber** con scope (paquete correcto, el sin scope abandonado desde 2021).
  - **@badeball/cypress-cucumber-preprocessor** (sucesor de facto para Cypress).
  - **Reqnroll** (.NET, sucesor de SpecFlow discontinuado dic 2024).
  - **Karate DSL** (JVM + APIs).
  - **Behat** mencionado solo como mantenimiento, no recomendado para nuevos.
- **Setup completo** con playwright-bdd: instalación, `playwright.config.ts`, estructura de carpetas, step definitions con fixtures, scripts `test:bdd*`.
- **Three Amigos + Example Mapping** (técnica de Matt Wynne): tarjetas amarillas/azules/verdes/rojas, reglas operativas (>3 rojas = historia no lista), validación INVEST con IA como segundo revisor.
- **BDD asistido por IA**: prompt RACEO completo para generación de escenarios desde user-story, generación de step definitions con restricciones.
- **7 anti-patrones críticos de Gherkin generado por IA**: escenarios imperativos de UI, demasiado técnicos, múltiples When, falta de Examples, lenguaje inconsistente, escenarios fantasma, pérdida del lenguaje ubicuo.
- **Living documentation**: Cucumber Reports Service, Allure, HTML reporter de Playwright. Pickles y SpecFlow+ LivingDoc discontinuados.
- **MCP server**: confirmado que NO existe MCP oficial de Cucumber/BDD a fecha actual.
- **Integración con Architect-Brain**: `On(bdd_setup)`, `On(task_complete)` con feature, `/cuestionar` sobre step definitions, `/revisar-frontend` lee `.feature` files.
- **Antipatrones**: saltarse Three Amigos, step definitions con regex permisivos, acoplamiento a UI, BDD solo para E2E pesados, tags inflados, idioma mezclado.

### 2. Skill nueva: `.cursor/skills/integration-testing-skill/SKILL.md`

**Estado:** archivo nuevo (auto-invocable).

Cubre la **capa intermedia** de la pirámide (entre unit y E2E):

- **Principio fundamental**: unit tests mienten cómodamente, E2E son caros, integration está en el punto óptimo de la curva coste/valor.
- **Qué es y qué NO es** integración (con ejemplos concretos para distinguir de unit y de E2E).
- **Herramientas 2026 por stack** (Node, PHP, Python, JVM): Supertest, MSW, Testcontainers, Pact, pytest + TestClient, MockMvc, Karate DSL.
- **Patrón 1**: Test de API HTTP con Supertest + BD real (Node + Testcontainers + Prisma) con código completo de `testcontainer-db.ts` helper y test ejemplo de creación de usuario con verificación de BD.
- **Patrón 2**: MSW para mocks de red reutilizables (mismos handlers en tests + dev local + E2E) con código de handlers, server.ts, setup y override puntual por test.
- **Patrón 3**: PHP + PHPUnit + cliente HTTP real con Guzzle, app levantada vía Docker Compose.
- **Patrón 4**: Contract testing con Pact (mención breve, útil solo con >2 microservicios).
- **Cuándo escalar unit → integration → E2E** con tabla operativa y regla del coste (cada nivel arriba multiplica 5x-20x tiempo).
- **Pirámide flexible 2026**: clásica de Mike Cohn vs Testing Trophy de Kent C. Dodds.
- **Convenciones de carpetas, scripts `package.json` con `--pool=forks` recomendado**.
- **Integración con Architect-Brain**.
- **Antipatrones**: mockear BD en tests llamados "integration", reiniciar container entre tests, compartir estado, depender de internet real, `expect(status).toBe(200)` huérfano, suites sin paralelización, no documentar setup.

### 3. Skill nueva: `.cursor/skills/ai-testing-skill/SKILL.md`

**Estado:** archivo nuevo (auto-invocable).

Cubre el flujo IA-asistido aplicado a todas las capas:

- **Principio fundamental**: la IA acelera la parte mecánica si tú entiendes el dominio. Sin entendimiento, produce test theater.
- **IA tradicional (ML clásico) vs IA generativa (LLMs)** con tabla de casos donde gana cada una:
  - ML clásico gana en self-healing selector-level (Healenium), flakiness prediction, test impact analysis, generación determinista en lenguajes tipados (Diffblue Cover).
  - LLMs ganan en generación desde user-story, comprensión semántica de errores, self-healing semántico, debugging conversacional, Gherkin.
  - Regla práctica: combinar ambos enfoques.
- **Playwright MCP** (Microsoft, 22 marzo 2025): qué es, cómo funciona con Snapshot Mode (accessibility tree), instalación en Cursor desde plantilla `.cursor/mcp.json`, capacidades expuestas.
- **Playwright CLI** (2026, recomendado para Cursor con filesystem): comparativa de tokens (~27k CLI vs ~114k MCP), comandos típicos.
- **5 patrones de prompting efectivos**: Given/When/Then, queries accesibles obligatorias, verificación de estabilidad (pasar 3 veces seguidas), Page Object Model en dos pasos, restricciones de dominio para BDD asistido.
- **8 anti-patrones de tests generados por IA**: test theater, snapshot tests gigantes, selectores frágiles que parecen razonables, acoplamiento a implementación, datos irreales, setup duplicado, expects huérfanos, sesgos del training data.
- **Self-healing por categorías** (3 niveles tecnológicos): selector-level (ML clásico, Healenium), visual (Applitools, Percy), semántico (LLMs, Testim, Mabl). Cuándo es peligroso (un test que se auto-cura cuando un botón crítico desaparece). Política recomendada.
- **Plataformas SaaS de testing con IA** con ejes de evaluación + categorías representativas (Qodo, Diffblue Cover, Octomind, Checksum, Reflect, Chromatic, Percy, Applitools, Argos CI, Lost Pixel, Mabl, Functionize, Testim). Confusiones a evitar (Meticulous NO es unit, TestCafe retirado).
- **Trazabilidad de prompts en CI** (obligatorio): patrón `*.prompt.md` junto a `*.spec.ts` con prompt original + decisiones de revisión humana. Gates en CI (lint, ejecución completa sin retries en main, flakiness check, coverage de prompts, code review obligatorio).
- **Riesgos y límites no negociables**:
  - **Coste en tokens**: 10-100 USD/ejecución completa, mitigación con ML clásico primero + CLI sobre MCP + cacheo.
  - **No-determinismo**: `temperature: 0`, seed cuando disponible, NUNCA mergear sin revisión humana.
  - **GDPR**: DPA firmado, LLMs locales (Ollama, vLLM) para datos sensibles, redactar antes de enviar.
  - **EU AI Act**: calendario completo (2 feb 2025 AI literacy + prohibiciones, 2 ago 2025 GPAI, 2 ago 2026 alto riesgo Annex III), multas hasta 35M€ o 7%. Aplicabilidad indirecta a tests si la app es de alto riesgo.
  - **Sesgos**: forzar locale en el prompt.
  - **Casos donde la IA NO funciona bien**: performance, security profundo, contract testing, property-based, sistemas con lógica regulada.
- **Integración con Architect-Brain**.
- **Checklist obligatorio antes de mergear tests generados por IA** (9 items).
- **Antipatrones**: dejar a la IA decidir qué testear, mergear sin code review, mismo prompt genérico, no versionar prompts, pagar por semántico cuando basta selector-level, enviar PII sin DPA, dependencia de un solo proveedor.

### 4. Workflow nuevo: `.cursor/skills/bdd/SKILL.md`

**Estado:** archivo nuevo (slash command con `disable-model-invocation: true`).

Slash command `/bdd`. Adopción guiada con 10 pasos:

1. Verificar aplicabilidad (stakeholders no técnicos + disposición Three Amigos). Si no se cumple, NO procede.
2. Detectar stack y elegir herramienta según regla 2026.
3. Instalación y configuración (playwright-bdd / Cucumber.js).
4. Smoke test BDD verificado y borrado.
5. Primera sesión Three Amigos asistida (opcional pero recomendada) con Example Mapping.
6. Generación del primer `.feature` validado por humano.
7. Validación humana del `.feature` antes de step definitions.
8. Generación de step definitions reutilizando los existentes.
9. Documentación y ADR.
10. Cierre con `On(task_complete)`.

Filosofía: BDD sin Three Amigos es Gherkin disfrazado; el comando bloquea la adopción si las precondiciones no se cumplen.

### 5. Modificado: `.cursor/skills/tests-skill/SKILL.md`

**Estado:** ampliado.

- **Frontmatter `description:`** actualizado con referencias a las 5 skills hermanas y mención de Vitest como recomendación 2026.
- Cabecera con referencias explícitas a `integration-testing-skill`, `visual-testing-skill`, `bdd-skill`, `legacy-testing-skill`, `ai-testing-skill`.
- Sección **"JS/TS + Jest o Vitest"** con recomendación 2026 (Vitest preferido en proyectos NUEVOS con Vite/Next.js 15+/React 19) y patrón con `vi.mock`.
- Sección nueva **"Cuándo escalar a integration o E2E"** con tabla operativa de niveles y dos reglas de decisión rápida.

### 6. Modificado: `.cursor/skills/visual-testing-skill/SKILL.md`

**Estado:** ampliado.

- **Frontmatter `description:`** actualizado con referencias a `integration-testing-skill`, `bdd-skill` y `ai-testing-skill`.
- Cabecera con mención explícita a las skills hermanas y carga de `ai-testing-skill` cuando se usen agentes IA para E2E.
- Sección **Playwright** actualizada con referencia a Playwright MCP (ya disponible como plantilla en `.cursor/mcp.json`) y Playwright CLI (recomendado para Cursor con filesystem).
- Sección nueva **"Alternativas SaaS para visual regression"** con tabla comparativa de 5 herramientas (Chromatic, Percy, Applitools, Argos CI, Lost Pixel) y recomendación por escenario.

### 7. Modificado: `docs/testing-strategy.md`

**Estado:** ampliado con 4 secciones nuevas.

- Sección inicial **"Pirámide de tests del proyecto"** con tabla de los 5 niveles (Unit, Integración, E2E + visual + a11y, BDD opcional, Legacy condicional), referencias a las skills aplicables y nota sobre pirámide flexible 2026 (Mike Cohn vs Testing Trophy).
- Sección nueva **"BDD (opcional)"**: herramienta elegida, estructura de carpetas, política de adopción, tags semánticos.
- Sección nueva **"Testing con IA"**: agentes y herramientas en uso, trazabilidad de prompts, gates en CI.
- Sección nueva **"Cumplimiento normativo (si aplica)"**: GDPR, EU AI Act con calendario y consideraciones para apps de alto riesgo.

### 8. Modificado: `prompts/architect-brain.md`

**Estado:** versión v4.4 → v4.5 + protocolo `On(bdd_setup)` nuevo + ampliaciones.

- Cabecera actualizada a v4.5.
- `On(testing_setup)` ampliado de 10 a 13 pasos:
  - Paso 6 nuevo: instalación de Supertest + Testcontainers + MSW si hay APIs HTTP o BD.
  - Paso 10 nuevo: pregunta opt-in sobre adopción de BDD.
  - Paso 11 nuevo: pregunta sobre uso de IA en testing + recordatorio plantilla Playwright MCP en `.cursor/mcp.json`.
- **`On(bdd_setup)` nuevo** insertado antes de `On(database_setup)` con 11 puntos resumen referenciando `.cursor/skills/bdd/SKILL.md`.
- `On(task_complete)` ampliado con 2 líneas nuevas (Features BDD añadidas, Tests IA con `*.prompt.md`).
- Sección **"Restricciones y Calidad"** ampliada con 4 reglas nuevas (BDD sin Three Amigos, un When por escenario, tests IA con prompt.md, GDPR + EU AI Act).

### 9. Modificado: `.cursor/rules/architect-brain.mdc`

**Estado:** rule global con `alwaysApply: true` ampliada (sin tildes, estilo original).

- **Pregunta 7 nueva en interrupción de seguridad**: criterios de aceptación validables → adoptar BDD.
- **Pregunta 8 nueva**: uso de agentes IA → política de trazabilidad + GDPR + EU AI Act.
- **Fila nueva en tabla de comandos**: `/bdd` con disparadores ("adoptar BDD", "montar Gherkin", "escenarios de aceptacion").
- **Bloque TESTING ampliado** con los 5 niveles de la pirámide referenciando `@.cursor/skills/X/SKILL.md`.
- **Bloque BDD nuevo** completo: precondiciones, reglas de Gherkin, herramientas 2026, validación humana.
- **Bloque TESTING CON IA nuevo** completo: trazabilidad, queries accesibles, Playwright MCP/CLI, IA tradicional vs LLMs, GDPR, EU AI Act, AI literacy.
- **4 reglas de oro nuevas (14-17)**: BDD sin Three Amigos, tests IA trazables, AI literacy obligatoria, GDPR no negociable.

### 10. Modificado: `CLAUDE.md`

**Estado:** v4.4 → v4.5.

- Versión actualizada.
- Lista de slash commands incluye `/bdd`.
- Lista de skills auxiliares incluye `integration-testing-skill`, `bdd-skill`, `ai-testing-skill`.
- Reglas de oro 10 → 13 (añadidas BDD sin Three Amigos, tests IA trazables, AI literacy + GDPR).
- Interrupción de seguridad con preguntas 7 y 8 nuevas.

### 11. Modificado: `AGENTS.md`

**Estado:** v4.4 → v4.5.

- Versión actualizada.
- Lista de 12 → 13 slash commands (añadido `/bdd`).
- Skills auxiliares ampliada con 3 nuevas (`integration-testing-skill`, `bdd-skill`, `ai-testing-skill`).
- Reglas de oro 10 → 13 (añadidas 11, 12, 13 sobre BDD/IA/GDPR).

### 12. Modificado: `README.md`

**Estado:** v4.4 → v4.5.

- Lista de problemas que resuelve: 7 → 8 (añadido "tests IA son test theater").
- Tabla de slash commands: 12 → 13 (añadido `/bdd`).
- "Qué hace el sistema" pasa de 8 a 9 pilares (nuevo: adopción controlada de BDD y testing con IA con 4 garantías).
- Estructura de archivos actualizada: 18 → 21 skills (13 commands + 8 auxiliares).
- Reglas de oro 13 → 17.
- Sección "Sobre este sistema" añade entrada v4.5 con paridad Windsurf.
- Soporte Claude Code actualizado a v4.5.

### 13. Modificado: `inicio-chat.txt`

**Estado:** v4.4 → v4.5 (sin tildes, formato ASCII).

- Cabecera actualizada con párrafo introductorio sobre las 3 capas nuevas.
- Tabla de 12 → 13 slash commands (añadido `/bdd`).
- Skills auxiliares ampliada con `integration-testing-skill`, `bdd-skill`, `ai-testing-skill`.
- Ciclo de vida típico incluye "Quieres adoptar BDD → /bdd".
- Sección detallada nueva para `/bdd` con los 10 pasos del protocolo.
- Reglas de oro 13 → 17 (añadidas 14, 15, 16, 17).
- Estructura de archivos actualizada (16 → 21 skills).
- Protocolo de defensa amplía menciones a "adopta BDD sin Three Amigos" y "mergea tests IA sin *.prompt.md" como desviaciones a corregir.

---

## Migración

Para integrar v4.5 en tu repo local:

1. Reemplaza la carpeta entera `junior-doc-gen-tests-cursor/` con la del repo actualizado.
2. O bien, añade/reemplaza solo estos archivos individualmente:

**Nuevos:**
- `.cursor/skills/bdd-skill/SKILL.md` (**nuevo**)
- `.cursor/skills/integration-testing-skill/SKILL.md` (**nuevo**)
- `.cursor/skills/ai-testing-skill/SKILL.md` (**nuevo**)
- `.cursor/skills/bdd/SKILL.md` (**nuevo**, workflow con `disable-model-invocation: true`)
- `CHANGELOG-v4.5.md` (**nuevo**)

**Modificados:**
- `.cursor/skills/tests-skill/SKILL.md`
- `.cursor/skills/visual-testing-skill/SKILL.md`
- `.cursor/rules/architect-brain.mdc`
- `prompts/architect-brain.md`
- `docs/testing-strategy.md`
- `CLAUDE.md`
- `AGENTS.md`
- `README.md`
- `inicio-chat.txt`

No hay cambios destructivos. Las skills y comandos existentes (`/inicio`, `/regularizar`, `/revisar-bd`, `/revisar-frontend`, etc.) siguen funcionando idénticos. El sistema es retrocompatible.

**Para proyectos en curso bajo v4.4**: los protocolos nuevos NO se disparan retroactivamente. Si quieres aplicarlos a un proyecto ya empezado:

- Para adoptar BDD: invoca `/bdd` manualmente.
- Para añadir tests de integración: carga `@.cursor/skills/integration-testing-skill/SKILL.md` con la IA y aplica los patrones.
- Para empezar a usar agentes IA con trazabilidad: carga `@.cursor/skills/ai-testing-skill/SKILL.md` y aplica el checklist obligatorio antes de mergear.

---

## Verificación post-migración

1. Abre Cursor en un proyecto cualquiera con esta estructura.
2. En el chat de Agent escribe `/` — deberían aparecer **13 comandos** en el desplegable, incluido `/bdd`.
3. Pide al Agent: "Lee `@.cursor/skills/bdd-skill/SKILL.md` y dime qué herramienta BDD recomiendas en 2026 si mi stack es Node.js + TypeScript con UI Next.js."
   - Si responde "playwright-bdd" con razón (paralelización + fixtures + HTML reporter + Trace Viewer) → integrado correctamente.
4. Pide: "Si arranco un proyecto Next.js + Postgres con BMADT, ¿qué pasos sigue `On(testing_setup)` ahora en v4.5?"
   - Debería mencionar entre 13 pasos: instalación de Vitest/Jest, instalación de Supertest + Testcontainers + MSW si hay APIs/BD, pregunta opt-in sobre adopción de BDD, pregunta sobre uso de IA en testing con recordatorio de plantilla Playwright MCP en `.cursor/mcp.json`.
5. Pide: "¿Qué riesgos no negociables debo tener en cuenta al usar agentes IA para generar tests bajo GDPR y EU AI Act?"
   - Debería mencionar: DPA con proveedor, NO enviar PII sin garantías, AI literacy obligatoria desde 2 febrero 2025, calendario EU AI Act, multas hasta 35M€ o 7%, tests en expediente de calidad si la app es de alto riesgo (Annex III).
6. Pide: "Quiero adoptar BDD en el proyecto pero el equipo no quiere hacer Three Amigos. ¿Procedemos?"
   - Debería responder NO y recomendar registrar ADR "BDD descartado". Sin Three Amigos, Gherkin es disfraz técnico.
7. Pide: "Genérame un test E2E para el login con Playwright."
   - Debería usar `getByRole`, `getByLabel` (queries accesibles), NUNCA selectores de clase ni IDs autogenerados, y recordarte la trazabilidad obligatoria (`*.prompt.md`).

---

## Paridad con la versión Windsurf

Esta versión iguala funcionalmente a `junior-doc-gen-tests` v4.5 (Windsurf). Las únicas diferencias son las propias de cada IDE:

| Concepto | Windsurf v4.5 | Cursor v4.5 |
|---|---|---|
| Reglas globales | `.windsurfrules` | `.cursor/rules/architect-brain.mdc` con `alwaysApply: true` |
| Slash commands | `.windsurf/workflows/*.md` con frontmatter | `.cursor/skills/*/SKILL.md` con `disable-model-invocation: true` |
| Skills auxiliares | `prompts/skills/*.md` (carga manual con `@`) | `.cursor/skills/*/SKILL.md` (auto-invocables) |
| MCP servers | `~/.codeium/windsurf/mcp_config.json` (global) | `.cursor/mcp.json` con plantillas comentadas incluida Playwright MCP |
| Resumen para agentes | `AGENTS.md` | `AGENTS.md` |

Todo lo demás (cerebro `architect-brain.md`, docs SSOT, protocolos `On(...)`, comandos disponibles, skills nuevas BDD/Integration/AI, reglas de oro) es idéntico entre ambas versiones.

---

## Lo que NO entró en v4.5 (decisiones explícitas)

- **No se añadió `Q` (Quality) a BMADT como sexta dimensión**. BDD y testing con IA son decisiones de T (Testing) ampliada, no una dimensión paralela. `On(bdd_setup)` y la pregunta sobre IA se disparan dentro de `On(testing_setup)`. Mantener BMADT con 5 dimensiones simplifica la entrevista inicial.

- **No se integró un MCP server específico de Cucumber/BDD**. A día de hoy no existe oficial. La generación de Gherkin con IA se hace desde IDEs generales con prompting (`prompt-skill` patrón RACEO + anti-patrones de `bdd-skill`).

- **No se añadió mutation testing real como obligatorio**. Sigue siendo opt-in vía `/cuestionar` (mutation thinking sin instalar) y, si el usuario quiere ir más allá, ADR para instalar StrykerJS/Infection/PIT/mutmut. La razón: el coste de mantenimiento es alto y no compensa para proyectos pequeños.

- **No se añadió validación automática de cumplimiento EU AI Act**. La regulación es compleja y depende del contexto (alto riesgo o no, sector, datos tratados). `ai-testing-skill` documenta los criterios y obliga a registrar como ADR la decisión, pero NO automatiza la clasificación de riesgo. Eso requiere análisis legal.

- **No se cambió la pirámide por defecto**. Sigue siendo Mike Cohn (muchos unit, pocas E2E). El "Testing Trophy" de Kent C. Dodds se menciona como alternativa válida en `docs/testing-strategy.md`, pero la elección es del proyecto y se documenta como ADR.

- **No se integraron plataformas SaaS específicas como recomendación por defecto** (Octomind, Mabl, Functionize, Qodo, Applitools, etc.). `ai-testing-skill` las cataloga con ejes de decisión, pero NO recomienda una concreta: depende del presupuesto, stack y necesidades del proyecto.

- **No se modificó la versión del archivo `CLAUDE.md` a "thick"**. Sigue siendo delgado (apunta a las skills y al cerebro). El refactor a "AGENTS.md como SSOT principal con `CLAUDE.md` y `.cursor/rules/architect-brain.mdc` como punteros" se reserva para una hipotética v5.0 si demuestra valor real durante el uso de v4.5.
