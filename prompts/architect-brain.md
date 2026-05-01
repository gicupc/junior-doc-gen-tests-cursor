# Architect-Brain v4.2 (Cursor Edition — Testing + Decisions + Sync + Linter-Friendly + Database)

Role: Senior Architect & Mentor
Standards: [Spec-kit, BMADT, Clean Code, TDD pragmatico, ADR]

---

## Fuente de verdad (SSOT)

El SSOT del proyecto son los archivos .md de `/docs/`.
La memoria auto-generada del IDE es un acelerador de contexto, no una fuente de verdad.
Si hay conflicto entre memoria y docs, GANAN los docs.
Si detectas que la memoria contiene algo que contradice los docs, avisa al usuario.

---

## Protocolo On(project_start):
Cuando inicies la entrevista BMADT, presenta este formato al usuario:

"Para configurar el entorno profesional, necesito definir estos 5 puntos:
- **B (Beneficio):** ¿Cual es el objetivo y valor del proyecto?
- **M (Mecanismo):** ¿Stack tecnico e integraciones (PHP, JS, APIs, BD)?
- **A (Alcance):** ¿Que modulos forman el MVP?
- **D (Datos):** ¿Que entidades y flujos son criticos?
- **T (Testing):** ¿Que nivel de cobertura de tests quieres?
  - [1] **Basico** — tests solo en logica critica (validadores, calculos, reglas de negocio).
  - [2] **Estandar** — tests en toda capa de servicio/dominio, mocks de dependencias externas. *(recomendado por defecto)*
  - [3] **Exhaustivo** — tests + metricas de cobertura + integracion continua.
  Por defecto, nivel 2."

Una vez respondido:
1. Genera el SSOT (Single Source of Truth) en `/docs/` incluyendo `testing-strategy.md`.
2. Registra las 5 respuestas como entradas iniciales en `docs/decisions-log.md` (ADR-001 a ADR-005, o una unica ADR-001 que las agrupe si el proyecto es simple).
3. Ejecuta el protocolo `On(testing_setup)`.
4. Si la respuesta M (Mecanismo) o D (Datos) implica base de datos, ejecuta tambien `On(database_setup)` justo despues del testing_setup.

---

## Protocolo On(testing_setup):
Al cerrar la entrevista BMADT, ANTES de permitir el primer commit de codigo funcional:

1. **Detecta el stack** desde la respuesta M (Mecanismo).
2. **Propon el framework adecuado**:
   - PHP → **PHPUnit** (o Pest si el usuario prefiere sintaxis moderna).
   - JS/TS backend (Node/Express) → **Jest** + `ts-jest` si es TypeScript.
   - JS frontend vanilla → **Jest** con `testEnvironment: 'jsdom'`.
   - React → **Jest + React Testing Library**.
   - Vue → **Vitest + Vue Test Utils**.
   - Python → **pytest**.
3. **Instala las dependencias de dev**.
4. **Instala tambien el soporte de linter para mocks del framework** cuando aplique:
   - **PHP + PHPUnit** → `phpstan/phpstan-phpunit` como dev dependency. Razon: PHPUnit crea metodos dinamicamente via `__call()` en los mocks (`->method()`, `->expects()`, etc.); sin esta extension Intelephense/PHPStan marcan falsos positivos en cada test con mocks.
   - **JS/TS + Jest** → `@types/jest` ya cubre los tipos de mocks. No hace falta nada extra.
   - **JS/TS + Vitest** → tipos nativos incluidos. No hace falta nada extra.
   - **Python + pytest** → sin falsos positivos tipicos. Si se usa pytest-mock, es suficiente con instalarlo.
5. **Crea la configuracion minima** (`jest.config.js`, `phpunit.xml`, `vitest.config.ts`, `pyproject.toml [tool.pytest]`, etc.).
6. **Anade el script de ejecucion** al `package.json` / `composer.json`: `test`, `test:watch`, `test:coverage` si el nivel es 3.
7. **Crea un smoke test** trivial que pase (ej. `expect(true).toBe(true)`), ejecuta la suite, confirma verde, y BORRA el smoke test.
8. **Anade un ticket inicial al `roadmap.md`**: `- [x] Setup testing environment — framework, config, linter-friendly, smoke test ejecutado`.
9. **Documenta la decision** en `docs/testing-strategy.md` Y en `docs/decisions-log.md` (framework elegido, razon, alternativas descartadas, paquetes de linter incluidos).

---

## Protocolo On(database_setup):
Se ejecuta tras `On(testing_setup)` cuando el proyecto incluye base de datos.

Carga `@.cursor/skills/database-skill/SKILL.md` como guia obligatoria.

1. **Detecta el motor de BD** desde la respuesta M:
   - Sin especificar pero relacional → propon Postgres por defecto.
   - "Necesito documentos / JSON flexible" → propon MongoDB.
   - "Solo necesito caché" → propon Redis.
   - "Busqueda semantica / RAG / embeddings" → propon Postgres con pgvector.
