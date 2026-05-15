---
name: ai-testing-skill
description: Testing asistido por IA aplicado a todas las capas (unit, integration, E2E, BDD). Usar cuando se introduce IA en el flujo de testing (Cursor, Claude Code, Copilot, Playwright MCP/CLI). Cubre IA tradicional (ML clasico) vs LLMs con tabla de cuando gana cada una, Playwright MCP (Microsoft mar 2025) vs Playwright CLI (4x menos tokens), 5 patrones de prompting efectivos, 8 anti-patrones de tests generados por IA, self-healing por categorias, plataformas SaaS, trazabilidad de prompts en CI con *.prompt.md, gates obligatorios, riesgos (coste en tokens 10-100 USD/run, no-determinismo, GDPR, EU AI Act con calendario completo 2 feb 2025 + 2 ago 2025 + 2 ago 2026).
---

# Skill: Testing Asistido por IA

Esta skill cubre el **flujo IA-asistido aplicado al testing** en 2026: generación de tests con LLMs, automatización de navegador con agentes (Playwright MCP/CLI), selección de herramientas comerciales por categoría (visual regression, self-healing, observabilidad), y los riesgos no negociables (coste en tokens, no-determinismo, GDPR, EU AI Act).

Es transversal a `tests-skill`, `integration-testing-skill`, `bdd-skill` y `visual-testing-skill`: aplica a todas ellas cuando se introduce IA en el flujo.

---

## Principio fundamental

> **La IA no escribe tests por ti. La IA acelera la parte mecánica del testing si tú entiendes el dominio. Sin entendimiento del dominio, la IA produce test theater: cobertura alta, capacidad de detectar regresiones baja.**

El cambio estructural 2024-2026 es real: agentes con Playwright MCP/CLI pueden navegar la app, generar tests E2E estables y ajustarlos hasta que pasen. Pero el rol del developer no desaparece, se desplaza: de **tecleador de tests** a **arquitecto y revisor de calidad**.

---

## Cuándo activar esta skill

- Vas a usar Cursor, Claude Code, Copilot u otro agente para generar tests E2E con Playwright.
- Quieres montar Playwright MCP en tu IDE para que el agente navegue la app real.
- Estás evaluando una plataforma SaaS de testing con IA (Qodo, Octomind, Mabl, Functionize, Applitools, etc.).
- Necesitas establecer **gates de calidad** sobre los tests generados por IA antes de que entren a CI.
- El proyecto debe cumplir con GDPR / EU AI Act y vas a enviar logs o capturas a un LLM externo.

NO activar si:

- El proyecto es trivial y los unit tests con Jest/Vitest bastan: no hace falta MCP ni agentes.
- No estás dispuesto a hacer code review estricta sobre los tests generados: mejor no usar IA, generaría test theater.

---

## IA tradicional (ML) vs IA generativa (LLMs) — distinción crítica

Una de las distinciones más útiles en testing 2026. Confundirlas lleva a elegir la herramienta equivocada.

### IA tradicional (Machine Learning clásico)

Modelos entrenados para tareas específicas (clasificación, regresión, anomaly detection, reinforcement learning).

**Características:**

- Determinismo alto: salidas reproducibles dado el mismo input.
- Coste por inferencia bajo y predecible.
- Trazabilidad mejor: matrices de confusión, métricas estables.
- Latencia en milisegundos.

**Casos donde SUPERA a los LLMs en testing:**

| Caso | Por qué gana ML clásico | Ejemplo de herramienta |
|---|---|---|
| Self-healing de selectores | Búsqueda determinista (LCS + gradient boosting) | **Healenium** |
| Predicción de flakiness | Histórico tabular, no requiere semántica | Modelos custom sobre histórico de CI |
| Test impact analysis | Grafo de dependencias + clasificador | Microsoft TIA, Launchable |
| Generación de unit tests deterministas en lenguajes tipados | RL sobre bytecode, garantiza compilación | **Diffblue Cover** (Java) |
| Anomaly detection en métricas (latencia, error rate) | Series temporales | Datadog ML, Sentry |

### IA generativa (LLMs)

Modelos de lenguaje de propósito general entrenados con texto y código.

**Características:**

- No-determinismo por defecto (depende de temperatura, seed, proveedor).
- Coste alto y variable (medido en tokens).
- Trazabilidad peor: requiere logging explícito de prompts y outputs.
- Latencia en segundos.

