---
name: bdd-skill
description: Behavior-Driven Development end-to-end. Usar cuando el proyecto adopta BDD para acordar comportamiento del sistema en lenguaje comun entre developer, QA y negocio. Cubre Gherkin, herramientas 2026 (playwright-bdd recomendado para JS/TS con UI, @cucumber/cucumber para APIs sin UI, Reqnroll para .NET, Karate DSL para JVM), setup completo, Three Amigos + Example Mapping de Matt Wynne, BDD asistido por IA con 7 anti-patrones criticos de Gherkin generado por LLMs, living documentation y integracion con Architect-Brain. Complementaria a tests-skill (unit), integration-testing-skill (integracion) y visual-testing-skill (E2E + a11y).
---

# Skill: Behavior-Driven Development (BDD)

Esta skill se activa cuando el proyecto adopta **BDD** (Behavior-Driven Development) como metodología para acordar el comportamiento del sistema en lenguaje común entre developer, QA y negocio. Es complementaria a `tests-skill` (unit) y `visual-testing-skill` (E2E + a11y): BDD vive en la capa de **especificaciones ejecutables** que conectan user-stories con tests reales.

---

## Principio fundamental

> **BDD no es Cucumber. BDD es una conversación que produce ejemplos y se traduce a tests.**

El error más común al adoptar BDD es saltar directamente a escribir `.feature` files sin haber tenido la conversación de descubrimiento (Three Amigos, Example Mapping). El resultado son escenarios técnicos disfrazados de Gherkin que ningún stakeholder lee y que se rompen al primer cambio de UI.

Hacerlo bien significa: **primero conversación → después ejemplos → después Gherkin → después automatización**. Si saltas algún paso, el coste de mantenimiento se multiplica.

---

## Cuándo activar esta skill

- El proyecto tiene stakeholders no técnicos que necesitan validar comportamiento (product owner, cliente, regulador).
- Los criterios de aceptación de las user-stories tienden a ser ambiguos y generan retrabajo.
- Hay tests E2E ya escritos pero su intención es ilegible para quien no es developer.
- Vas a integrar tests de aceptación en el pipeline como gate antes de release.

NO actives BDD si:

- Es un proyecto pequeño de un solo developer sin stakeholders externos. El overhead no compensa.
- El equipo no está dispuesto a hacer las reuniones Three Amigos. Sin la conversación, Gherkin es disfraz.
- Solo hay tests unitarios y de integración, sin UI testeable end-to-end.

---

## TDD vs BDD (resumen operativo)

| Eje | TDD | BDD |
|---|---|---|
| **Foco** | Implementación interna | Comportamiento observable |
| **Ciclo** | Red → Green → Refactor | Discovery → Formulation → Automation |
| **Lenguaje** | Código (assertions) | Gherkin (lenguaje natural estructurado) |
| **Nivel** | Principalmente unitarias | Aceptación + integración + (a veces) unidad |
| **Audiencia** | Developers | Todo el equipo (incluye negocio) |
| **Cuándo gana** | Algoritmos, lógica interna, refactors seguros | Flujos de usuario, criterios de aceptación, regulación |

Los dos son compatibles. Lo profesional en 2026 es: TDD para la lógica interna, BDD para los flujos de aceptación. No son alternativas; son capas distintas.

---

## Sintaxis Gherkin (referencia rápida)

| Palabra clave | Para qué |
|---|---|
| `Feature` | Descripción general de la funcionalidad |
| `Scenario` | Caso de uso concreto |
| `Background` | Pasos comunes a todos los escenarios de la Feature |
| `Given` | Precondición o estado inicial |
| `When` | **UN solo** evento o acción de negocio |
| `Then` | Resultado observable esperado |
| `And` / `But` | Pasos encadenados al anterior |
| `Scenario Outline` + `Examples` | Plantilla parametrizada con tabla de datos |
| `@tag` | Etiqueta para filtrar escenarios (p.ej. `@smoke`, `@regression`) |

Ejemplo canónico:

