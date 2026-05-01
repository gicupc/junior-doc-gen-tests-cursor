---
name: revisar-bd
description: Auditar la base de datos del proyecto en 4 categorias - modelo, seguridad, rendimiento, calidad de datos. Genera reporte ejecutivo y crea tickets en roadmap por hallazgos criticos. NO modifica BD ni codigo. La IA detecta, el humano decide.
disable-model-invocation: true
---

# /revisar-bd — Auditoria de base de datos

Este comando NO modifica codigo ni base de datos. Examina la estructura, queries y datos para detectar problemas. Genera un reporte clasificado por severidad para que tu decidas que arreglar.

La filosofia: **la IA detecta. El humano decide. El humano repara.**

---

## Cuando usar este comando

- Heredas un proyecto con BD y quieres saber su estado.
- Llevas tiempo desarrollando y sospechas que la BD tiene problemas (queries lentas, datos inconsistentes, falta de indices).
- Antes de un release importante, como auditoria preventiva.
- Tras `/regularizar` opcion [4], se ejecuta automaticamente si hay BD.
- En cualquier momento que quieras una segunda opinion sobre tu modelo de datos.

---

## Requisitos previos

- El proyecto debe tener BD (verificable en `package.json`/`composer.json` o en la respuesta M del BMADT).
- Si tienes el MCP de BD configurado en Cursor (ej. `postgres-XXX`), aprovechalo: la auditoria sera mucho mas profunda al inspeccionar BD real.
- Si NO tienes MCP, la auditoria sera estatica (solo schema y codigo). Util pero limitada.

---

## Pasos del protocolo

### Paso 1 — Determinar el alcance

Pregunta al usuario:

"Voy a auditar la base de datos. Indica el alcance:
- [1] **Auditoria completa** — modelo + seguridad + queries + calidad de datos.
- [2] **Solo modelo de datos** — schema, tipos, indices, normalizacion.
- [3] **Solo seguridad** — PII, FKs, restricciones, RLS.
- [4] **Solo rendimiento** — queries problematicas, N+1, indices faltantes.
- [5] **Solo calidad de datos** — nulos, duplicados, rangos, integridad.

Por defecto, [1] auditoria completa."

### Paso 2 — Inventario inicial

Carga `@.cursor/skills/database-skill/SKILL.md` como guia.

Si hay MCP de BD configurado:
- Lista todas las tablas.
- Cuenta filas por tabla.
- Detecta el motor (Postgres, MySQL, SQLite, etc.).
- Verifica si `pg_stat_statements` esta activado (Postgres).

Si NO hay MCP:
- Lee el schema declarativo (`schema.prisma`, `schema.sql`, modelos Eloquent, etc.).
- Lee las migraciones existentes.
- Lee el codigo del backend para inferir queries.

Presenta un resumen breve antes de empezar la auditoria.

### Paso 3 — Auditoria 1: Modelo de datos

Detectar:
- Tablas sin primary key (CRITICO).
- Foreign keys sin indice (ALTA).
- Tipos de datos inadecuados (`Float` para dinero, `String` para fechas, etc.).
- Campos NOT NULL sin DEFAULT que dificultan migraciones futuras.
- Falta de audit fields (`createdAt`, `updatedAt`).
- Violaciones de 1FN (listas separadas por comas, JSON dentro de TEXT).
- Redundancias detectables (misma informacion en varias tablas).
- Cardinalidades cuestionables (1-1 que probablemente deberian ser N-1).
- Reglas de borrado en cascada faltantes o erroneas.

Output: tabla clasificada por severidad.

### Paso 4 — Auditoria 2: Seguridad e integridad

Detectar:
- **PII en texto plano**: campos email, telefono, direccion, DNI, fecha_nacimiento sin cifrar ni hash.
- **Restricciones debiles**: VARCHAR sin longitud maxima, campos string que aceptan cualquier valor sin CHECK.
- **FKs no indexadas**: causa de queries lentas.
- **Falta de RLS** (Row Level Security): tablas multi-tenant sin policies de aislamiento.
- **Audit logging ausente**: no hay forma de saber quien modifico que registro y cuando.
- **Secrets en BD**: passwords en plano, API keys sin cifrar.
- **Conexiones inseguras**: codigo que se conecta sin SSL.

Output: documento `docs/investigacion_seguridad.md` con tabla de findings clasificados por severidad, impacto y probabilidad.

### Paso 5 — Auditoria 3: Rendimiento de queries

Si hay acceso a `pg_stat_statements` (vivo via MCP):