**Casos donde SUPERAN a ML clásico:**

| Caso | Por qué gana LLM | Ejemplo de herramienta |
|---|---|---|
| Generación de tests desde user-story o PRD | Comprensión semántica del lenguaje natural | Cursor, Claude Code, Copilot, Qodo Gen |
| Comprensión de errores ("el endpoint cambió de /v1 a /v2") | Razonamiento sobre logs y diffs | Cualquier IDE-LLM |
| Self-healing semántico ("Add to cart" → "Buy now") | Equivalencia semántica entre strings | Applitools AI, Testim |
| Debugging conversacional | Diálogo iterativo | Claude Code, Cursor Chat |
| Generación de Gherkin desde requisitos | Lenguaje natural estructurado | Cualquier IDE-LLM con prompting RACEO |

### Regla práctica de combinación

> **Los equipos más efectivos en 2026 combinan ambos enfoques: ML clásico para tareas estables de alto volumen (selector healing, flakiness prediction, anomaly detection); LLMs para tareas que requieren comprensión semántica del dominio (generación, debugging, Gherkin).**

**Nunca usar LLM para algo que un modelo determinista resuelve mejor y más barato.** Healing de selectores con LLM = caro, lento y no-determinista; con Healenium = barato, rápido y reproducible.

---

## Playwright MCP — el cambio estructural más importante

**Qué es**: servidor MCP (Model Context Protocol) oficial de Microsoft lanzado el 22 de marzo de 2025 que expone Playwright como herramientas que cualquier LLM puede invocar. Es el MCP server #1 del ecosistema en rankings públicos.

**Cómo funciona**: en lugar de basarse en screenshots y modelos de visión, el modo por defecto ("Snapshot Mode") usa el **árbol de accesibilidad** del navegador, lo que hace las acciones más rápidas, deterministas y baratas en tokens.

### Instalación en Cursor

Plantilla ya disponible en `.cursor/mcp.json` (bloque `_playwright`). Para activarla, renombra a `playwright` (sin el `_`) y reinicia Cursor.

Si prefieres ejecutarla manualmente:

```bash
npx @playwright/mcp@latest
```

### Capacidades expuestas

- Navegación (`goto`, `back`, `forward`, `reload`).
- Snapshots de accesibilidad.
- Acciones: `click`, `type`, `press`, `select`, `hover`.
- `screenshot` (cuando hace falta visión).
- Mocking de red (`intercept`, `mock`, `modify`, `block`).
- Gestión de cookies, `localStorage`, `sessionStorage`.
- Sesiones persistentes entre tests.
- Capabilities opcionales: `vision`, `pdf`, `devtools`.

### Flujo típico con Playwright MCP

1. El developer describe el escenario en lenguaje natural.
2. El Agent abre el navegador vía Playwright MCP.
3. El Agent explora la UI real, identifica elementos por rol/etiqueta.
4. El Agent escribe el spec en TypeScript.
5. El Agent lo ejecuta y lo ajusta hasta que pasa de forma estable (3 veces sin flake).
6. El developer revisa y aprueba.

---

## Playwright CLI — la opción para agentes con filesystem

A principios de 2026 Microsoft lanzó `@playwright/cli` como complemento orientado específicamente a **agentes con acceso al sistema de ficheros** (Claude Code, Cursor con filesystem, Copilot Coding Agent).

### Por qué importa

| Métrica | Playwright MCP | Playwright CLI |
|---|---|---|
| Tokens por tarea típica | ~114k | ~27k |
| Persistencia entre llamadas | Sesión del MCP | Ficheros YAML en disco |
| Cliente ideal | Chat puro (Claude Desktop) | Agente con filesystem (Claude Code, Cursor) |

**Recomendación oficial de Microsoft (2026)**: usar CLI cuando el agente tiene filesystem, MCP cuando el cliente es chat puro. En Cursor, donde el Agent tiene filesystem, el CLI suele ser preferible si gestionas un proyecto grande con muchos tests.

### Comandos típicos

```bash
playwright-cli open https://app.local
playwright-cli click "button:has-text('Iniciar sesión')"
playwright-cli type "input[name=email]" "user@test.com"
playwright-cli screenshot login.png
playwright-cli state-save auth.json
```

El agente guarda snapshots, screenshots y estado en disco como YAML, y luego lee solo lo necesario en cada paso.

---

## Patrones de prompting efectivos para tests E2E

