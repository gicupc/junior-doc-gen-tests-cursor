# Testing Strategy

> Este documento lo rellena el Architect-Brain durante el protocolo `On(testing_setup)` o al elegir la opción [4] en proyectos existentes.
> No editar a mano sin avisar a la IA — es parte del SSOT.

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