```gherkin
Feature: Filtrado de candidatos por fase del proceso

  Como manager de un proceso de selección
  Quiero filtrar candidatos por la fase en la que están
  Para revisar rápidamente el estado del proceso

  Background:
    Given existe un proceso de selección "Senior Backend"
    And el proceso tiene fases "Aplicación", "Entrevista técnica", "Oferta"

  Scenario: Filtrar por una fase con candidatos
    Given hay 3 candidatos en la fase "Entrevista técnica"
    When el manager filtra el proceso por la fase "Entrevista técnica"
    Then la lista muestra exactamente 3 candidatos
    And todos los candidatos mostrados están en la fase "Entrevista técnica"

  Scenario: Filtrar por una fase vacía
    Given no hay candidatos en la fase "Oferta"
    When el manager filtra el proceso por la fase "Oferta"
    Then la lista muestra el mensaje "Sin candidatos en esta fase"

  Scenario Outline: Combinación de filtros
    Given hay candidatos en varias fases
    When el manager aplica filtros "<fase>" y "<estado>"
    Then la lista muestra los candidatos esperados "<resultado>"

    Examples:
      | fase                 | estado    | resultado        |
      | Entrevista técnica   | Activo    | 2 candidatos     |
      | Oferta               | Activo    | 1 candidato      |
      | Aplicación           | Rechazado | Sin candidatos   |
```

**Reglas no negociables del Gherkin profesional:**

1. **Un solo `When` por escenario.** Si necesitas dos, son dos escenarios.
2. **Lenguaje del dominio, no de la UI.** "El manager filtra el proceso" no "el usuario hace click en el botón Filtrar".
3. **Datos realistas pero mínimos.** Lo justo para hacer el caso entendible.
4. **`Background` solo para precondiciones comunes a todos los escenarios** de la Feature. No para encadenar setup arbitrario.
5. **`Scenario Outline` cuando varían solo los datos**, no la lógica. Si la lógica cambia, son escenarios separados.

---

## Herramientas en 2026

### Cucumber.js (motor canónico Node.js)

- **Paquete correcto**: `@cucumber/cucumber` (con scope). El paquete `cucumber` sin scope está abandonado desde 2021 — ignorar tutoriales que lo usen.
- Versión actual: 12.x, requiere Node 20+ o 24+.
- **Cuándo usarlo**: tests de servicios sin UI (APIs, microservicios), contract testing, casos donde no necesitas navegador.

### playwright-bdd (recomendado para BDD E2E en JS/TS)

- Plugin que ejecuta `.feature` files con **Playwright como runner**, no con Cucumber.js.
- Mantenedor: Vitaly Slobodin (`vitalets/playwright-bdd`).
- **Ventajas reales sobre Cucumber.js puro**:
  - Paralelización, sharding y workers de Playwright aplicados a escenarios BDD.
  - Acceso a fixtures de Playwright (`test.extend`) en step definitions.
  - HTML reporter de Playwright + Trace Viewer para depurar.
  - Project dependencies (setup/teardown global) sin equivalente en Cucumber.js.
- Requisitos: Node ≥18 y `@playwright/test` ≥1.44.
- **Estado en 2026**: maduro y recomendado para nuevos proyectos JS/TS con UI.

### @badeball/cypress-cucumber-preprocessor (Cypress)

- Preprocessor activo, versión 24.x soporta Cypress 14 y 15.
- **Sucesor de facto**. Ignorar `cypress-cucumber-preprocessor` (sin scope, archivado 2021) y el fork de Klaveness (superado).

### Reqnroll (sucesor de SpecFlow, .NET)

- SpecFlow está **oficialmente discontinuado** desde diciembre 2024. Cualquier proyecto .NET nuevo o en mantenimiento debe migrar a Reqnroll (fork BSD-3 mantenido por la comunidad).

### Karate DSL (BDD para APIs en JVM)

- Framework BDD orientado a API testing donde los pasos `Given/When/Then` se escriben sin step definitions separadas. Versión 1.5.x bajo Karate Labs, requiere Java 17+.
- Útil mencionar si trabajas en proyectos políglotas con stack JVM.