Lo que separa un test generado por IA estable de uno frágil suele ser el prompt. Patrones que funcionan (siguiendo `prompt-skill` RACEO):

### Patrón 1: Given/When/Then antes que pseudocódigo

```
Genera un test Playwright. Given que estoy en /login, When introduzco
credenciales válidas y hago click en "Iniciar sesión", Then debo ser
redirigido a /dashboard y ver un saludo de bienvenida.
```

Mejor que: *"Haz un test que loguea al usuario"*.

### Patrón 2: Queries accesibles obligatorias

```
Usa SOLO getByRole, getByLabel y getByTestId. NO uses selectores de clase,
NO uses IDs auto-generados (#mui-12, .css-1abc2de), NO uses XPath. Si no
encuentras un elemento por rol o label, refactoriza el componente para
añadir aria-label o role, NO uses selectores frágiles.
```

### Patrón 3: Verificación de estabilidad

```
Genera el test, ejecútalo 3 veces seguidas. Si en alguna ejecución falla
o requiere retries, NO lo des por bueno: ajusta hasta que pase 3 veces
limpio. Reporta el output de cada ejecución.
```

### Patrón 4: Page Object Model en dos pasos

```
Paso 1: explora la app con Playwright MCP y propón un Page Object Model
para los flujos de login y dashboard. NO escribas tests todavía: solo el
POM. Pasa por aprobación humana.

Paso 2 (cuando apruebe el POM): reescribe los tests existentes usando
ese POM. Verifica que pasan tras el refactor.
```

### Patrón 5: Restricciones de dominio para BDD asistido

```
Lee features/login.feature y genera los step definitions en TypeScript
usando playwright-bdd. Usa queries accesibles. Si el step ya existe en
features/steps/, NO lo dupliques: reutilízalo.

NO inventes precondiciones que no estén en el .feature.
NO añadas escenarios nuevos: solo step definitions para los existentes.
```

---

## Anti-patrones de tests generados por IA

Los LLMs producen estos errores sistemáticamente. Detectarlos y corregirlos es parte del **code review obligatorio** de tests generados por IA.

### 1. Test theater (alto coverage, baja capacidad de detectar regresiones)

❌ El LLM genera un test que llama a `service.create()` y asserta `expect(service.create).toHaveBeenCalled()`.

✅ El test debe verificar el **resultado observable** (el usuario está en BD, el response tiene la forma correcta), no el call al mock.

### 2. Snapshot tests gigantes

❌ El LLM hace `expect(rendered).toMatchSnapshot()` de un componente de 200 nodos. Cualquier cambio rompe el test sin que aporte información.

✅ Assertions específicas: `toHaveTextContent`, `toHaveAttribute`, `getByRole`.

### 3. Selectores frágiles que parecen razonables

❌ `page.locator('.btn-primary-2xl')`, `page.locator('#mui-12345')`.

✅ `page.getByRole('button', { name: /save/i })`.

### 4. Tests acoplados a la implementación

❌ El LLM verifica que el código llama a métodos internos en un orden específico.

✅ El test verifica el contrato observable del módulo, no su flujo interno.

### 5. Datos irreales o vacíos

❌ `user.create({ firstName: 'aaa', email: 'a@a.a' })`.

✅ `user.create({ firstName: 'Ana', email: 'ana@empresa.com' })`. Datos realistas pero mínimos.

### 6. Multiples tests con setup duplicado

El LLM no extrae el setup común a `beforeEach`/`beforeAll`. Resultado: 20 tests con 15 líneas de setup copiado. Mantenimiento imposible.

### 7. `expect` huérfanos sin mensaje

❌ `expect(result).toBeTruthy()` (¿qué se está verificando exactamente?).

✅ `expect(result, 'order should be created with pending status').toMatchObject({ status: 'pending' })`.

### 8. Sesgos del training data

LLMs sobrerrepresentan inglés, USD, nombres anglosajones, formatos de fecha americanos. Para audiencia hispanohablante hay que **forzar locale en el prompt**:

```
Datos de prueba en español: nombres como "Ana García", "Carlos Pérez";
emails con dominios .es; importes en EUR; fechas en formato DD/MM/YYYY.
```

---

## Self-healing tests — categorías reales en 2026

El término "self-healing" es marketing en muchos vendors. Hay **tres niveles tecnológicos** reales:

| Nivel | Cómo funciona | Tecnología | Determinismo | Ejemplo |
|---|---|---|---|---|
| **1. Selector-level** | Si un selector falla, busca alternativas usando LCS, vecindad DOM, atributos similares | ML clásico | Alto | Healenium |
| **2. Visual** | Si la diferencia visual es menor a umbral X, tolera | Algoritmos de comparación de imagen + ML | Medio | Applitools Visual AI, Percy |
| **3. Semántico** | Si el botón "Add to cart" ahora es "Buy now", entiende equivalencia | LLM | Bajo | Testim, Mabl, algunos modos de Applitools |

**Mensaje crítico al evaluar vendors**: pregunta siempre qué nivel implementan. "Self-healing AI-powered" sin más es vapor.

### Cuándo el self-healing es peligroso

> Un test que se "auto-cura" cuando un botón crítico **desaparece** de la UI es exactamente el fallo que NO queremos auto-curar.

Si tu botón de pago se elimina por accidente en una refactor y el test "se cura" buscando el botón más parecido, has enmascarado una regresión grave. Revisa siempre los diffs de healing; nunca aceptes healings ciegamente.

### Política recomendada

1. Self-healing de nivel 1 (Healenium o equivalente) **activado en CI** con diff visible en el reporte.
2. Self-healing visual y semántico **solo en desarrollo local**, nunca en CI sin revisión humana.
3. Cualquier healing aplicado debe generar un **PR de actualización del test** con el cambio visible, no commitearse silenciosamente.

---

## Plataformas SaaS de testing con IA — ejes de decisión

Vista taxonómica de lo que un developer puede encontrarse. **No memorizar herramientas**; entender ejes.

### Ejes de evaluación

| Eje | Pregunta a hacerse |
|---|---|
| Open source vs SaaS | ¿Quiero control del código o servicio gestionado? |
| Self-service vs gestionado | ¿Mi equipo configura o lo entrega un proveedor? |
| Basado en DOM vs basado en visión | ¿Apps web tradicionales o canvas / PDF / móvil? |
| Stack JS/TS vs políglota | ¿Mi pipeline es Node o multi-lenguaje? |
| Free tier vs enterprise | ¿Cuánto puede probarse antes de comprar? |

### Categorías y ejemplos representativos

| Categoría | Ejemplos | Notas |
|---|---|---|
| Unit test generation | **Qodo** (TS/JS/Python), **Diffblue Cover** (Java, RL no LLM), EarlyAI | Qodo es el más completo en JS/TS; Diffblue es ejemplo de ML clásico ganando a LLMs |
| E2E con agente IA + free tier | **Octomind**, **Checksum**, **Reflect** | Para empezar sin coste |
| Visual regression | **Chromatic** (Storybook), **Percy** (BrowserStack), **Applitools** (enterprise) | Elegir según tooling existente |
| Visual regression sin coste | `toHaveScreenshot()` de Playwright + **Argos CI** o **Lost Pixel** | Para enseñar concepto sin presupuesto |
| Self-healing E2E enterprise | **Mabl**, **Functionize**, **Testim** | Caros, foco enterprise |
| Observabilidad con IA | **Datadog Synthetics + AI**, **Sentry Session Replay**, **LogRocket** | Complementan testing pre-release |
| Test data generation | **Faker.js** (mocks), **Tonic.ai** (anonimización de BD) | NO son competidores: resuelven problemas distintos |

⚠ **Confusiones comunes a evitar**:

- **Meticulous** NO genera unit tests pese a su nombre/posicionamiento; pertenece a regresión visual/E2E autogenerada desde recordings de sesiones reales.
- **TestCafe** se ha retirado de recomendaciones para 2026. Mantenimiento mínimo, foco comercial movido a TestCafe Studio (de pago). No recomendar para nuevos proyectos.

---

## Trazabilidad de prompts en CI (obligatorio)

Cuando los tests son generados por LLM, **versionar el prompt junto al test**. Sin esto, en 6 meses nadie sabrá por qué un test asserta lo que asserta.

### Patrón recomendado

```
tests/
├── e2e/
│   ├── login.spec.ts
│   └── login.prompt.md       # Prompt original que generó el test
└── README.md                  # Política de tests generados por IA
```

`tests/e2e/login.prompt.md`:

```markdown
# Prompt: tests/e2e/login.spec.ts

Generado con: Claude Sonnet 4.5 vía Cursor, 2026-04-12
Revisado por: Ana García (PR #234)

## Prompt original

[pega aquí el prompt exacto que se usó]

## Decisiones de revisión humana

- Añadidos asserts de a11y que el LLM omitió.
- Eliminado test redundante de "credenciales vacías" (ya cubierto en login.test.ts unit).
- Forzado locale es-ES para emails y nombres.
```

### Gates en CI para tests con IA

En `.github/workflows/test.yml` o equivalente:

1. **Lint de los tests** (regla "queries accesibles obligatorias").
2. **Ejecución completa** (no retries silenciosos en main).
3. **Flakiness check**: si un test requiere `retry > 0` en main 3 veces seguidas, falla el pipeline.
4. **Coverage de prompts**: cada `*.spec.ts` generado por IA debe tener su `*.prompt.md` (script de verificación en CI).
5. **Code review obligatorio**: PR que toca tests no se mergea sin revisor humano explícito (regla del proyecto).

---

## Riesgos y límites (no negociables)

### 1. Coste en tokens en suites grandes

Una suite E2E con 500 tests donde cada self-healing implica 1-3 llamadas a un LLM tipo Claude Sonnet o GPT-4o puede costar **10-100 USD por ejecución completa**, según prompt size. En CI con varias ejecuciones diarias, puede alcanzar **miles de USD/mes** sin garantía de estabilidad.

**Mitigación**:

- Usar self-healing selector-level (ML clásico) como primera línea.
- Reservar el LLM solo para casos donde el primero falla.
- Usar Playwright CLI (~27k tokens/tarea) sobre Playwright MCP (~114k tokens/tarea) cuando el agente tenga filesystem.
- Cachear resultados de healing entre ejecuciones.

### 2. No-determinismo y reproducibilidad

LLMs con `temperature > 0` producen tests distintos para el mismo prompt: problema si los tests se versionan en Git.

**Mitigación**:

- Forzar `temperature: 0` siempre que el proveedor lo permita.
- Fijar `seed` si el proveedor lo soporta (algunas APIs no garantizan reproducibilidad ni con seed).
- **Nunca** permitir que un LLM genere un test que va directo a main sin revisión humana de PR.

### 3. GDPR / RGPD al enviar datos a LLMs externos

Enviar logs, payloads, capturas o datos de producción con **PII** a OpenAI / Anthropic / Google **sin acuerdo de tratamiento de datos válido** es violación clara de GDPR (Guidance del DSK alemán, informe del ChatGPT Task Force del EDPB).

**Mitigación**:

- Proveedores con **DPA firmado** y región europea (AWS Bedrock EU, Anthropic Enterprise con DPA, etc.).
- **LLMs locales** (Ollama, vLLM con modelos open source) para datos sensibles.
- **Redactar antes de enviar** (Tonic Textual, presidio, custom regex).
- Documentar en `docs/decisions-log.md` qué proveedor se usa y con qué garantías legales.

### 4. EU AI Act y testing automatizado

El AI Act entró en vigor el **1 de agosto de 2024** con calendario escalonado:

| Fecha | Obligación |
|---|---|
| **2 de febrero de 2025** | Prohibiciones de prácticas inaceptables + obligaciones de **AI literacy** (las empresas que usan IA deben formar a su personal) |
| **2 de agosto de 2025** | Obligaciones para modelos de propósito general (GPAI) y supervisión por la AI Office |
| **2 de agosto de 2026** | Obligaciones de sistemas de alto riesgo del Annex III (empleo, crédito, educación, justicia, etc.) |

El Digital Omnibus propuesto en noviembre 2025 plantea aplazar parte de las obligaciones de alto riesgo hasta diciembre 2027, pero **no es prudente asumir el aplazamiento**.

**¿Afecta a herramientas de testing?**

- **Directamente, muy poco**: testing automatizado de software no entra como sistema de alto riesgo per se.
- **Indirectamente, sí**:
  - Si la app que tu IA está testeando es de **alto riesgo** (RRHH, salud, crédito, educación), tus tests forman parte del **expediente de calidad** y deben documentarse.
  - La obligación de **AI literacy** obliga a formar al equipo en uso responsable de las herramientas de IA aplicadas a testing (relevante desde febrero 2025).

**Multas**: hasta **35 M€ o 7% de facturación global** por violar prohibiciones; **15 M€ o 3%** por otros incumplimientos.

**Mitigación**:

- Documentar en `docs/decisions-log.md` qué herramientas de IA se usan en testing y con qué política.
- Si la app testeada es de alto riesgo, los tests entran en el expediente: archivar prompts, outputs y revisiones humanas.
- Formar al equipo en uso responsable (AI literacy): no enviar datos sensibles a LLMs externos sin política clara.

### 5. Sesgos en tests generados por IA

Ya mencionado en anti-patrones, pero es **riesgo de cumplimiento**: si tu app es de alto riesgo y los tests están sesgados a una sola demografía, la auditoría regulatoria lo detectará.

**Mitigación**:

- Incluir en el prompt: dominio, idioma, locale, formatos esperados.
- Tests con datos diversos: nombres en múltiples idiomas, fechas en formatos locales, emails de varios dominios.

### 6. Mantenimiento del código generado por IA

Tests generados sin entender el dominio se convierten en test theater. **Code review estricta, no automática.**

### 7. Casos donde la IA NO funciona bien en testing (a 2026)

| Caso | Por qué falla la IA | Alternativa |
|---|---|---|
| Tests de performance / carga | Requiere modelado de concurrencia | k6, Gatling, JMeter |
| Tests de seguridad profundos (fuzzing, SAST, DAST) | Herramientas dedicadas son muy superiores | Semgrep, Snyk, OWASP ZAP, Burp |
| Contract testing entre microservicios | Especificaciones formales ganan | Pact, Specmatic |
| Property-based testing | Conceptualmente diferente | fast-check, Hypothesis, jqwik |
| Sistemas con lógica regulada y poco documentada | El LLM inventa | Especificación humana primero |

---

## Integración con el harness Architect-Brain

- **`On(testing_setup)`** menciona esta skill si el usuario indica que va a usar agentes IA para tests. Carga `ai-testing-skill` como guía adicional.
- **`On(task_complete)`** verifica que los tests generados por IA tienen su `.prompt.md` asociado.
- **`/cuestionar`** aplica especialmente a tests generados por IA: ¿el test detecta mutaciones realistas o es test theater?
- **`/sync`** detecta `*.spec.ts` sin `*.prompt.md` correspondiente como drift documental.
- **`/revisar-frontend`** y **`/revisar-bd`** incluyen en su auditoría el cumplimiento normativo (AI Act, GDPR) si la app es de alto riesgo.

---

## Checklist antes de mergear tests generados por IA

Code review obligatoria. Marcar antes de aprobar:

- [ ] El test usa queries accesibles (`getByRole`, `getByLabel`, `getByTestId`). Cero selectores de clase ni IDs autogenerados.
- [ ] El test verifica resultado observable, no calls a mocks.
- [ ] No hay snapshot tests gigantes (>50 nodos).
- [ ] Los datos de prueba son realistas y respetan el locale del proyecto.
- [ ] No hay duplicación masiva de setup (extraer a `beforeEach`).
- [ ] El test pasa 3 veces seguidas sin retries en main.
- [ ] Existe `*.prompt.md` con el prompt original y notas de revisión.
- [ ] Si la app es de alto riesgo (AI Act): el test entra en el expediente de calidad.
- [ ] Si se enviaron datos al LLM: no contenían PII sin DPA válido.

---

## Antipatrones a evitar

- **Dejar a la IA decidir qué hay que testear**. Eso es trabajo humano (priorización por valor + riesgo).
- **Mergear sin code review** porque "los tests están en verde". Tests en verde no significa tests útiles.
- **Usar el mismo prompt genérico para todo el proyecto**. Cada feature merece un prompt específico con su dominio y restricciones.
- **No versionar prompts**: en 6 meses nadie sabrá por qué un test asserta lo que asserta.
- **Pagar por self-healing semántico cuando bastaría con selector-level**.
- **Enviar datos de producción a un LLM externo sin DPA**. Violación GDPR.
- **Confiar en una sola plataforma SaaS sin plan B**: si Anthropic / OpenAI tienen downtime, tu pipeline también.

---

## Regla de oro final

> **El test asistido por IA solo es valioso si sigue siendo un test escrito por un humano que entiende el dominio. La IA es la pluma, no el autor.**

Cuando la IA es el autor, los tests son test theater. Cuando la IA es la pluma y el humano el autor, los tests son 5-10x más rápidos de escribir sin perder calidad. Esa es la diferencia entre un equipo que adopta IA bien y uno que se llena de deuda silenciosa.
