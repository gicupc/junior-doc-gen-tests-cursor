---
name: integration-testing-skill
description: Pruebas de integracion (capa intermedia entre unit y E2E). Usar cuando el proyecto expone APIs HTTP o tiene logica que cruza multiples capas (controller -> service -> repository -> DB). Cubre Supertest, MSW (mocks de red reutilizables), Testcontainers (BD real en Docker), Pact (contract testing) con patrones completos para Node/PHP/Python/JVM. Captura ~70% de bugs de integracion con ~30% del coste de un E2E. Complementaria a tests-skill (unit) y visual-testing-skill (E2E + a11y).
---

# Skill: Pruebas de Integración

Esta skill cubre la **capa intermedia** de la pirámide de testing: tests que verifican la interacción entre múltiples componentes o sistemas (API + base de datos, servicio + repositorio, frontend + backend mediante mocks de red), pero **sin la complejidad ni la lentitud de un E2E completo en navegador real**.

Es complementaria a `tests-skill` (unit) y `visual-testing-skill` (E2E + a11y). Cuando se usa correctamente, captura ~70% de los bugs de integración con ~30% del coste de un E2E.

---

## Principio fundamental

> **Los tests unitarios mienten cómodamente. Los tests E2E son caros. Los de integración están en el punto óptimo de la curva coste/valor.**

Un test unitario con todo mockeado puede pasar en verde aunque la integración real esté rota (el mock no refleja el contrato real de la dependencia). Un test E2E captura el bug, pero requiere navegador, datos seed, autenticación y entorno completo. El test de integración corre **componentes reales contra dependencias reales o de alta fidelidad** (BD en Docker, MSW interceptando red), sin la sobrecarga del navegador.

---

## Cuándo activar esta skill

- El proyecto expone APIs HTTP (REST, GraphQL, gRPC) que otros sistemas o el frontend consumen.
- Hay lógica que cruza múltiples capas (controller → service → repository → DB).
- Tienes mocks de red en frontend que también querrías reutilizar en E2E (MSW).
- Has tenido bugs en producción que los unit tests "cubrían" pero el contrato real estaba roto.

NO activar si:

- El proyecto es 100% lógica pura sin dependencias externas (entonces unit tests bastan).
- Ya tienes E2E completos cubriendo cada endpoint (entonces integration testing sería redundante).

---

## Qué es y qué NO es un test de integración

**SÍ es integración:**

- Endpoint HTTP + lógica de servicio + BD real (Testcontainers) ejecutándose juntos.
- Componente React + fetch real interceptado por MSW que devuelve fixtures realistas.
- Servicio que consume una cola (Redis, RabbitMQ) con la cola corriendo en Docker.
- Job programado que escribe en BD y dispara un webhook (con el receptor mockeado a nivel HTTP).

**NO es integración (es unitario):**

- Servicio con repositorio mockeado a nivel de objeto JavaScript/PHP.
- Componente React con fetch global stubeado a nivel de función.
- Cualquier test donde la dependencia externa es un objeto fake en memoria.

**NO es integración (es E2E):**

- Test que abre navegador real, navega a la app, hace login y completa un flujo completo.
- Test que despliega el sistema completo en staging y lo verifica con peticiones reales.

La diferencia operativa: **integración usa componentes reales, pero NO arranca el navegador ni todo el stack de UI**.

---

## Stacks y herramientas en 2026

### Node.js / TypeScript

| Caso | Herramienta |
|---|---|
| Tests de API HTTP (Express, Fastify, NestJS) | **Supertest** + Vitest/Jest |
| Mocks de red reutilizables (front + integración + E2E) | **MSW** (Mock Service Worker) |
| BD real en Docker durante tests | **Testcontainers** (`@testcontainers/postgresql`, `@testcontainers/mongodb`, etc.) |
| Contract testing entre microservicios | **Pact** (`@pact-foundation/pact`) |
| Tests con colas reales | **Testcontainers** + cliente real |

### PHP

| Caso | Herramienta |
|---|---|
| Tests de API HTTP | **PHPUnit** + cliente HTTP real contra app levantada en modo test |
| BD real durante tests | **Testcontainers PHP** o transacciones rollback en PHPUnit |
| Contract testing | **Pact PHP** |

### Python

| Caso | Herramienta |
|---|---|
| Tests de API (FastAPI, Flask, Django) | **pytest** + **TestClient** (httpx-based) |
| BD real durante tests | **pytest-postgresql** o **Testcontainers Python** |
| Mocks de red | **responses** o **respx** (httpx-native) |
| Contract testing | **Pact Python** |