### Behat (PHP)

- Mantenido pero en declive. Mencionar solo si el proyecto ya lo tiene en producción; no recomendar para nuevos.

**Regla de elección en 2026**:

- Stack JS/TS con UI → **playwright-bdd**.
- Stack JS/TS sin UI (APIs puras) → **@cucumber/cucumber**.
- Stack .NET → **Reqnroll**.
- Stack JVM APIs → **Karate DSL**.
- Stack JS con Cypress legacy → **@badeball/cypress-cucumber-preprocessor**.

---

## Setup recomendado con playwright-bdd

Instalación:

```bash
npm install --save-dev playwright-bdd @playwright/test
npx playwright install chromium
```

`playwright.config.ts`:

```ts
import { defineConfig } from '@playwright/test';
import { defineBddConfig } from 'playwright-bdd';

const testDir = defineBddConfig({
  features: 'features/**/*.feature',
  steps: 'features/steps/**/*.ts',
});

export default defineConfig({
  testDir,
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  reporter: [['html'], ['list']],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { browserName: 'chromium' } },
  ],
});
```

Estructura de carpetas:

```
project-root/
├── features/
│   ├── login.feature
│   ├── candidates-filter.feature
│   └── steps/
│       ├── login.steps.ts
│       └── candidates.steps.ts
├── playwright.config.ts
└── package.json
```

Step definitions con fixtures de Playwright:

```ts
// features/steps/login.steps.ts
import { createBdd } from 'playwright-bdd';
import { expect } from '@playwright/test';

const { Given, When, Then } = createBdd();

Given('el usuario está en la página de login', async ({ page }) => {
  await page.goto('/login');
});

When('el usuario introduce credenciales válidas', async ({ page }) => {
  await page.getByLabel('Correo electrónico').fill('alumno@ai4devs.dev');
  await page.getByLabel('Contraseña').fill('Secreta-123!');
  await page.getByRole('button', { name: /iniciar sesión/i }).click();
});

Then('debe ser redirigido al dashboard', async ({ page }) => {
  await expect(page).toHaveURL('/dashboard');
  await expect(page.getByRole('heading', { name: /bienvenida/i })).toBeVisible();
});
```

Scripts en `package.json`:

```json
{
  "scripts": {
    "test:bdd": "bddgen && playwright test",
    "test:bdd:ui": "bddgen && playwright test --ui",
    "test:bdd:smoke": "bddgen && playwright test --grep @smoke"
  }
}
```

El comando `bddgen` regenera los specs de Playwright a partir de los `.feature` files antes de cada ejecución.

---

## Three Amigos y Example Mapping (la fase de descubrimiento)

La parte más infravalorada de BDD: las **conversaciones antes del código**.

### Three Amigos

Tres roles en una mesa antes de codificar:

1. **Business / Product** — qué quiere el negocio.
2. **Developer** — cómo se podría implementar.
3. **QA / Tester** — qué podría salir mal.

Si la IA participa, lo hace como **cuarto participante asistente**, no como sustituto de ninguno de los tres. Su rol útil: proponer ejemplos adicionales y casos borde durante la sesión.

### Example Mapping (técnica de Matt Wynne)

Estructura una sesión de descubrimiento con cuatro tipos de tarjetas de colores:

- 🟡 **Amarilla** — la historia de usuario (una sola).
- 🔵 **Azules** — reglas de negocio extraídas de la historia.
- 🟢 **Verdes** — ejemplos concretos por cada regla.
- 🔴 **Rojas** — preguntas sin resolver durante la sesión.

**Reglas operativas:**

- Si tienes **muchas tarjetas rojas**, la historia NO está lista para desarrollarse. Vuelta a refinamiento.
- Si tienes **muchas tarjetas azules sin verdes**, las reglas son abstractas. Hace falta más conversación con ejemplos concretos.
- Si tienes **muchas verdes sin azules**, los ejemplos no se traducen a reglas. Hay riesgo de overfitting.

