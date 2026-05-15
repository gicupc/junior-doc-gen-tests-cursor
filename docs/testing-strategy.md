<!-- TEMPLATE-PLACEHOLDER -->
# Testing Strategy

> Este documento lo rellena el Architect-Brain durante el protocolo `On(testing_setup)` o al elegir la opción [4] en proyectos existentes.
> No editar a mano sin avisar a la IA — es parte del SSOT.

---

## Pirámide de tests del proyecto

Este proyecto sigue la siguiente distribución de niveles de testing. Cada nivel cubre preguntas diferentes y tiene su skill asociada en `.cursor/skills/`.

| Nivel | Pregunta que responde | Skill | Cobertura objetivo |
|---|---|---|---|
| **Unit** | ¿Esta función/hook/clase se comporta correctamente con estos datos? | `tests-skill` | Toda la lógica crítica |
| **Integración** | ¿Las piezas reales colaboran correctamente (API + BD + servicios)? | `integration-testing-skill` | Endpoints HTTP, flujos servicio-repositorio, componentes con fetch real |
| **E2E + visual + a11y** | ¿El sistema completo funciona como un usuario lo espera? | `visual-testing-skill` | Flujos críticos del usuario (login, checkout, alta) |
| **BDD / Aceptación** *(opcional)* | ¿El comportamiento cumple los criterios acordados con negocio? | `bdd-skill` | Solo features acordadas vía Three Amigos |
| **Legacy / Characterization** *(condicional)* | ¿Qué hace HOY este código legacy antes de tocarlo? | `legacy-testing-skill` | Solo código legacy sospechoso |

*Pirámide flexible 2026*: la pirámide clásica de Mike Cohn (muchas unit, pocas E2E) sigue siendo la guía, pero con Playwright + paralelización el "Testing Trophy" de Kent C. Dodds (énfasis en integración) es alternativa válida. La decisión elegida para este proyecto se documenta como ADR.

*Marcar las casillas:*
- [ ] Unit (obligatorio)
- [ ] Integración (recomendado si hay APIs o BD)
- [ ] E2E + visual + a11y (recomendado si hay frontend crítico)
- [ ] BDD / Aceptación (opcional, solo si hay stakeholders no técnicos)
- [ ] Legacy / Characterization (solo si aplica)

---

## Decisiones

### Framework elegido
*Ejemplo: PHPUnit 10.x / Jest 29.x + ts-jest / Vitest 1.x*

### Razón de la elección
*Ejemplo: PHPUnit por ser estándar de facto en el ecosistema PHP y tener integración madura con Composer.*

### Nivel de cobertura (respuesta T del BMADT)
- [ ] **[1] Básico** — solo lógica crítica.
- [ ] **[2] Estándar** — capa de servicio/dominio completa con mocks de dependencias externas.
- [ ] **[3] Exhaustivo** — tests + métricas de cobertura + integración continua.

---

## Estructura de carpetas

*Ejemplo:*
```
tests/
├── unit/
│   ├── ValidatorTest.php
│   └── services/
│       └── UserServiceTest.php
└── integration/    (si aplica)
```

---

## Convenciones

- **Patrón**: Arrange-Act-Assert (AAA) visible en cada test.
- **Nombres**: frases descriptivas en inglés o español, consistente con el idioma del proyecto.
- **Datos de prueba**: realistas pero mínimos, constantes al principio del archivo o en fixtures dedicadas.
- **Aislamiento**: `beforeEach` / `setUp` limpia mocks y estado compartido.

---

## Estrategia de mocks

### Qué se mockea
- Base de datos (framework ORM: *[Prisma / PDO / Eloquent / TypeORM / ...]*).
- APIs externas.
- Sistema de archivos.
- Tiempo (`Date.now()`, equivalentes) cuando el test depende de ello.

### Qué NO se mockea
- Lógica pura del propio módulo.
- Funciones helper deterministas sin dependencias externas.

### Patrón de mock para BD
*[Pegar aquí el snippet de cómo se monta el mock de la BD en este proyecto. Ejemplo para Prisma + Jest, PDO en PHPUnit, etc. — para que todos los tests sigan el mismo estilo.]*

#### Patrones de referencia por stack

**Prisma + Jest (Node/TypeScript)** — usar `jest.fn(() => obj)` (NO `mockImplementation`):

```ts
jest.mock('@prisma/client', () => {
    const mPrisma = {
        user: { findMany: jest.fn(), findUnique: jest.fn(), update: jest.fn() },
    };
    return { PrismaClient: jest.fn(() => mPrisma) };
});

import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient() as jest.Mocked<any>;

beforeEach(() => jest.clearAllMocks());
(prisma.user.findUnique as jest.Mock).mockResolvedValue({ id: 1, name: 'Ana' });
```

**Razón crítica:** si el código de producción usa Active Record (cada modelo de dominio hace `const prisma = new PrismaClient()` dentro del archivo), `mockImplementation` crea una instancia nueva por cada llamada, y los mocks configurados en el test NO alcanzan a la instancia interna del modelo. Con `jest.fn(() => mPrisma)` todas las invocaciones devuelven el mismo objeto.

**PDO + PHPUnit (PHP)**:

```php
$pdo = $this->createMock(PDO::class);
$stmt = $this->createMock(PDOStatement::class);
$pdo->method('prepare')->willReturn($stmt);
$stmt->method('execute')->willReturn(true);
$stmt->method('fetch')->willReturn(['id' => 1, 'name' => 'Ana']);
```

Recordar instalar `phpstan/phpstan-phpunit` para evitar falsos positivos del linter en `->method()` y `->expects()`.