```sql
SELECT
  substring(query, 1, 100) AS query_preview,
  calls,
  ROUND((total_exec_time / calls)::numeric, 2) AS avg_ms,
  ROUND(total_exec_time::numeric, 2) AS total_ms
FROM pg_stat_statements
WHERE query NOT LIKE 'EXPLAIN%' AND calls > 10
ORDER BY total_exec_time DESC LIMIT 10;
```

Para las top 10 queries, ejecutar `EXPLAIN ANALYZE` y proponer indices.

Si NO hay acceso a logs reales:
- Analisis estatico del codigo del backend.
- Detectar patrones problematicos:
  - **N+1**: bucle que ejecuta SELECT por cada iteracion.
  - **Queries en bucles**: `for { db.query(...) }`.
  - **`SELECT *`**: traer mas datos de lo necesario.
  - **Falta de `include`/`with`**: cargar entidades sin sus relaciones esperadas (potencial N+1).
  - **Sin caching de queries repetidas en la misma sesion**.

Output: las 3-5 mejoras de mayor impacto, priorizadas. Para cada una mostrar:
- El codigo actual.
- El codigo mejorado.
- La mejora estimada.

### Paso 6 — Auditoria 4: Calidad de datos

Generar SQL checks que verifiquen invariantes de datos:

- **Valores nulos en campos NOT NULL**: si los hay, hay un bug.
- **Duplicados en campos UNIQUE**: si los hay, hay un bug.
- **Rangos invalidos**: edades negativas, salarios fuera de rango razonable, fechas en el futuro cuando no deberia.
- **Referencias huerfanas**: foreign keys apuntando a IDs inexistentes.
- **Inconsistencias de dominio**: estados que no existen en el catalogo, formatos de email malformados.

Output: archivo `tests/data-quality/checks.sql` (o equivalente del stack) con queries SQL ejecutables periodicamente. Cada query devuelve los registros que violan la regla.

### Paso 7 — Generar el reporte ejecutivo

Crear `docs/database-audit.md` con esta estructura:

```
# Database Audit Report - [Fecha]

## Resumen ejecutivo
- Hallazgos criticos: X
- Hallazgos altos: Y
- Hallazgos medios: Z
- Hallazgos bajos: W

## Categoria 1 - Modelo de datos
[Tabla con findings]

## Categoria 2 - Seguridad
[Resumen + referencia a docs/investigacion_seguridad.md]

## Categoria 3 - Rendimiento
[Top 3-5 queries problematicas con propuestas]

## Categoria 4 - Calidad de datos
[Resumen + referencia a tests/data-quality/]

## Recomendaciones priorizadas
1. [Hallazgo critico mas urgente] - ticket: ROADMAP-XXX
2. [Siguiente] - ticket: ROADMAP-XXX
...
```

### Paso 8 — Crear tickets en el roadmap

Por cada hallazgo critico o de severidad alta:
- Anadir un ticket nuevo en `roadmap.md` marcado como `[ ]`.
- Linkear el ticket al hallazgo en el reporte.

NO arreglar nada automaticamente. El usuario decide.

### Paso 9 — Sugerir siguiente paso

Pregunta al usuario:

"Auditoria completada. Hay X hallazgos criticos y Y de severidad alta.

¿Que prefieres ahora?
- [1] Empezar a reparar el hallazgo mas critico (uso `/reparar` para cada uno).
- [2] Revisar primero el reporte completo (`docs/database-audit.md`).
- [3] Cerrar la auditoria sin acciones inmediatas (tickets quedan en el roadmap)."

### Paso 10 — Cierre

Ejecuta `On(task_complete)` con el checklist de sincronizacion documental.

---

## Reparacion de hallazgos

Cuando el usuario decida arreglar un hallazgo:

1. **NUNCA arreglar varios a la vez.** Un hallazgo a la vez para mantener trazabilidad.
2. Usar `/reparar` con el codigo del ticket: el flujo de reparacion ya cubre el patron correcto (test rojo → fix → test verde → checklist).
3. Si la reparacion implica una migracion destructiva, **forzar el patron Expand-Contract** (ver `database-skill`).
4. Tras cada hallazgo arreglado, marcar el ticket como `[x]` en el roadmap.

---

## Que NO hace este comando

- **No modifica el schema** ni ejecuta migraciones.
- **No modifica datos** de la BD.
- **No instala dependencias** nuevas.
- **No crea archivos** de codigo, solo documentacion y SQL checks (read-only sobre el codigo).

Es un comando de **inteligencia, no de accion**.

---

## Regla de oro

> **Auditar antes de cambiar. Documentar antes de arreglar. Probar antes de desplegar.**

En proyectos con BD, saltarse la auditoria es la causa mas comun de bugs caros que aparecen meses despues.