El output de Example Mapping son las **tarjetas verdes** convertidas en escenarios Gherkin. Cada tarjeta verde es un escenario candidato.

### Validación INVEST (segundo revisor)

La IA puede actuar como revisor para validar si la user-story cumple los criterios **INVEST**:

- **I**ndependent — independiente de otras historias.
- **N**egotiable — no es un contrato cerrado.
- **V**aluable — aporta valor al usuario.
- **E**stimable — el equipo puede estimar el coste.
- **S**mall — cabe en un sprint.
- **T**estable — los criterios de aceptación son verificables.

NO usar a la IA como **autor único** de la user-story. Como segundo revisor sí.

---

## BDD asistido por IA

La IA generativa cambió cómo se escribe Gherkin. Bien usada, acelera el flujo. Mal usada, produce escenarios frágiles y desconectados del dominio.

### Generación de escenarios desde user-stories

Prompt efectivo (siguiendo el patrón RACEO de `prompt-skill`):

```
# Role
Eres un QA senior con experiencia en BDD. Tu trabajo es traducir
user-stories a escenarios Gherkin que un manager pueda leer y validar.

# Objective
Producir escenarios .feature en Gherkin para la user-story descrita,
listos para revisión con el equipo (Three Amigos).

# Context
User-story:
  "Como manager de un proceso de selección, quiero filtrar candidatos
   por fase del proceso para revisar rápidamente el estado de cada uno."

Reglas de negocio acordadas:
  - Las fases del proceso son: Aplicación, Entrevista técnica, Entrevista
    cultural, Oferta, Contratado, Rechazado.
  - Un candidato está en UNA sola fase a la vez.
  - El filtro puede combinarse con estado (Activo/Rechazado).

Lenguaje del dominio:
  - "Manager", "candidato", "proceso", "fase". NO "usuario", "elemento",
    "etapa".

# Constraints
- Un único When por escenario.
- Lenguaje del dominio, NO de la UI ("filtra el proceso", NO "hace click
  en el botón Filtrar").
- Cubre: (1) caso feliz, (2) sin candidatos en la fase, (3) filtro inválido,
  (4) combinación de filtros.
- Usa Scenario Outline si los casos comparten estructura.
- NO inventes precondiciones que no estén en las reglas dadas.

# Expected Output
Un archivo .feature en Gherkin con Background, Scenarios y Scenario Outline
cuando aplique. Sin step definitions: solo el .feature.
```

### Generación de step definitions desde .feature

Cuando el `.feature` ya está validado por el equipo, la IA puede generar los step definitions:

```
Lee features/candidates-filter.feature y genera los step definitions en
TypeScript usando playwright-bdd. Usa queries accesibles (getByRole,
getByLabel, getByTestId). Si el step ya existe en features/steps/, NO lo
dupliques: reutilízalo.

Ejecuta los tests al terminar y confirma verde antes de dar la tarea por
acabada.
```

### Anti-patrones de Gherkin generado por IA (lista crítica)

Los LLMs producen estos errores de forma sistemática. El alumno/developer debe **detectarlos y corregirlos**:

1. **Escenarios imperativos paso a paso de UI**:
   - ❌ `When I click the submit button`
   - ✅ `When el cliente realiza el pedido`

2. **Demasiado técnicos**: referencias a IDs del DOM, JSON payloads o nombres de columnas de BD.
   - ❌ `Then the API returns 201 with {"id": 1, "status": "active"}`
   - ✅ `Then el sistema confirma la creación del pedido`

3. **Múltiples When/Then por escenario**: un escenario describe UN solo evento de negocio.
   - ❌ `When el usuario hace login And cuando navega al perfil And edita su email`
   - ✅ Tres escenarios separados, o un solo evento "actualiza su email" como When.

4. **Falta de Examples o sobreespecificación de datos**:
   - Si los casos comparten estructura, usar `Scenario Outline` con `Examples`. No copiar-pegar el escenario 5 veces cambiando una palabra.

