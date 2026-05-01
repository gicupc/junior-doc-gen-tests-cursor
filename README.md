# junior-doc-gen-tests-cursor

Sistema de scaffolding para [Cursor](https://cursor.com) que convierte al Agent en un arquitecto senior disciplinado. Mantiene **documentación**, **código**, **tests**, **decisiones** y **base de datos** sincronizados durante todo el ciclo de vida del proyecto, evitando el clásico *documentation drift*.

> **Versión Cursor** del sistema. Existe también la [versión Windsurf](https://github.com/gicupc/junior-doc-gen-tests).

---

## Qué resuelve

Cuando trabajas con una IA generando código día tras día, aparecen seis problemas silenciosos:

1. **Los docs se quedan atrás del código** — nadie los actualiza porque no duele inmediatamente.
2. **Las decisiones técnicas se olvidan** — ¿por qué elegimos esta librería? Nadie se acuerda tres meses después.
3. **Los tests son un "luego" eterno** — se añaden al final, si sobra tiempo, con calidad baja.
4. **El código legacy se toca sin red de seguridad** — cambiar primero y rezar después es receta para bugs nuevos.
5. **Cobertura ≠ calidad** — tests que pasan no garantizan detectar bugs reales.
6. **Las bases de datos crecen sin auditoría** — índices faltantes, datos personales sin proteger, queries lentas que nadie detecta.

Este sistema impone protocolos que obligan a la IA a resolver los seis problemas por diseño.

---

## Cómo se usa en 30 segundos

1. Copia esta estructura completa (`.cursor/`, `docs/`, `prompts/`, `AGENTS.md`, `inicio-chat.txt`) en la raíz de tu proyecto.
2. Abre Cursor apuntando a la raíz del proyecto.
3. En el chat de Agent, escribe `/` y te aparecerá un menú con los comandos disponibles.
4. Elige el comando que toque según la situación (ver tabla abajo).

La rule `architect-brain.mdc` se carga automáticamente con `alwaysApply: true`, así que las reglas globales del sistema están presentes en cada chat sin necesidad de invocarlas.

---

## Slash commands disponibles

| Comando         | Cuándo usarlo                                                                 |
|-----------------|-------------------------------------------------------------------------------|
| `/inicio`       | Proyecto vacío. Entrevista BMADT + docs + setup de tests + setup de BD si aplica. |
| `/regularizar`  | Proyecto con código pero sin docs. Menú [1][2][3][4]. La opción [4] incluye auditoría de BD. |
| `/hoy`          | "¿Qué hacemos hoy?" Briefing con sugerencia concreta para el día.             |
| `/nueva`        | Añadir funcionalidad. Activa la interrupción de seguridad antes de tocar código. |
| `/reparar`      | Corregir un bug. Si no está cubierto, primero escribe el test que lo reproduce. |
| `/decidir`      | Cambiar una decisión pasada. Propaga cambios sin perder histórico.            |
| `/sync`         | Detectar drift entre código y documentación. Solo reporta.                    |
| `/cerrar`       | Forzar checklist de sincronización si sospechas que se saltó.                 |
| `/cuestionar`   | Auditar la calidad real de los tests con mutation thinking.                   |
| `/revisar-bd`   | Auditar la BD: modelo, seguridad, rendimiento, calidad de datos.              |

---

## Cómo funciona en Cursor

### Rules y skills (la diferencia con Windsurf)

Cursor unifica `rules` y `commands` bajo el concepto de **skills**. Esta versión aprovecha esa arquitectura:

- **`.cursor/rules/architect-brain.mdc`** con `alwaysApply: true` → reglas globales cargadas en cada chat.
- **`.cursor/skills/<nombre>/SKILL.md`** con `disable-model-invocation: true` → comportan como slash commands clásicos. Solo se invocan manualmente con `/`.
- **`.cursor/skills/<auxiliar>/SKILL.md`** sin esa línea → la IA las carga automáticamente cuando detecta que aplican (por ejemplo, `database-skill` cuando trabajas con BD).

### MCP de base de datos

El archivo `.cursor/mcp.json` incluye una plantilla para conectar Cursor a una BD Postgres local. Sustituye la connection string por la tuya y reinicia Cursor.

**Regla crítica de seguridad**: en producción, los MCPs de BD deben ir en modo **read-only**. Para desarrollo local con datos de prueba, full-access es aceptable.

---

## Qué hace el sistema (los 7 pilares)

### 1. Entrevista inicial estructurada (BMADT)

Al arrancar un proyecto nuevo con `/inicio`, la IA te hace 5 preguntas:
- **B (Beneficio):** objetivo y valor del proyecto.
- **M (Mecanismo):** stack técnico e integraciones.
- **A (Alcance):** módulos del MVP.
- **D (Datos):** entidades y flujos críticos.
- **T (Testing):** nivel de cobertura deseado (básico/estándar/exhaustivo).

Con las respuestas genera automáticamente los docs del SSOT en `/docs/` y **no escribe código** hasta que apruebes.

### 2. Setup automático del entorno de tests

Según el stack detectado en la respuesta M, instala el framework adecuado con su soporte de linter.

### 3. Setup automático de base de datos

Si la respuesta M o D del BMADT implica BD, el sistema dispara automáticamente `On(database_setup)`: detecta motor + ORM, identifica campos PII, genera `database-strategy.md`, configura auditoría futura, prepara Testcontainers.

### 4. Trazabilidad de decisiones (ADRs)

Todas las decisiones no triviales se registran en `decisions-log.md` como **Architecture Decision Records** inmutables. Si cambias de opinión, se añade una nueva ADR y la antigua queda marcada como `Superseded por ADR-XXX`.

### 5. Antidrift documental

Al cerrar **cualquier tarea** de código, la IA muestra un checklist visible marcando cada uno de los docs del SSOT como actualizado o "sin cambios" con justificación. Una tarea no está cerrada si el checklist no aparece.

### 6. Auditorías de calidad

Tres niveles de auditoría disponibles:
- **`/sync`** — detecta drift entre código y docs.
- **`/cuestionar`** — audita calidad de los tests con mutation thinking.
- **`/revisar-bd`** — audita la BD en 4 categorías: modelo, seguridad, rendimiento, calidad de datos.

Filosofía común: **la IA detecta, el humano decide, el humano repara**.

### 7. Retomar proyectos tras pausas

`/hoy` lee los docs clave, ejecuta un mini health-check y devuelve un briefing de 10 líneas:

```
📋 Proyecto: [Nombre]
🎯 Objetivo: [1 frase del PRD]
📈 Progreso: [X/Y tickets — Z%]
✅ Último cerrado: [título]
👉 Siguiente: [título]
🧪 Tests: [N passing / M skipped / K failing]
💡 Últimas decisiones: [2-3 ADRs]
⚠️ Warnings: [drift detectado o "ninguno"]
🚀 Sugerencia: [acción concreta para hoy]
```

---

## Estructura de archivos

```
/
├── AGENTS.md                        # Resumen para agentes (al estilo Cursor)
├── README.md                        # Este archivo
├── LICENSE                          # MIT
├── inicio-chat.txt                  # Manual operativo completo
├── .cursor/
│   ├── mcp.json                     # Plantilla de configuración MCP
│   ├── rules/
│   │   └── architect-brain.mdc      # Rule global con alwaysApply: true
│   └── skills/                      # 13 skills (10 commands + 3 auxiliares)
│       ├── inicio/SKILL.md
│       ├── regularizar/SKILL.md
│       ├── hoy/SKILL.md
│       ├── nueva/SKILL.md
│       ├── reparar/SKILL.md
│       ├── decidir/SKILL.md
│       ├── sync/SKILL.md
│       ├── cerrar/SKILL.md
│       ├── cuestionar/SKILL.md
│       ├── revisar-bd/SKILL.md
│       ├── tests-skill/SKILL.md           # auto-invocable
│       ├── legacy-testing-skill/SKILL.md  # auto-invocable
│       └── database-skill/SKILL.md        # auto-invocable
├── docs/                            # SSOT - plantillas
│   ├── prd.md
│   ├── architecture.md
│   ├── blueprints.md
│   ├── user-stories.md
│   ├── roadmap.md
│   ├── testing-strategy.md
│   ├── database-strategy.md
│   └── decisions-log.md
└── prompts/
    └── architect-brain.md           # Cerebro completo del sistema
```

---

## Reglas de oro

1. **Centralización** — toda documentación vive en `/docs/`. Nada de subcarpetas `docs` internas.
2. **Fuente de verdad** — los docs ganan sobre la memoria auto-generada del IDE.
3. **Sincronización** — ticket terminado = `[x]` en roadmap + tests en verde + checklist visible.
4. **No proliferación** — no crear archivos nuevos para cambios individuales; editar los maestros.
5. **Tests como contrato** — un `.skip` durante más de una semana es un bug del roadmap.
6. **Decisiones trazables** — si no está en `decisions-log.md`, no existe como decisión oficial.
7. **Antidrift** — ninguna tarea se cierra sin el checklist de sincronización documental.
8. **Legacy primero congelar, luego cambiar** — en código legacy, nunca modificar sin haber capturado antes el comportamiento actual.
9. **Cobertura ≠ calidad** — un test que pasa no garantiza detectar bugs.
10. **BD primero auditar, luego cambiar** — en proyectos con BD, ejecutar `/revisar-bd` antes de cambios destructivos.

---

## Sobre este sistema

Construido como adaptación a Cursor del sistema [`junior-doc-gen-tests`](https://github.com/gicupc/junior-doc-gen-tests) (versión original para Windsurf).

Las dos versiones comparten la misma filosofía y los mismos protocolos. La diferencia es la sintaxis de configuración del IDE:

| Concepto | Windsurf | Cursor |
|---|---|---|
| Reglas globales | `.windsurfrules` | `.cursor/rules/*.mdc` con `alwaysApply: true` |
| Slash commands | `.windsurf/workflows/*.md` con frontmatter | `.cursor/skills/*/SKILL.md` con `disable-model-invocation: true` |
| Skills auxiliares | `prompts/skills/*.md` | `.cursor/skills/*/SKILL.md` (auto-invocables) |
| MCP servers | `~/.codeium/windsurf/mcp_config.json` | `.cursor/mcp.json` o `~/.cursor/mcp.json` |

---

## Licencia

MIT
