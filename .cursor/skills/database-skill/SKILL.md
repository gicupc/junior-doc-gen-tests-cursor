---
name: database-skill
description: Diseno y auditoria de bases de datos. Usar cuando el proyecto tenga BD - en proyectos nuevos para disenar schema (motor, ORM, PII, indices, audit fields, onDelete), en proyectos existentes para auditar (modelo, seguridad, rendimiento, calidad de datos). Cubre Postgres, MySQL, MongoDB, ORMs principales.
---

# Skill: Base de datos (diseño y auditoría)

Esta skill se activa cuando el proyecto tiene base de datos. Cubre dos escenarios:

1. **Proyecto NUEVO**: cómo diseñar el modelo de datos correctamente desde el principio (qué tablas, qué relaciones, qué índices, qué documentar).
2. **Proyecto EXISTENTE**: cómo auditar una BD que ya está hecha — detectar problemas, generar documentación que falta, generar tests de calidad.

---

## Principio fundamental

**La base de datos es el activo más crítico de cualquier sistema.** Un bug en código se arregla con un deploy. Un bug en BD puede corromper datos para siempre. Por eso:

- En proyectos nuevos: invertir tiempo en el diseño del modelo ANTES de escribir código.
- En proyectos existentes: auditar antes de cambiar. Nunca tocar BD sin red de seguridad (backup + tests).

---

## ESCENARIO 1: Proyecto nuevo

### Decisiones obligatorias antes de escribir código

Cuando la entrevista BMADT (respuesta D = Datos) menciona BD, tomar estas decisiones explícitamente y documentarlas:

#### 1. Motor de BD

| Tipo de datos | Recomendación |
|---|---|
| Relacionales con integridad fuerte | PostgreSQL (referencia open-source) |
| Relacionales pequeños/embebidos | SQLite |
| Documentos flexibles (JSON), alta escala | MongoDB |
| Caché o sesiones | Redis |
| Búsqueda semántica (RAG, embeddings) | Postgres + extensión pgvector |

Por defecto: PostgreSQL salvo razón específica.

#### 2. ORM

| Stack | ORM recomendado |
|---|---|
| Node.js / TypeScript | Prisma (referencia) o Drizzle (si Cloudflare Workers) |
| PHP | Eloquent (Laravel) o Doctrine (Symfony) |
| Python | SQLAlchemy o Django ORM |
| Java | JPA / Hibernate |

#### 3. Identificar campos PII (datos personales)

PII = información personal identificable. Es **obligatorio** identificar al inicio del proyecto qué campos contendrán PII porque tienen requisitos legales (GDPR, LOPD).

Ejemplos típicos: `email`, `phone`, `address`, `dni`, `nombre completo`, `fecha_nacimiento`, `ip_address`.

Para cada campo PII, decidir:
- ¿Se guarda en texto plano o cifrado?
- ¿Quién puede leerlo (RLS, Row Level Security)?
- ¿Cuánto tiempo se conserva (retention policy)?

#### 4. Audit fields obligatorios

Toda tabla debe tener:
- `createdAt DateTime @default(now())`
- `updatedAt DateTime @updatedAt`

Y para tablas de negocio críticas (Users, Orders, Payments), considerar también:
- `deletedAt DateTime?` (soft delete en vez de borrado real)
- `createdBy String?` (quién creó el registro)

#### 5. Índices manuales

Prisma crea automáticamente índices en:
- Foreign keys.
- Campos `@unique`.
- Primary keys.

Hay que añadir manualmente con `@@index` en:
- Campos usados en `WHERE` frecuentes (`status`, `type`, `category`).
- Campos usados en `ORDER BY` frecuentes (`createdAt DESC`).
- Combinaciones de columnas si se filtran juntas (`@@index([userId, createdAt])`).

#### 6. Reglas de borrado en cascada

Para cada relación, decidir explícitamente:

```prisma
@relation(fields: [postId], references: [id], onDelete: Cascade)
@relation(fields: [postId], references: [id], onDelete: Restrict)
@relation(fields: [postId], references: [id], onDelete: SetNull)
```