5. **Lenguaje inconsistente**: la misma acción descrita de tres formas distintas en features distintas ("manager filtra", "user busca", "admin lista").
   - Mantener el **lenguaje ubicuo** del dominio.

6. **Escenarios fantasma**: el LLM inventa precondiciones no acordadas con negocio porque "rellenan bien".
   - Marcar el prompt explícitamente: "NO inventes precondiciones".

7. **Pérdida del lenguaje ubicuo**: el LLM sustituye términos del dominio por sinónimos genéricos.
   - ❌ "usuario" en vez de "candidato".
   - ❌ "elemento" en vez de "vacante".
   - Marcar el prompt: "Usa exactamente estos términos: candidato, manager, fase, proceso".

---

## Living documentation

Una vez los `.feature` files están bajo control de versiones y los tests se ejecutan en CI, el `.feature` mismo es **documentación viva**: si pasa, refleja lo que el sistema hace ahora.

### Reportes recomendados en 2026

- **Cucumber Reports Service** (`messages.cucumber.io`): publica HTML reports automáticamente con `publish: true`.
- **Allure Report**: estándar de facto para reportes ricos. Integrado con Cucumber.js, playwright-bdd y Cypress.
- **HTML reporter de Playwright**: cuando uses playwright-bdd, ya lo tienes gratis.

### Discontinuados (no usar)

- **Pickles** — discontinuado.
- **SpecFlow+ LivingDoc** — discontinuado junto con SpecFlow.

### MCP server específico de Cucumber/BDD

A día de hoy **NO existe** un MCP server oficial publicado por el proyecto Cucumber. La generación de Gherkin con IA se hace desde IDEs generales (Cursor, Claude Code, Windsurf) con prompting, no desde MCPs dedicados. Si en el futuro aparece uno oficial, esta skill debe actualizarse.

---

## Integración con el harness Architect-Brain

- **`On(bdd_setup)`** (definido en `prompts/architect-brain.md`) se ejecuta cuando el proyecto adopta BDD durante `/inicio` o `/regularizar`. Instala `playwright-bdd` (o `@cucumber/cucumber` si no hay UI), crea estructura `features/` y `features/steps/`, añade scripts a `package.json`, registra ADR, añade tickets al `roadmap.md`.
- **`On(task_complete)`** debe verificar que las nuevas features con criterios de aceptación BDD tienen su `.feature` correspondiente. Si falta, no cerrar la tarea.
- **`/cuestionar`** (mutation thinking) aplica también a step definitions: ¿el step "el usuario está logueado" cubre todos los casos relevantes o solo el caso feliz?
- **`/revisar-frontend`** debe leer los `.feature` files si existen, ya que documentan los flujos críticos que deberían tener tests E2E.

---

## Antipatrones a evitar

- **Saltarse Three Amigos**. Sin la conversación, Gherkin es disfraz técnico.
- **Reusar step definitions con regex demasiado permisivos** que esconden cambios de comportamiento. Mejor cinco steps específicos que uno genérico con cinco regex groups.
- **Acoplar el `.feature` a la UI**: si el escenario rompe cuando se renombra un botón, el escenario no estaba describiendo comportamiento.
- **Usar BDD solo para tests E2E pesados**. BDD también vale para tests de aceptación a nivel de API.
- **No actualizar el `.feature` cuando cambia el dominio**. El `.feature` desactualizado es peor que no tener feature.
- **Tags inflados**: 15 tags por escenario hace que nadie sepa qué corre cuándo. Máximo 2-3 tags semánticos (`@smoke`, `@regression`, `@critical`).
- **`.feature` files en idioma mezclado**: o todo en español o todo en inglés. Mezclar es deuda técnica futura.

---

## Regla de oro final

> **BDD es una conversación que produce ejemplos. Los `.feature` son el resultado, no el origen.**

Si la IA escribe el `.feature` antes de que haya conversación, has saltado el paso que justifica BDD. Mejor no hacer BDD que hacer BDD ceremonial.
