---
name: visual-testing-skill
description: Tests de UI con accesibilidad y E2E. Usar cuando el proyecto tenga frontend con interaccion. Cubre tres niveles - unitario (Vitest), componente (RTL + vitest-axe) y E2E + visual (Playwright + axe-core + opcionalmente pixelmatch). Complementaria a tests-skill (logica pura), integration-testing-skill (capa intermedia) y bdd-skill (aceptacion). Cuando uses agentes IA para tests E2E, carga ademas ai-testing-skill (Playwright MCP/CLI, prompting, anti-patrones, trazabilidad de prompts, GDPR + EU AI Act).
---

# Skill: Visual & Accessibility Testing

Esta skill se activa cuando el proyecto tiene frontend y se desea cerrar el loop de validación visual y de accesibilidad de forma automática. Es complementaria a `tests-skill` (lógica pura), `integration-testing-skill` (capa intermedia) y `bdd-skill` (aceptación).

Cuando uses agentes IA (Cursor, Claude Code, Copilot) para generar tests E2E, **carga además `ai-testing-skill`**: cubre Playwright MCP/CLI, patrones de prompting, anti-patrones, trazabilidad de prompts y cumplimiento normativo (GDPR, EU AI Act).

---

## Principio fundamental

**Hay tres niveles de tests para frontend, y cada uno responde a una pregunta distinta:**

| Nivel | Pregunta que responde | Herramientas | Velocidad |
|---|---|---|---|
| 1. Unitario | ¿Esta función/hook se comporta como espero con estos datos? | Vitest/Jest | Muy rápido |
| 2. Componente | ¿Este componente renderiza/reacciona correctamente y es accesible? | Vitest/Jest + RTL + vitest-axe | Rápido |
| 3. E2E + visual | ¿La página completa se ve y funciona como el diseño en un navegador real? | Playwright + axe-core + pixelmatch | Lento pero determinista |

Saltar el nivel 2 o el 3 es la causa más común de bugs de UI que llegan a producción y de no-conformidad WCAG. Hacerlos los tres bien es lo que separa frontend amateur de profesional.

---

## Nivel 2: tests de componente con accesibilidad

### Setup recomendado

Stack: **Vitest + React Testing Library (RTL) + vitest-axe**.

Instalación:

```bash
npm i -D vitest @vitest/ui jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event vitest-axe
```

`vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./tests/setup.ts'],
  },
});
```

`tests/setup.ts`:

```ts
import '@testing-library/jest-dom/vitest';
import { expect, afterEach } from 'vitest';
import { cleanup } from '@testing-library/react';
import * as matchers from 'vitest-axe/matchers';

expect.extend(matchers);
afterEach(() => cleanup());
```

### Patrón de test de componente con a11y

```ts
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { axe } from 'vitest-axe';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('renders form fields with proper labels', () => {
    render(<LoginForm onSubmit={vi.fn()} />);
    expect(screen.getByLabelText(/email/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/password/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /sign in/i })).toBeInTheDocument();
  });

  it('shows validation error when email is invalid', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={vi.fn()} />);
    await user.type(screen.getByLabelText(/email/i), 'not-an-email');
    await user.click(screen.getByRole('button', { name: /sign in/i }));
    expect(await screen.findByRole('alert')).toHaveTextContent(/invalid email/i);
  });

  it('has no axe accessibility violations', async () => {
    const { container } = render(<LoginForm onSubmit={vi.fn()} />);
    expect(await axe(container)).toHaveNoViolations();
  });
});
```

**Reglas RTL no negociables:**
- Buscar elementos por **rol** (`getByRole`) o **etiqueta accesible** (`getByLabelText`), nunca por clase ni testid si hay alternativa accesible.
- Usar `userEvent` (no `fireEvent`) para simular interacción real.
- Esperar elementos asíncronos con `findBy*`, no con `setTimeout`.

**Reglas axe-core:**
- Cada componente con UI no trivial debe tener **al menos un test `axe(container)` sin violaciones**.
- Si axe reporta violaciones, NO suprimir la regla salvo justificación documentada.

---

## Nivel 3: E2E + validación visual con Playwright

### Setup recomendado

```bash
npm i -D @playwright/test @axe-core/playwright pixelmatch pngjs
npx playwright install chromium
```

`playwright.config.ts`:

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  reporter: [['html'], ['list']],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium-desktop', use: { ...devices['Desktop Chrome'] } },
    { name: 'mobile-safari', use: { ...devices['iPhone 14'] } },
  ],
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Patrón E2E + a11y

```ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Login flow', () => {
  test('user can sign in with valid credentials', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('user@test.com');
    await page.getByLabel('Password').fill('correctpass');
    await page.getByRole('button', { name: 'Sign in' }).click();
    await expect(page).toHaveURL('/dashboard');
  });

  test('login page has no accessibility violations', async ({ page }) => {
    await page.goto('/login');
    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21aa', 'wcag22aa'])
      .analyze();
    expect(results.violations).toEqual([]);
  });
});
```

**Por qué Playwright en lugar de Cypress en 2026:**
- Opera sobre el accessibility tree (determinista).
- Multi-navegador real (Chromium + Firefox + Safari).
- Multi-tab y multi-context nativos.
- `@axe-core/playwright` es la integración oficial recomendada.
- Soporta **Playwright MCP** (ya disponible como plantilla en `.cursor/mcp.json`) y **Playwright CLI** (recomendado para Cursor con filesystem por menor coste en tokens) para que el agente IA pueda generar y mantener tests E2E. Ver `ai-testing-skill` para setup, prompting y gates en CI.

### Loop visual con pixelmatch (opcional pero potente)

Para diseños con alta exigencia de fidelidad visual:

```ts
// tests/visual/visual-utils.ts
import { PNG } from 'pngjs';
import pixelmatch from 'pixelmatch';
import fs from 'node:fs';
import path from 'node:path';

export async function compareScreenshots(
  current: Buffer,
  baselinePath: string,
  diffPath: string,
  threshold = 0.1
): Promise<{ diffPixels: number; total: number }> {
  if (!fs.existsSync(baselinePath)) {
    fs.mkdirSync(path.dirname(baselinePath), { recursive: true });
    fs.writeFileSync(baselinePath, current);
    return { diffPixels: 0, total: 0 };
  }
  const a = PNG.sync.read(fs.readFileSync(baselinePath));
  const b = PNG.sync.read(current);
  const { width, height } = a;
  const diff = new PNG({ width, height });
  const diffPixels = pixelmatch(a.data, b.data, diff.data, width, height, { threshold });
  fs.mkdirSync(path.dirname(diffPath), { recursive: true });
  fs.writeFileSync(diffPath, PNG.sync.write(diff));
  return { diffPixels, total: width * height };
}
```

```ts
// tests/visual/dashboard.spec.ts
import { test, expect } from '@playwright/test';
import { compareScreenshots } from './visual-utils';
import path from 'node:path';

test('dashboard matches design baseline', async ({ page }, testInfo) => {
  await page.goto('/dashboard');
  await page.waitForLoadState('networkidle');
  const screenshot = await page.screenshot({ fullPage: true });
  const baseline = path.join('tests/visual/baselines', `${testInfo.project.name}-dashboard.png`);
  const diffOut = path.join('tests/visual/diffs', `${testInfo.project.name}-dashboard.diff.png`);
  const { diffPixels, total } = await compareScreenshots(screenshot, baseline, diffOut);
  const diffRatio = total === 0 ? 0 : diffPixels / total;
  expect(diffRatio).toBeLessThan(0.005); // <0.5% píxeles distintos
});
```

**Cómo se usa en el ciclo agente IA:**

1. La primera vez que se ejecuta, se crea el baseline a partir del diseño aprobado.
2. En sucesivas ejecuciones, el test pasa si la diferencia es <0.5%.
3. Cuando un cambio intencional rompe el baseline, se regenera tras revisión humana.
4. **Patrón con agente**: agente genera código → Playwright corre → si rojo, agente lee el `diff.png` con su modelo de visión, ajusta código, vuelve a ejecutar.

**Cuándo NO montar pixelmatch:**
- Proyectos con UI dinámica (mucha animación, datos en tiempo real, anuncios).
- Proyectos sin diseño de partida estricto.

---

## Alternativas SaaS para visual regression

Montar `pixelmatch` da control total sin coste, pero hay plataformas SaaS que aportan colaboración, baselines compartidos entre branches y UI de aprobación de diffs:

| Herramienta | Cuándo elegirla | Notas |
|---|---|---|
| **Chromatic** | El proyecto ya usa Storybook | Free tier limitado; integración nativa con Storybook |
| **Percy** (BrowserStack) | El equipo ya usa BrowserStack | Free tier 5k snapshots/mes |
| **Applitools** | Suite enterprise grande, necesita visual AI semántico | Pricing por contacto; el más caro |
| **Argos CI** | Alternativa open-friendly para Playwright/Cypress | Free tier generoso |
| **Lost Pixel** | Open source self-hosted o cloud | Free OSS; opción más flexible |