- **Cascade**: si se borra el padre, los hijos también. Útil cuando los hijos no tienen sentido sin el padre (Comment → Post).
- **Restrict**: no permite borrar el padre si tiene hijos. Útil para integridad referencial fuerte (User → Order).
- **SetNull**: si se borra el padre, el hijo queda con NULL. Útil cuando el hijo puede existir solo (Author → Article).

Si no se documenta, Prisma usa Restrict por defecto. Mejor hacerlo explícito.

### Setup obligatorio de tests con BD

#### Tests unitarios (mocks)

Para lógica pura sin BD real. Mock del cliente de BD según el ORM. Ver patrones en `tests-skill`.

#### Tests de integración (Testcontainers)

**Estándar de facto en 2026**. Levanta una BD real en Docker durante los tests, con paridad total a producción.

Ejemplo TypeScript + Jest + Postgres:

```ts
// vitest.globalSetup.ts (también vale para Jest)
import { PostgreSqlContainer } from '@testcontainers/postgresql';

export async function setup() {
  const c = await new PostgreSqlContainer('postgres:17-alpine').start();
  process.env.DATABASE_URL = c.getConnectionUri();
  // ejecutar migraciones aquí
}
```

Para PHP, Python y otros stacks existen equivalentes (`@testcontainers/python`, etc.).

**Cuándo usar mocks vs Testcontainers:**
- Lógica de validación pura → mocks (rápido).
- Lógica que depende de queries reales → Testcontainers (paridad).
- CI/CD → Testcontainers (no romper en producción).

#### Reset entre tests

Más rápido que `TRUNCATE`: usar transacción + `ROLLBACK` al final de cada test. ~10× más rápido.

### Documentación obligatoria

Generar `docs/database-strategy.md` con:
- Motor elegido y razón.
- ORM elegido y razón.
- Lista de campos PII y cómo se protegen.
- Convención de audit fields.
- Estrategia de migraciones (zero-downtime, Expand-Contract).
- Estrategia de tests (mocks + Testcontainers).
- Configuración de auditoría futura (`pg_stat_statements` activado).

### Configuración inicial recomendada

Para Postgres, en `postgresql.conf` (o equivalente del proveedor cloud):

```
shared_preload_libraries = 'pg_stat_statements'
```

Y en la BD:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Esto permite que en el futuro se pueda analizar qué queries son lentas. Sin esto activado desde el principio, no hay datos cuando se necesiten.

---

## ESCENARIO 2: Proyecto existente — Auditoría

Cuando se invoca `/revisar-bd` (o se llega aquí desde `/regularizar` opción [4]) sobre un proyecto con BD ya construida.

### Las 4 auditorías

#### Auditoría 1: Diseño del modelo

Detecta problemas estructurales:

- **Foreign keys sin índice**: aunque Prisma las indexe, en SQL plano puede haber FKs sin índice → JOINs lentos.
- **Campos sin tipo apropiado**: salarios como `Float` en vez de `Decimal`, fechas como `String`.
- **Tablas sin primary key**: error grave.
- **Campos NOT NULL sin default**: causa problemas en migraciones.
- **Campos en 1FN no respetada**: listas separadas por comas, JSON dentro de texto.
- **Redundancias detectables**: misma información en varias tablas.
- **Falta de audit fields**: tablas sin `createdAt/updatedAt`.

Output: lista clasificada por severidad (alta / media / baja) y recomendación de corrección.

#### Auditoría 2: Seguridad e integridad

- **PII en texto plano**: emails, teléfonos, direcciones sin cifrar.
- **Restricciones débiles**: `String` sin longitud máxima en campos críticos.
- **FKs no indexadas**: causa de queries lentas.
- **Falta de RLS** (Row Level Security): tablas multi-tenant sin policies de aislamiento.
- **Audit logging ausente**: no hay forma de saber quién modificó qué y cuándo.
- **Secrets en BD**: contraseñas en plano, API keys sin cifrar.

Output: documento `docs/investigacion_seguridad.md` con tabla de findings clasificados por severidad, impacto y probabilidad.

#### Auditoría 3: Rendimiento de queries