### Java / JVM

| Caso | Herramienta |
|---|---|
| Tests de API (Spring Boot) | **MockMvc** o **WebTestClient** |
| BD real | **Testcontainers** (origen del proyecto) |
| Contract testing | **Pact JVM** o **Spring Cloud Contract** |

**Regla 2026**: Testcontainers es el estándar de facto para tests con dependencias reales en cualquier stack. MSW es estándar para mocks de red en JS/TS. Pact es estándar para contract testing cross-stack.

---

## Patrón 1: Test de API HTTP con Supertest + BD real (Node + Testcontainers)

Estructura recomendada:

```
tests/
├── integration/
│   ├── helpers/
│   │   ├── testcontainer-db.ts
│   │   └── test-app.ts
│   ├── users.api.test.ts
│   └── orders.api.test.ts
└── unit/
    └── ...
```

`tests/integration/helpers/testcontainer-db.ts`:

```ts
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { PrismaClient } from '@prisma/client';

let container: StartedPostgreSqlContainer;
let prisma: PrismaClient;

export async function startTestDb(): Promise<{ prisma: PrismaClient; connectionString: string }> {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('testdb')
    .withUsername('test')
    .withPassword('test')
    .start();

  const connectionString = container.getConnectionUri();
  process.env.DATABASE_URL = connectionString;

  prisma = new PrismaClient({ datasources: { db: { url: connectionString } } });

  // Aplica migraciones
  const { execSync } = await import('node:child_process');
  execSync('npx prisma migrate deploy', { env: { ...process.env, DATABASE_URL: connectionString } });

  return { prisma, connectionString };
}

export async function stopTestDb(): Promise<void> {
  await prisma?.$disconnect();
  await container?.stop();
}

export function getPrisma(): PrismaClient {
  return prisma;
}
```

`tests/integration/users.api.test.ts`:

```ts
import request from 'supertest';
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { startTestDb, stopTestDb, getPrisma } from './helpers/testcontainer-db';
import { createApp } from '../../src/app';

let app: ReturnType<typeof createApp>;

beforeAll(async () => {
  await startTestDb();
  app = createApp();
}, 60_000); // Testcontainers puede tardar la primera vez

afterAll(async () => {
  await stopTestDb();
});

beforeEach(async () => {
  // Limpia datos entre tests pero NO la BD entera
  await getPrisma().user.deleteMany();
});

describe('POST /api/users', () => {
  it('creates a new user with valid data', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ email: 'ana@test.com', name: 'Ana' });

    expect(response.status).toBe(201);
    expect(response.body).toMatchObject({
      id: expect.any(Number),
      email: 'ana@test.com',
      name: 'Ana',
    });

    // Verifica que ESTÁ en la BD, no solo en la respuesta
    const inDb = await getPrisma().user.findUnique({ where: { email: 'ana@test.com' } });
    expect(inDb).not.toBeNull();
  });

  it('rejects duplicate emails with 409', async () => {
    await getPrisma().user.create({ data: { email: 'ana@test.com', name: 'Ana' } });

    const response = await request(app)
      .post('/api/users')
      .send({ email: 'ana@test.com', name: 'Otra Ana' });

    expect(response.status).toBe(409);
  });

  it('rejects invalid email format with 400', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ email: 'no-arroba', name: 'Ana' });

    expect(response.status).toBe(400);
    expect(response.body.error).toMatch(/email/i);
  });
});
```

**Reglas no negociables:**

- BD real, NO mockeada. Si quieres mocks de BD, eso es un unit test.
- Limpia datos entre tests con `deleteMany()` o transacciones rollback, **no** reinicies el container (lento).
- Verifica el estado de BD tras la petición, no solo el body de respuesta (un controller puede devolver 201 sin haber persistido).
- Timeouts generosos en `beforeAll` (60s) porque Testcontainers descarga imagen la primera vez.

---

## Patrón 2: MSW para mocks de red reutilizables

MSW (Mock Service Worker) intercepta peticiones HTTP a nivel de red. Los **mismos handlers** se usan en tests de integración del frontend, en desarrollo local y opcionalmente en E2E.

Estructura:

```
src/
├── mocks/
│   ├── handlers.ts        # Handlers compartidos
│   ├── server.ts          # Setup para Node (tests + dev SSR)
│   └── browser.ts         # Setup para navegador (dev frontend)
└── ...
tests/
└── integration/
    └── checkout.test.tsx  # Usa los mismos handlers
```

