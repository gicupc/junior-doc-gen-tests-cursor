---
name: bdd
description: Adoptar BDD (Behavior-Driven Development) en el proyecto - verifica aplicabilidad (stakeholders no tecnicos + disposicion a Three Amigos), instala herramienta segun stack 2026 (playwright-bdd para JS/TS con UI, @cucumber/cucumber para APIs sin UI, Reqnroll para .NET, Karate DSL para JVM), crea estructura features/, ofrece primera sesion Three Amigos asistida con Example Mapping, genera primer .feature validado por humano antes de step definitions, registra ADR y tickets.
disable-model-invocation: true
---

# /bdd — Adoptar BDD en el proyecto

Este comando guia la adopcion de BDD (Behavior-Driven Development) en un proyecto: setup tecnico de la herramienta (playwright-bdd o @cucumber/cucumber segun stack), creacion de la estructura `features/`, primera sesion Three Amigos asistida, generacion del primer `.feature` validado, y registro de la decision como ADR.

NO ejecuta BDD a ciegas: pregunta al usuario si BDD aplica a su proyecto y, si la respuesta es no, propone alternativas mas ligeras.

---

## Cuando usar este comando

- El proyecto tiene stakeholders no tecnicos (product owner, cliente, regulador) que necesitan validar criterios de aceptacion.
- Las user-stories actuales tienden a ser ambiguas y generan retrabajo.
- Hay tests E2E ya escritos pero su intencion es ilegible para quien no es developer.
- Vas a integrar tests de aceptacion en el pipeline como gate antes de release.

Cuando NO usar:

- Proyecto pequeno de un solo developer sin stakeholders externos. El overhead de BDD no compensa.
- El equipo NO esta dispuesto a hacer las conversaciones Three Amigos. Sin la conversacion, Gherkin es disfraz.
- Solo hay tests unitarios y de integracion, sin flujos completos testeables E2E.

---

## Pasos del protocolo

### Paso 1 - Verificar aplicabilidad

Lee `@.cursor/skills/bdd-skill/SKILL.md` y pregunta al usuario:

```
Antes de adoptar BDD, necesito confirmar que aplica a tu caso:

1. ¿Hay stakeholders no tecnicos que validan criterios de aceptacion?
2. ¿Estais dispuestos a hacer Three Amigos (developer + QA + business)
   antes de codificar las features importantes?
3. ¿El proyecto tiene UI o APIs que se puedan testear end-to-end?

Si la respuesta a alguna de las primeras dos es NO, BDD probablemente
NO aporta valor real. Mejor seguir con tests unitarios + E2E sin la
capa Gherkin.

¿Procedemos con el setup de BDD?
```

Si el usuario responde NO o duda, **NO procedas**. Cierra el comando con una recomendacion: "Mantener tests-skill y visual-testing-skill como cobertura. Volver a /bdd cuando aparezcan los stakeholders no tecnicos."

Si el usuario confirma, sigue.

### Paso 2 - Detectar stack y elegir herramienta

Lee `package.json` / `composer.json` / `pyproject.toml` para detectar stack.

Aplica la **regla de eleccion en 2026** de `bdd-skill`:

| Stack detectado | Herramienta a instalar |
|---|---|
| Node.js + TS con frontend testeable (Playwright presente o instalable) | **playwright-bdd** |
| Node.js + TS solo APIs (sin UI) | **@cucumber/cucumber** |
| Node.js + TS con Cypress legacy | **@badeball/cypress-cucumber-preprocessor** |
| .NET | **Reqnroll** |
| JVM + APIs | **Karate DSL** |
| PHP (proyecto legacy con Behat) | **Behat** (mantener, no recomendar para nuevos) |

Confirma con el usuario antes de instalar:

```
He detectado stack [X]. Recomiendo [Y] como herramienta BDD por estas razones:
[razones]

Alternativas validas: [lista].

¿Procedo con [Y]?
```

### Paso 3 - Instalacion y configuracion

Una vez confirmada la herramienta, instala y configura siguiendo `bdd-skill`:

**Si playwright-bdd:**

```bash
npm install --save-dev playwright-bdd @playwright/test
npx playwright install chromium
```

Crea/actualiza `playwright.config.ts` con `defineBddConfig`. Si ya existe un `playwright.config.ts` del setup E2E previo, **NO lo sobreescribas**: anade la configuracion BDD coexistiendo (ver `bdd-skill` para el patron).

**Si @cucumber/cucumber:**

```bash
npm install --save-dev @cucumber/cucumber @types/node typescript ts-node
```

Crea `cucumber.js` (o `cucumber.json`) con la configuracion estandar.

**En ambos casos**, crea la estructura de carpetas:

```
features/
├── .gitkeep
└── steps/
    └── .gitkeep
```

Y anade scripts al `package.json`:

```json
{
  "scripts": {
    "test:bdd": "bddgen && playwright test",
    "test:bdd:ui": "bddgen && playwright test --ui",
    "test:bdd:smoke": "bddgen && playwright test --grep @smoke"
  }
}
```

(adaptar segun la herramienta elegida).

### Paso 4 - Smoke test BDD

Crea un `.feature` minimo y su step definition para verificar que el pipeline funciona:

```gherkin
# features/smoke.feature
@smoke
Feature: BDD smoke test

  Scenario: BDD pipeline is configured correctly
    Given the BDD pipeline is installed
    When I run the smoke test
    Then the test passes
```