---

## Cobertura actual

### Módulos cubiertos
*Lista que se actualiza cada vez que se cierra un ticket con tests asociados.*

- [ ] Validadores
- [ ] Servicios de dominio
- [ ] Controladores / endpoints (si aplica)

### Módulos pendientes
*Tickets pendientes de cobertura en `roadmap.md`.*

---

## Scripts de ejecución

*Ejemplo:*
- `npm test` → corre toda la suite.
- `npm run test:watch` → modo watch para desarrollo.
- `npm run test:coverage` → reporte de cobertura *(solo en nivel exhaustivo)*.

---

## Excepciones documentadas

*Si alguna funcionalidad se cerró como `[x]` en el roadmap sin tests asociados, razón aquí.*

Ejemplo:
- `Migración de esquema inicial de BD` — no testeable unitariamente, verificación manual en entorno de staging.

---

## BDD (opcional)

*Rellenar solo si el proyecto adopta BDD vía `/bdd`.*

### Herramienta elegida
*Ejemplo: playwright-bdd 8.x sobre @playwright/test 1.59*

### Estructura de carpetas BDD
```
features/
├── *.feature           # Especificaciones Gherkin
└── steps/
    └── *.steps.ts      # Step definitions
```

### Política de adopción
- [ ] Three Amigos obligatorios antes de codificar features con tag `@critical`.
- [ ] Lenguaje del dominio acordado y documentado.
- [ ] Anti-patrones de Gherkin generado por IA conocidos por el equipo (ver `bdd-skill`).

### Tags semánticos en uso
*Ejemplo:*
- `@smoke` — suite rápida que corre en cada PR.
- `@regression` — suite completa que corre en main.
- `@critical` — flujos críticos del negocio; requieren Three Amigos.

---

## Testing con IA

*Rellenar si el proyecto usa herramientas de IA en su flujo de testing.*

### Agentes y herramientas en uso
*Ejemplo:*
- **Generación de tests E2E**: Cursor Agent con Playwright MCP (plantilla activada en `.cursor/mcp.json`).
- **Generación de unit tests**: Cursor con prompts en `prompts/templates/`.
- **Visual regression**: Argos CI.
- **Self-healing**: Healenium (selector-level, ML clásico).

### Trazabilidad de prompts
- [ ] Cada test generado por IA tiene su `*.prompt.md` asociado.
- [ ] Los prompts se versionan en Git junto al test.
- [ ] PR de tests generados por IA requiere revisor humano explícito.

### Gates en CI
- [ ] Tests con `retry > 0` en main 3 ejecuciones seguidas fallan el pipeline.
- [ ] Script CI verifica que cada `*.spec.ts` generado tiene `*.prompt.md`.
- [ ] Linter rechaza selectores frágiles (`#mui-*`, `.css-1*`).

Documentación completa en `.cursor/skills/ai-testing-skill/SKILL.md`.

---

## Cumplimiento normativo (si aplica)

*Rellenar solo si el proyecto procesa datos personales o cae bajo EU AI Act como sistema de alto riesgo.*

### GDPR / RGPD
- [ ] Si se envían datos a LLMs externos para generar/mantener tests: existe DPA con el proveedor.
- [ ] Fixtures de test NO contienen PII real (usar `@faker-js/faker` con locale del proyecto).
- [ ] Logs de tests con datos sensibles se redactan antes de almacenarse.

### EU AI Act
- [ ] Equipo formado en AI literacy (obligatorio desde 2 febrero 2025 si usa IA).
- [ ] Si la app es de **alto riesgo** (Annex III: empleo, crédito, educación, salud, justicia): los tests forman parte del expediente de calidad y se archivan junto a los prompts y revisiones.
- [ ] Decisión sobre proveedor de LLM y región registrada en `docs/decisions-log.md`.

Referencia: `.cursor/skills/ai-testing-skill/SKILL.md` sección "Riesgos y límites".

---

## Supply chain (si el proyecto usa npm registry)

*Rellenar solo si el proyecto tiene `package.json` (Node.js, JS/TS, frontend con npm).*

Las defensas detalladas viven en `docs/supply-chain-strategy.md`. Esta sección resume la intersección con la estrategia de testing.

### Configuración del package manager
*Ejemplo:*
- Package manager: **pnpm 11** (`packageManager` field en `package.json`).
- `minimumReleaseAge: 1440` (1 día, default pnpm 11).
- `blockExoticSubdeps: true` (default pnpm 11).
- `allowBuilds`: electron, esbuild, sharp, @swc/core (justificaciones en `supply-chain-strategy.md`).

### Tests con dependencias muy nuevas
- [ ] Tests de integración y E2E NO instalan paquetes con menos de 24h en CI.
- [ ] Si un test requiere un paquete recien publicado, se documenta como excepción en `minimumReleaseAgeExclude` con ADR.

### CI hardening (GitHub Actions)
- [ ] Workflows de test ejecutan `pnpm install --frozen-lockfile` (NUNCA `pnpm install` a secas).
- [ ] NUNCA `pull_request_target` con checkout del fork.
- [ ] Acciones pineadas a SHA, no a tag mutable.
- [ ] Cache no compartida entre workflows con distinto trust boundary.

### Respuesta a incidente
- [ ] Equipo conoce el orden critico: NO revocar token de GitHub antes de limpiar daemon de persistencia.
- [ ] Lockfile cross-referenciado regularmente con bases publicas (TanStack GHSA-g7cv-rxg3-hmpx, Shai-Hulud, axios, Trivy).

Documentación completa en `.cursor/skills/supply-chain-skill/SKILL.md`.