`src/mocks/handlers.ts`:

```ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({
      id: Number(params.id),
      email: 'ana@test.com',
      name: 'Ana',
    });
  }),

  http.post('/api/orders', async ({ request }) => {
    const body = (await request.json()) as { items: unknown[] };
    if (!body.items || body.items.length === 0) {
      return HttpResponse.json({ error: 'Empty order' }, { status: 400 });
    }
    return HttpResponse.json({ id: 'ord_123', status: 'pending' }, { status: 201 });
  }),
];
```

`src/mocks/server.ts`:

```ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

`tests/setup.ts`:

```ts
import { beforeAll, afterAll, afterEach } from 'vitest';
import { server } from '../src/mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

`tests/integration/checkout.test.tsx`:

```ts
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { http, HttpResponse } from 'msw';
import { server } from '../../src/mocks/server';
import { Checkout } from '../../src/components/Checkout';

describe('Checkout integration', () => {
  it('shows success when order is created', async () => {
    render(<Checkout items={[{ id: 1, qty: 1 }]} />);

    await userEvent.click(screen.getByRole('button', { name: /place order/i }));
    expect(await screen.findByText(/order confirmed/i)).toBeInTheDocument();
  });

  it('shows error when server returns 500', async () => {
    // Override puntual del handler para este test
    server.use(
      http.post('/api/orders', () => {
        return HttpResponse.json({ error: 'Internal error' }, { status: 500 });
      })
    );

    render(<Checkout items={[{ id: 1, qty: 1 }]} />);
    await userEvent.click(screen.getByRole('button', { name: /place order/i }));
    expect(await screen.findByRole('alert')).toHaveTextContent(/something went wrong/i);
  });
});
```

**Por qué MSW gana en 2026:**

- Mismos handlers en **tests + dev local + E2E** (vía `mockServiceWorker.js`).
- Intercepta a nivel de red, no de función: el código de producción usa `fetch` real, no un cliente mockeado.
- Permite **overrides puntuales** por test sin tocar los handlers base.
- Funciona con cualquier cliente HTTP (`fetch`, `axios`, `ky`, `got`).

---

## Patrón 3: PHP + PHPUnit + cliente HTTP real

Para proyectos PHP (Laravel, Symfony, Slim, plain PHP):

```php
// tests/Integration/UserApiTest.php
use PHPUnit\Framework\TestCase;
use GuzzleHttp\Client;

class UserApiTest extends TestCase
{
    private static Client $http;
    private static PDO $db;

    public static function setUpBeforeClass(): void
    {
        // App levantada en modo test mediante un script bash o docker-compose previo
        self::$http = new Client(['base_uri' => $_ENV['TEST_BASE_URL'] ?? 'http://localhost:8081']);
        self::$db = new PDO($_ENV['TEST_DATABASE_DSN'], $_ENV['TEST_DB_USER'], $_ENV['TEST_DB_PASS']);
    }

    protected function setUp(): void
    {
        // Limpia tabla entre tests
        self::$db->exec('TRUNCATE TABLE users');
    }

    public function test_creates_user_with_valid_data(): void
    {
        $response = self::$http->post('/api/users', [
            'json' => ['email' => 'ana@test.com', 'name' => 'Ana'],
        ]);

        $this->assertEquals(201, $response->getStatusCode());

        $body = json_decode((string) $response->getBody(), true);
        $this->assertEquals('ana@test.com', $body['email']);

        // Verifica en BD
        $stmt = self::$db->prepare('SELECT * FROM users WHERE email = ?');
        $stmt->execute(['ana@test.com']);
        $this->assertNotFalse($stmt->fetch());
    }
}
```

Pre-requisitos: la app debe levantarse en modo test antes de la suite. Patrones comunes:

- **Docker Compose** con servicios `app-test` y `db-test`.
- **Script `tests/bootstrap.sh`** que arranca la app, espera a que esté listo, ejecuta la suite, la baja.
- **PHPUnit hooks** (`bootstrap` en `phpunit.xml`) para preparar el entorno.

---

## Patrón 4: Contract testing con Pact (mención breve)

Si el proyecto consume o expone APIs entre microservicios, **Pact** evita el clásico "el consumer pasa los tests, el provider rompe en producción":

1. **Consumer test** (en el servicio que llama): define qué espera de la API.
2. **Pact file generado** (JSON): contrato derivado del test.
3. **Provider verification** (en el servicio que expone): el provider verifica que cumple el contrato.