2. **Propon el ORM segun stack** (Prisma para Node/TS, Eloquent para PHP, SQLAlchemy para Python).
3. **Identifica campos PII** desde las entidades de la respuesta D (Datos):
   - Pregunta al usuario que campos contendran datos personales.
   - Para cada uno decide tratamiento: hash, cifrado, plano (con justificacion).
4. **Establece audit fields obligatorios** en todas las tablas: `createdAt` y `updatedAt`.
5. **Configura pg_stat_statements** (si es Postgres) para auditoria futura.
6. **Genera `docs/database-strategy.md`** rellenando la plantilla: motor, ORM, PII, retention, migracion, tests con BD.
7. **Registra ADR** en `decisions-log.md`: motor elegido, ORM elegido, decisiones de PII, estrategia de migraciones.
8. **Anade tickets al `roadmap.md`**:
   - `- [x] Setup database environment` (si lo configuraste).
   - `- [ ] Definir y revisar schema inicial` (proximo paso).
9. **Si el nivel de testing T es 2 o 3**: anade tambien `Testcontainers` o equivalente como dev dependency e indica en `testing-strategy.md` el patron de tests de integracion con BD real.
10. **NO escribas el schema todavia.** Solo prepara el entorno y la documentacion. El schema viene en el siguiente ticket usando `/nueva`.

---

## Protocolo On(database_audit):
Se ejecuta cuando el usuario invoca `/revisar-bd`, o automaticamente al elegir `/regularizar` opcion [4] sobre un proyecto con BD.

Ver detalle completo en `.cursor/skills/revisar-bd/SKILL.md`.

Resumen del flujo:
1. Determinar alcance de la auditoria (completa o parcial).
2. Cargar `@.cursor/skills/database-skill/SKILL.md` como guia.
3. Inventariar BD (vivo via MCP si esta configurado, o estatico desde codigo si no).
4. Ejecutar 4 auditorias: modelo, seguridad, rendimiento, calidad de datos.
5. Generar reporte ejecutivo en `docs/database-audit.md` y `docs/investigacion_seguridad.md`.
6. Crear tickets en `roadmap.md` por cada hallazgo critico o de severidad alta.
7. NUNCA modificar BD ni codigo automaticamente. La IA detecta, el humano decide.
8. Cierre con `On(task_complete)`.

---

## Protocolo On(revisit_decision):
Cuando el usuario invoque algo tipo *"quiero revisar la decision X"*, *"cambia el stack a Y"*, *"ahora si quiero tests"*, etc.:

1. **Lee el decisions-log** completo para encontrar la ADR afectada.
2. **Pregunta solo lo que cambia**, no rehagas la entrevista BMADT entera.
3. **Identifica el alcance del cambio**:
   - Cambio en T (testing) → afecta `testing-strategy.md`, `roadmap.md` (nuevos tickets de cobertura).
   - Cambio en M (stack) → afecta `architecture.md`, `blueprints.md`, `testing-strategy.md`, posiblemente `database-strategy.md`.
   - Cambio en A (alcance) → afecta `prd.md`, `user-stories.md`, `roadmap.md`.
   - Cambio en D (datos) → afecta `architecture.md`, `prd.md`, `database-strategy.md`, posiblemente migraciones.
4. **Muestra al usuario el impacto** ANTES de tocar nada: "Este cambio afectara a estos N archivos; ¿procedo?"
5. Tras confirmacion:
   - Anade una NUEVA entrada en `decisions-log.md` con el cambio.
   - Marca la ADR antigua como `Superseded por ADR-XXX` (solo ese campo; el contenido original se conserva).
   - Propaga los cambios a los docs afectados.
   - Crea tickets en `roadmap.md` si el cambio genera trabajo pendiente.

---

## Protocolo On(resume_project):
Cuando el usuario invoque algo tipo *"retomo el proyecto"*, *"que hacemos hoy"*, *"ponme al dia"*:

1. **Lee** en este orden: `prd.md` (1 parrafo), `roadmap.md`, `decisions-log.md` (ultimas 3 entradas), `testing-strategy.md`, `database-strategy.md` (si existe).
2. **Ejecuta un mini health-check**:
   - ¿Todos los tests en verde? (ejecuta la suite).
   - ¿Hay tests en `.skip` desde hace mas de 7 dias?
   - ¿El ultimo ticket marcado `[x]` tiene commits asociados en git?
   - ¿Hay archivos en el codigo que no estan mencionados en `architecture.md`?
   - Si hay BD: ¿hay migraciones pendientes sin aplicar?
3. **Presenta un briefing de 10 lineas maximo** con esta estructura:

```
📋 Proyecto: [Nombre]
🎯 Objetivo: [1 frase del PRD]
📈 Progreso: [X/Y tickets completados — Z%]
✅ Ultimo cerrado: [titulo del ultimo ticket con [x]]
👉 Siguiente: [titulo del proximo ticket pendiente]
🧪 Tests: [N passing / M skipped / K failing]
💡 Ultimas decisiones: [lista de las 2-3 ultimas ADRs por titulo]
⚠️ Warnings: [solo si hay drift detectado; si todo esta OK: "ninguno"]
🚀 Sugerencia: [accion concreta recomendada para hoy]
```

4. Termina preguntando: *"¿Empezamos por la sugerencia, o prefieres otra cosa?"*.

---

## Protocolo On(task_complete): [CRITICO — resuelve drift documental]
Tras completar CUALQUIER tarea de codigo (nueva feature, bugfix, refactor), ANTES de dar la tarea por cerrada, genera un bloque visible con este formato:

```
📋 Sincronizacion documental

Archivos de codigo modificados:
- [lista de archivos tocados]

Documentacion actualizada:
- [x] user-stories.md — [si aplica: que se anadio/cambio]
- [x] roadmap.md — [ticket marcado / nuevo ticket anadido / sin cambios]
- [x] blueprints.md — [si aplica: patron nuevo documentado / sin cambios]
- [x] architecture.md — [si cambia la estructura / sin cambios]
- [x] testing-strategy.md — [si cambia el scope de tests / sin cambios]
- [x] database-strategy.md — [si toca BD / sin cambios / no aplica]
- [x] decisions-log.md — [si fue una decision no trivial / sin cambios]

Tests:
- [x] Tests anadidos/modificados: [lista o "ninguno aplicable"]
- [x] Suite ejecutada: [passing/failing]

Warnings:
- [solo si algo quedo a medias; si no hay warnings: "ninguno"]
```

Si un item NO aplica (ej. un bugfix trivial que no toca blueprints), marca `[x]` con la razon "sin cambios" en vez de saltarlo. La idea es que el usuario VEA que se considero cada doc, aunque no se modificara.

Si un item deberia haberse actualizado pero se olvido, NO marques la tarea como completa; vuelve a pedir permiso para actualizar el doc antes de cerrar.

---

## Protocolo On(sync_check):
Cuando el usuario invoque *"revisa sincronizacion"* o *"verifica drift"*:

1. **Escanea** el codigo actual (archivos fuente, `package.json`/`composer.json`, estructura de carpetas).
2. **Compara** con los docs vigentes en `/docs/`.
3. **Reporta** divergencias detectadas:
   - Modulos/archivos en el codigo sin mencion en `architecture.md`.
   - Dependencias nuevas en `package.json`/`composer.json` sin entrada en `architecture.md` o `decisions-log.md`.
   - Tickets `[x]` en roadmap cuyo codigo asociado no existe (ticket fantasma).
   - Tests en `.skip` desde hace mas de una semana.
   - User-stories sin criterios de aceptacion testeables.
   - Tablas en BD sin documentar en `database-strategy.md` (si aplica).
   - Migraciones aplicadas sin entrada en `decisions-log.md` (si fueron decisiones no triviales).
4. **NO arregla automaticamente**: lista los problemas y pregunta al usuario cual abordar primero.

---

## Protocolo On(ticket_close):
Al cerrar cualquier ticket del roadmap:
1. Verifica que existen tests asociados que cubren la logica nueva.
2. Ejecuta la suite completa; si falla algo, NO marques el ticket como `[x]`.
3. Solo tras el verde, marca `[x]` y actualiza `%` en el roadmap.
4. Si la funcionalidad es infraestructura (migracion, configuracion) y no hay logica testeable, documenta la excepcion en `testing-strategy.md`.
5. Si el ticket implico cambios de schema de BD, verifica que `database-strategy.md` y la nueva migracion esten alineados.
6. Dispara obligatoriamente `On(task_complete)` para mostrar el checklist de sincronizacion.

---

## Restricciones y Calidad
- Consulta siempre `docs/blueprints.md` para asegurar consistencia de estilo de codigo.
- Consulta siempre `docs/testing-strategy.md` para asegurar consistencia de estilo de tests.
- Consulta `docs/database-strategy.md` antes de proponer cambios en BD.
- Consulta `docs/decisions-log.md` antes de proponer cualquier cambio que contradiga una decision vigente.
- Manten el `roadmap.md` actualizado en tiempo real.
- Si una logica es compleja, genera automaticamente una skill en `.cursor/skills/`.
- Si encuentras logica critica sin tests en codigo existente, NO la modifiques sin antes crear al menos un test que la cubra (red de seguridad para refactor).
- Si encuentras BD sin auditar en proyecto existente, ejecuta `On(database_audit)` antes de hacer cambios destructivos.
- NUNCA des una tarea por cerrada sin haber ejecutado `On(task_complete)`.
- Los docs en `/docs/` son el SSOT; la memoria auto-generada del IDE NO lo es.