Recomendación por escenario: Chromatic si Storybook, Argos CI o Lost Pixel para empezar barato con Playwright, Applitools si la suite es grande y necesitas visual AI semántico, Percy si ya pagas BrowserStack.

La decisión se documenta en `docs/frontend-strategy.md` como ADR (criterios: stack existente, coste, necesidad de visual AI semántico, control sobre baselines).

Ver además `ai-testing-skill` para la diferencia entre **visual regression con ML clásico** (Chromatic, Percy, Argos, Lost Pixel) y **visual AI semántico** (Applitools nivel avanzado, Mabl). No confundirlos: el semántico es más tolerante pero menos determinista.

---

## Convenciones de carpetas

```
tests/
  setup.ts                          # Setup global de Vitest
  unit/                             # Lógica pura
    validators.test.ts
  components/                       # Componentes con RTL + axe
    LoginForm.test.tsx
    Button.test.tsx
  e2e/                              # Flujos completos con Playwright
    login.spec.ts
    checkout.spec.ts
  visual/                           # Visual diff (opcional)
    baselines/                      # PNGs de referencia
    diffs/                          # PNGs de diferencias (gitignore)
    visual-utils.ts
    dashboard.spec.ts
```

Documentar la convención en `docs/testing-strategy.md`.

---

## Scripts en `package.json`

```json
{
  "scripts": {
    "test": "vitest",
    "test:unit": "vitest run tests/unit",
    "test:components": "vitest run tests/components",
    "test:e2e": "playwright test",
    "test:visual": "playwright test tests/visual",
    "test:a11y": "playwright test --grep accessibility",
    "test:ci": "vitest run && playwright test"
  }
}
```

---

## Cuándo cubrir cada nivel

| Tipo de código | Nivel obligatorio | Nivel recomendado |
|---|---|---|
| Función pura, validador, parser | 1 (unitario) | — |
| Hook custom con lógica | 1 (unitario) | — |
| Componente con interacción (form, dialog, dropdown) | 2 (componente + axe) | 3 (e2e si está en flujo crítico) |
| Página de feature crítica (login, checkout, onboarding) | 2 + 3 (e2e + a11y) | 3 (visual diff si hay diseño estricto) |
| Componente trivial de presentación (badge, divider, icon) | — | 2 si es base del design system |

Regla operativa: **el test cuesta menos que el bug que evita**.

---

## Integración con el harness Architect-Brain

- **`On(testing_setup)`** ya prepara Vitest/Jest. Cuando hay frontend, esta skill se carga adicionalmente y añade RTL + axe + Playwright al setup.
- **`On(task_complete)`** debe verificar que toda página/componente nuevo tiene su test de a11y.
- **`/cuestionar`** (mutation thinking) sigue aplicando para tests unitarios; para visual y a11y la verificación de calidad es distinta:
  - Visual: ¿El baseline se actualizó con revisión humana o se commiteó ciego?
  - A11y: ¿Las violaciones suprimidas tienen justificación documentada?
- **`/revisar-frontend`** ejecuta auditoría de a11y como una de las 4 categorías.

---

## Antipatrones a evitar

- **Tests de componente que buscan por clase CSS**: si tu test no usa rol o label accesible, probablemente el componente no es accesible. Refactor.
- **Suprimir violaciones de axe sin justificación**: cada violación suprimida debe tener un comentario con la razón y, si es algo a arreglar, un ticket en `roadmap.md`.
- **Snapshot tests gigantes**: hacer `toMatchSnapshot()` de un componente de 200 nodos hace que cualquier cambio rompa el test.
- **Tests E2E que dependen de orden**: cada test crea su propio fixture.
- **Visual diff con umbral 0%**: nunca pasa por sub-pixel rendering. Mínimo 0.1% de tolerancia.
- **Tests de a11y solo automatizados**: tooling detecta ~30%. Para feature crítica, validación manual con NVDA/VoiceOver al menos una vez antes del release.
- **Playwright lanzado contra producción real**: nunca. Usa entornos staging o `webServer` local.

---

## Regla de oro final

> **Lo que no se prueba, no es accesible. Lo que no se mide, no es rápido.**

WCAG 2.2 AA y Core Web Vitals son criterios objetivos y verificables. Si el proyecto los pone como Definition of Done desde día 1, no hay deuda silenciosa al final.
