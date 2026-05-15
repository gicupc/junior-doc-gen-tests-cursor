<!-- TEMPLATE-PLACEHOLDER -->
# Database Strategy

> Este documento lo rellena el Architect-Brain durante el protocolo `On(database_setup)` cuando el proyecto incluye base de datos. No editar a mano sin avisar a la IA — es parte del SSOT.

---

## Decisiones del proyecto

### Motor de base de datos
*Ejemplo: PostgreSQL 17 / MongoDB 8 / Redis 7*

### Razón de la elección
*Ejemplo: PostgreSQL por integridad relacional fuerte, ecosistema maduro de extensiones (pgvector, pgaudit) y soporte nativo de JSONB para casos flexibles.*

### ORM elegido
*Ejemplo: Prisma 5.x*

### Razón de la elección del ORM
*Ejemplo: Prisma por type-safety en TypeScript, migraciones declarativas y excelente DX.*

---

## Modelo de datos

### Entidades principales
*Lista de tablas/colecciones del modelo, con una línea descriptiva cada una.*

- `User` — usuarios del sistema.
- `...`

### Diagrama de relaciones
*Insertar diagrama ERD aquí (formato Mermaid, imagen, o referencia a archivo separado).*

### Convención de nombres
- Tablas en singular, PascalCase: `User`, `Order`, `Product`.
- Columnas en camelCase: `createdAt`, `userEmail`.
- Foreign keys: `<tabla>Id` (ej. `userId`).
- Índices manuales nombrados: `<tabla>_<columna>_idx`.

---

## Audit fields obligatorios

Toda tabla del sistema debe incluir:
- `createdAt: DateTime` con default a `now()`.
- `updatedAt: DateTime` que se actualiza automáticamente.

Excepciones documentadas (tablas que NO necesitan audit):
- *[ej. tablas de configuración estática]*

---

## Datos personales (PII)

### Campos identificados como PII

| Tabla | Campo | Tipo | Tratamiento |
|---|---|---|---|
| User | email | String | Hash + búsqueda con índice cifrado |
| User | phone | String | Texto plano (no crítico para este MVP) |
| User | passwordHash | String | bcrypt (nunca password en plano) |

### Política de retención
*Ejemplo: los datos de usuarios eliminados se anonimizan tras 30 días.*

### Cifrado
*Ejemplo: cifrado at-rest gestionado por el proveedor (RDS/Supabase). Cifrado a nivel columna con `pgcrypto` para campos críticos.*

---

## Estrategia de migraciones

### Herramienta
*Ejemplo: `prisma migrate` para gestión declarativa.*

### Patrón para producción
**Expand-Contract** (zero-downtime), obligatorio para cambios destructivos:

1. **Expand**: añadir columna nueva (nullable), escribir en ambas (vieja y nueva).
2. **Backfill**: job por lotes para rellenar la nueva.
3. **Migrate reads**: cambiar lecturas a la nueva columna.
4. **Contract**: `DROP` columna antigua en migración separada.

### Reglas de oro
- Nunca `prisma migrate deploy` en producción sin haber leído el SQL generado.
- Nunca `DROP TABLE` o `DROP COLUMN` sin Expand-Contract.
- Toda migración pasa por revisión humana. La IA puede generarlas pero no aplicarlas en producción autónomamente.

---

## Estrategia de tests con BD

### Tests unitarios (lógica pura)
- Mock del cliente de BD según ORM.
- Sin BD real.
- Rápidos (<100ms por test).

### Tests de integración (lógica con BD)
- **Testcontainers**: BD real en Docker durante los tests.
- Imagen: *[ej. `postgres:17-alpine`]*
- Aislamiento: transacción + ROLLBACK al final de cada test.

### Configuración
*Referencia al archivo de setup de tests: `vitest.globalSetup.ts`, `tests/Setup.php`, etc.*

### Reset entre tests
**Patrón ROLLBACK**, no TRUNCATE. Es ~10× más rápido y garantiza aislamiento total.

---

## Configuración de auditoría futura

### Activado desde el inicio del proyecto

**Postgres** (si aplica):
```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pgaudit;
```

En `postgresql.conf`:
```
shared_preload_libraries = 'pg_stat_statements, pgaudit'
```

### Por qué activarlo desde día 1
Cuando llegue el momento de auditar queries lentas (vía `/revisar-bd`), no habrá datos históricos si no se activó al principio. Es como activar logging después de que pase el incidente.

---

## Seguridad

### Row Level Security (RLS)
*Si el sistema es multi-tenant, declarar políticas RLS desde el principio.*

Patrón Postgres + Supabase:
```sql
CREATE POLICY own_rows ON orders FOR SELECT TO authenticated
USING ((select auth.uid()) = user_id);
```

### SQL injection
- Nunca construir SQL concatenando strings.
- Usar siempre el ORM o queries parametrizadas.
- En Prisma: NO usar `$queryRawUnsafe`.

### Connection
- En producción: `sslmode=verify-full`.
- En desarrollo: `sslmode=disable` aceptable solo en local.

### Secrets
- `.env` NUNCA en git (verificar `.gitignore`).
- En producción: gestor de secretos (Doppler, Vault, AWS Secrets Manager).

---

## Configuración del MCP de la IA

*Si el proyecto usa Cursor y se quiere conectar la IA a la BD para asistencia avanzada (la configuración va en `.cursor/mcp.json` en la raíz del proyecto, o en `~/.cursor/mcp.json` si la quieres global).*

```json
{
  "mcpServers": {
    "postgres-[nombre-proyecto]": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "[CONNECTION_STRING]"
      ]
    }
  }
}
```

**REGLA OBLIGATORIA**: en producción, conectar el MCP en modo read-only. La IA NUNCA debe poder ejecutar `INSERT/UPDATE/DELETE/DDL` sobre BD de producción.

Para entornos locales y de desarrollo, modo completo es aceptable.

---

## Excepciones documentadas

*Si alguna decisión de las anteriores no se aplica, documentar aquí por qué.*

---

## Trabajo futuro

*Ideas que quedan fuera del MVP pero anotadas para más adelante.*

- Particionado por fecha en tabla X cuando supere Y filas.
- Replicas de lectura cuando el tráfico supere Z requests/segundo.
- Migración a vector store dedicado si pgvector no escala.