```ts
// features/steps/smoke.steps.ts
import { createBdd } from 'playwright-bdd';
import { expect } from '@playwright/test';

const { Given, When, Then } = createBdd();

let pipelineReady = false;
let testRan = false;

Given('the BDD pipeline is installed', () => {
  pipelineReady = true;
});

When('I run the smoke test', () => {
  if (pipelineReady) testRan = true;
});

Then('the test passes', () => {
  expect(testRan).toBe(true);
});
```

Ejecuta `npm run test:bdd:smoke`, confirma verde, y **BORRA** el smoke test. Solo el pipeline queda configurado.

### Paso 5 - Primera sesion Three Amigos asistida (opcional pero recomendada)

Pregunta al usuario:

```
¿Quieres que ejecutemos ahora una primera sesion Three Amigos asistida sobre
la primera user-story que quieras automatizar con BDD?

Si si: dame la user-story y los nombres de los participantes (puedo simular
el rol de QA si no esta presente). Yo guio la sesion paso a paso siguiendo
Example Mapping (Reglas - Ejemplos - Preguntas).

Si no: dejo el setup listo y volveras cuando quieras escribir tu primer
.feature real.
```

Si el usuario procede, ejecuta el sub-protocolo de Example Mapping (ver `bdd-skill` seccion "Three Amigos y Example Mapping"):

1. Pregunta la user-story (1 sola).
2. Extrae **reglas de negocio** candidatas y las lista como tarjetas azules.
3. Propon **ejemplos concretos** por regla (tarjetas verdes).
4. Marca explicitamente las **preguntas sin resolver** (tarjetas rojas).
5. Si hay mas de 3 rojas, alerta: "La historia no esta lista para desarrollarse. Vuelta a refinamiento."
6. Si pasa el filtro, **convierte las verdes en escenarios Gherkin** candidatos.
7. **Pide validacion humana** del `.feature` antes de salvarlo en disco.

### Paso 6 - Generacion del primer .feature

Si el usuario aprueba el `.feature` del paso 5 (o si decidio saltarse Three Amigos y dar la user-story directamente), genera el archivo en `features/`.

**Reglas no negociables al generar**:

- Un solo `When` por escenario.
- Lenguaje del dominio, NO de la UI.
- Datos realistas pero minimos.
- Sin inventar precondiciones que no esten en las reglas dadas.
- Usar `Scenario Outline` si los casos comparten estructura.
- Usar exactamente el lenguaje ubicuo del proyecto (preguntar al usuario los terminos si no los conoces).

Tras generar, **NO escribas los step definitions todavia**. El siguiente paso es la validacion humana del `.feature`.

### Paso 7 - Validacion del .feature

Muestra el `.feature` generado al usuario y pregunta:

```
Aqui tienes el .feature propuesto para [user-story]. Antes de generar los
step definitions, valida:

1. ¿Cada escenario es legible por un manager/cliente?
2. ¿El lenguaje es del dominio (no de la UI)?
3. ¿Hay un solo When por escenario?
4. ¿Los datos son realistas?
5. ¿He inventado alguna precondicion no acordada?

¿Apruebas el .feature o quieres cambios?
```

Si el usuario pide cambios, itera. **NO procedas a generar step definitions hasta que el `.feature` este aprobado.**

### Paso 8 - Generacion de step definitions

Solo tras la aprobacion del `.feature`:

1. Genera los step definitions en `features/steps/` siguiendo el patron de `bdd-skill`.
2. Usa queries accesibles (`getByRole`, `getByLabel`, `getByTestId`) si es playwright-bdd.
3. Si un step ya existe en `features/steps/` por features previas, **reutilizalo**, no lo dupliques.
4. Ejecuta `npm run test:bdd` y muestra el output completo.
5. Si hay rojos, NO los marques como aprobados: depura primero.

### Paso 9 - Documentacion y ADR

Actualiza el SSOT:

1. **`docs/testing-strategy.md`**:
   - Anade seccion "BDD" con la herramienta elegida, version, ubicacion de features y scripts.
   - Anade nivel "BDD / Aceptacion" a la piramide de tests del proyecto.
2. **`docs/decisions-log.md`**:
   - Registra ADR nueva: "Adopcion de BDD con [herramienta]" con razon, alternativas descartadas, alcance.
3. **`docs/roadmap.md`**:
   - `[x] Setup BDD environment - [herramienta] + estructura features/ + smoke test verificado`.
   - `[ ] Primer .feature productivo: [titulo de la user-story]` (si se genero en paso 5-8).

### Paso 10 - Cierre

Ejecuta `On(task_complete)` con el checklist de sincronizacion documental visible.

---

## Cuando reentrar a /bdd

- Cuando aparece una nueva user-story compleja que merece Three Amigos antes de codificar.
- Cuando un `.feature` existente se queda desactualizado tras un cambio de dominio.
- Cuando quieres anadir tags semanticos (`@smoke`, `@regression`, `@critical`) a la suite y consolidar gates en CI.

Para iteraciones simples (anadir 1 escenario a una Feature existente), basta con editar el `.feature` y los steps. NO hace falta `/bdd` cada vez.

---

## Regla de oro

> **BDD sin Three Amigos es Gherkin disfrazado. Three Amigos sin BDD es una reunion mas.**

El valor solo aparece cuando las dos partes (conversacion humana + automatizacion) van juntas. Este comando intenta que ambas existan; si no es posible, mejor NO adoptar BDD.