Útil cuando hay >2 microservicios cambiando en paralelo. Overkill para proyectos monolíticos.

Herramientas por stack: `@pact-foundation/pact` (Node), `pact-php`, `pact-python`, `pact-jvm`.

---

## Cuándo escalar: unit → integration → E2E

Reglas operativas para no convertir cada test en E2E:

| Tipo de lógica | Nivel adecuado |
|---|---|
| Función pura, validador, parser, cálculo | **Unit** (`tests-skill`) |
| Servicio con dependencias mockeadas a nivel de interfaz | **Unit** |
| Endpoint HTTP con BD real y lógica completa de servicio | **Integration** (esta skill) |
| Componente React con fetch interceptado por MSW | **Integration** |
| Flujo completo de usuario: login → navegación → acción → logout | **E2E** (`visual-testing-skill`) |
| Verificación visual de pixel-perfect | **E2E + visual diff** |
| Criterio de aceptación de negocio validado por stakeholders | **BDD** (`bdd-skill`) |

**Regla del coste**: si puedes cubrir el caso con el nivel inferior sin perder fidelidad, usa el nivel inferior. Cada nivel hacia arriba multiplica entre 5x y 20x el tiempo de ejecución.

**Regla de la pirámide flexible 2026**: la pirámide clásica de Mike Cohn (muchas unit, pocas E2E) sigue siendo guía, pero con tests rápidos en navegador (Playwright + paralelización) el "Testing Trophy" de Kent C. Dodds (énfasis en integration) es alternativa válida. Documentar la decisión en `docs/testing-strategy.md`.

---

## Convenciones de carpetas

Recomendado:

```
tests/
├── unit/             # tests-skill
├── integration/      # ESTA skill
│   ├── helpers/
│   ├── api/
│   └── components/   # Si hay frontend con MSW
├── e2e/              # visual-testing-skill
└── bdd/              # bdd-skill (si aplica)
    ├── features/
    └── steps/
```

Documentar la convención exacta en `docs/testing-strategy.md`.

---

## Scripts en `package.json` (Node)

```json
{
  "scripts": {
    "test": "vitest",
    "test:unit": "vitest run tests/unit",
    "test:integration": "vitest run tests/integration --pool=forks",
    "test:e2e": "playwright test",
    "test:bdd": "bddgen && playwright test",
    "test:ci": "npm run test:unit && npm run test:integration && npm run test:e2e"
  }
}
```

`--pool=forks` recomendado para integración: evita problemas de aislamiento entre tests cuando hay BD real compartida.

---

## Integración con el harness Architect-Brain

- **`On(testing_setup)`** instala Vitest/Jest + Testcontainers + MSW cuando el proyecto tiene BD o frontend con red. Lo hace automáticamente para nivel T 2 o 3.
- **`On(database_setup)`** registra que los tests de integración usarán Testcontainers en `database-strategy.md`.
- **`On(task_complete)`** verifica que las nuevas APIs tienen al menos un test de integración (no solo unitario).
- **`/cuestionar`** aplica también a tests de integración: ¿el test cubre el caso de BD vacía? ¿de duplicados? ¿de constraint violation?
- **`/revisar-bd`** detecta APIs sin tests de integración como hallazgo de rendimiento (queries no optimizadas que solo se ven con datos reales).

---

## Antipatrones a evitar

- **Mockear la BD en tests llamados "integration"**. Si está todo mockeado, es unit con etiqueta engañosa.
- **Reiniciar el container entre tests**. Lentísimo. Mejor: `TRUNCATE` o `deleteMany()` entre tests, container compartido toda la suite.
- **Compartir estado entre tests** (un test crea un usuario que el siguiente usa). Acoplamiento brutal: si reordenan, todo rompe.
- **Tests de integración que dependen de internet real**. Usa MSW para APIs externas. Tests offline son tests reproducibles.
- **`expect(response.status).toBe(200)` y nada más**. Verifica también el body y, sobre todo, **el estado de BD/cola/etc.** tras la petición.
- **Suites de integración sin paralelización**. En 2026 son baratas. Configura `--pool=forks` o equivalente.
- **No documentar el setup**: si el siguiente developer no puede correr `npm run test:integration` desde clean clone, el test no existe.

---

## Regla de oro final

> **El test de integración es el único que demuestra que la suma de las partes funciona. Los unit tests demuestran que las partes funcionan; los E2E demuestran que el sistema completo funciona; pero el contrato entre piezas vive en integración.**

Si saltas esta capa, los bugs aparecen en producción donde son 10x más caros de arreglar.