Si hay acceso a `pg_stat_statements` (vía MCP o consulta directa):
- **N+1 queries**: cargar N entidades y hacer otra query por cada una.
- **Queries en bucles**: ejecutar SELECT dentro de un `forEach`.
- **Consultas innecesarias**: cargar 100 columnas cuando solo se usan 2.
- **Falta de caching**: queries idénticas repetidas en la misma sesión.

Si NO hay acceso a logs reales: análisis estático del código del backend buscando patrones problemáticos.

Output: las 3-5 mejoras de mayor impacto, priorizadas. Para cada una: el código actual, el código mejorado y la mejora estimada.

#### Auditoría 4: Calidad de datos

Generar tests tipo dbt o SQL checks que verifiquen invariantes de datos:

- **Valores nulos en campos NOT NULL**: si los hay, hay un bug.
- **Duplicados en campos UNIQUE**: si los hay, hay un bug.
- **Rangos inválidos**: edades negativas, salarios fuera de rango razonable, fechas en el futuro cuando no debería.
- **Referencias huérfanas**: foreign keys apuntando a IDs inexistentes (debería ser imposible con FKs bien puestas, pero a veces ocurre).
- **Inconsistencias de dominio**: estados que no existen en el catálogo, formatos de email malformados.

Output: archivo `tests/data-quality/` (o equivalente del stack) con queries SQL que se pueden ejecutar periódicamente. Cada query devuelve los registros que violan la regla.

### Filosofía de la auditoría

> **La IA detecta. El humano decide. El humano repara.**

`/revisar-bd` NUNCA modifica código ni BD. Solo genera reportes. Es decisión del usuario qué arreglar y en qué orden. Para arreglar usa `/reparar` ticket por ticket.

### Output estándar

Tras ejecutar las 4 auditorías, generar:

1. **`docs/database-audit.md`**: informe ejecutivo con resumen de hallazgos por categoría y severidad.
2. **`docs/investigacion_seguridad.md`** (si hay hallazgos de seguridad): detalle de problemas con clasificación.
3. **Tickets en `roadmap.md`**: uno por cada problema crítico o de alta severidad. Marcados como `[ ]` para que el usuario decida cuándo abordarlos.
4. **Tests de calidad de datos** (si aplica): scripts SQL en `tests/data-quality/`.

---

## Patrones por ORM

### Prisma (Node.js / TypeScript)

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique @db.VarChar(255)
  name      String   @db.VarChar(100)
  status    String   @default("active") @db.VarChar(20)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  posts     Post[]

  @@index([status])
}
```

### Drizzle (Node.js / TypeScript)

```ts
export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  email: varchar('email', { length: 255 }).unique().notNull(),
  name: varchar('name', { length: 100 }).notNull(),
  status: varchar('status', { length: 20 }).default('active'),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
}, (t) => [
  index('users_status_idx').on(t.status),
]);
```

### Eloquent (PHP / Laravel)

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('email', 255)->unique();
    $table->string('name', 100);
    $table->string('status', 20)->default('active')->index();
    $table->timestamps();
});
```

---

## Antipatrones a evitar

- **No usar transacciones para operaciones críticas**: si dos `INSERT` deben ir juntos, deben estar en transacción. Sin esto, si falla el segundo te quedas con datos inconsistentes.
- **`SELECT *` en código de producción**: trae más datos de lo necesario, rompe si cambia el schema.
- **Construir SQL con concatenación de strings**: SQL injection. Usar siempre queries parametrizadas o el ORM.
- **Migración destructiva sin Expand-Contract**: borrar columna directamente en producción → caída si algún cliente la usa.
- **Tests que comparten estado**: si un test crea un User con id=1 y otro test asume que ese User existe, los tests pasan en orden A pero fallan en orden B. Aislamiento mediante transacción + ROLLBACK.
- **Cifrado a mano**: no implementar tu propio AES. Usar la extensión del motor (`pgcrypto`) o librerías auditadas.
- **Borrado físico cuando debería ser lógico**: si una entidad tiene relaciones históricas (Order → User), borrar el User huérfana las Orders. Soft delete es la solución.

---

## Regla de oro final

> **En proyectos nuevos: diseñar antes de escribir.**
> **En proyectos existentes: auditar antes de cambiar.**

Saltarse cualquiera de los dos genera bugs caros que aparecen meses después.
