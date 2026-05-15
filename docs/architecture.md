<!-- TEMPLATE-PLACEHOLDER -->
# Arquitectura del Sistema (Architect-Brain v4.3 — Cursor Edition) — Estándar Universal

> Meta-documentación del scaffolding `junior-doc-gen-tests-cursor`. Describe cómo está construido el propio sistema, no el proyecto que lo use. Para el SSOT del proyecto concreto, ver el resto de archivos en `/docs/`.

---

## 1. Capa de Intención (Intent)
*Define el propósito y el flujo de comunicación de alto nivel.*
- **Actor:** Usuario (Product Owner / Developer) + IA (Arquitecto Senior).
- **Objetivo:** Traducir ideas o código existente en especificaciones técnicas claras manteniendo documentación, código, tests y decisiones sincronizados.
- **Interfaz:** Cursor + Agent operando bajo la rule `.cursor/rules/architect-brain.mdc` (cargada automáticamente con `alwaysApply: true`).
- **Cerebro:** `@prompts/architect-brain.md` — orquestador central de protocolos `On(...)`.

---

## 2. Capa de Implementación (Implementation)
*Define las reglas de construcción y la ubicación de la verdad.*

- **Single Source of Truth (SSOT):** Carpeta `/docs/` en la raíz del proyecto. Ninguna documentación oficial vive fuera de aquí. La memoria auto-generada del IDE NO es SSOT.
- **Flujo de Trabajo estándar:**
    1. Identificar necesidad/error o invocar un slash command.
    2. Sincronizar `user-stories.md` y `roadmap.md` (estado `[SYNC_PENDING]`).
    3. Si la tarea implica más de 2 archivos, activar `On(implementation_phase)` con paradas obligatorias entre pasos.
    4. Ejecutar código basándose en `blueprints.md`.
    5. Cerrar con `On(task_complete)` (checklist visible obligatorio).
- **Estándares Técnicos (Blueprints):** referencia obligatoria para conexiones a BD, patrones UI, librerías y convenciones de código del proyecto.

---

## 3. Capa de Memoria (Skills & Knowledge)
*Define cómo la IA y el usuario almacenan reglas reutilizables sin ensuciar la arquitectura global.*

Cursor unifica `rules` y `commands` bajo el concepto de **skills**. Hay tres tipos:

### Skills tipo slash command (slash commands clásicos)
Distribuidas en `.cursor/skills/<nombre>/SKILL.md` con `disable-model-invocation: true`. Solo se invocan manualmente con `/` en el chat:
- `inicio`, `regularizar`, `hoy`, `nueva`, `reparar`, `decidir`, `sync`, `cerrar`, `cuestionar`, `revisar-bd`, **`onboarding`** *(nuevo en v4.3)*.

### Skills auxiliares (auto-invocables)
Distribuidas en `.cursor/skills/<nombre>/SKILL.md` sin `disable-model-invocation`. Cursor las carga automáticamente cuando detecta que aplican:

- **`tests-skill`** — convenciones de testing, patrones de mock por stack, antipatrones.
- **`legacy-testing-skill`** — protocolo de characterization tests para código legacy sin red de seguridad.
- **`database-skill`** — auditorías de modelo, seguridad, rendimiento y calidad de datos.
- **`prompt-skill`** *(nuevo en v4.3)* — manual del **usuario** para redactar prompts profesionales con el patrón **RACEO** (Role + Objective + Context + Constraints + Expected Output).

### Skills dinámicas (las genera la IA durante el proyecto)
Cuando aparece lógica compleja específica del dominio (reglas fiscales, integraciones SFTP, flujos de pago), el Agent crea archivos nuevos en `.cursor/skills/` para que el conocimiento persista entre sesiones.

---

## 4. Ciclo de Vida del Sistema (Spec-kit Lifecycle)
*Estados por los que pasa cualquier tarea o inicio de proyecto.*

- **[SCANNING]** — análisis inicial de la raíz para detectar modo (proyecto nuevo, proyecto existente con docs, proyecto existente sin docs, codebase ajeno).
- **[ONBOARDING]** *(nuevo en v4.3)* — mapa rápido de codebase ajeno sin generar SSOT (skill `onboarding`). Solo lectura.
- **[SYNC_PENDING]** — pausa obligatoria de seguridad para actualizar documentos antes de tocar código.
- **[ARCHITECTING]** — redacción de requisitos y plan en `/docs/`.
- **[IMPLEMENTING]** *(formalizado en v4.3)* — ejecución del plan paso a paso bajo `On(implementation_phase)`. Una modificación = un paso. Paradas obligatorias entre pasos para revisión humana.
- **[TESTING]** — ejecución de la suite y registro del output. Si hay rojos, se vuelve a `[IMPLEMENTING]` o `[ARCHITECTING]` según corresponda.
- **[CLOSING]** — disparo obligatorio de `On(task_complete)` con checklist visible de sincronización documental.
- **[DONE]** — ticket marcado `[x]` en roadmap, tests en verde, todos los docs alineados.

---

## 5. Protocolos centrales (definidos en `architect-brain.md`)

| Protocolo | Disparador típico | Output |
|---|---|---|
| `On(project_start)` | `/inicio` | Entrevista BMADT + SSOT inicial |
| `On(testing_setup)` | tras BMADT | Framework de tests + linter-friendly + smoke test |
| `On(database_setup)` | tras testing si hay BD | `database-strategy.md` + Testcontainers + ADR |
| `On(database_audit)` | `/revisar-bd` | Auditoría de 4 dimensiones, sin modificar nada |
| `On(implementation_phase)` *(v4.3)* | "implementa el plan" | Pasos atómicos con paradas obligatorias |
| `On(task_complete)` | tras cada tarea | Checklist visible de sincronización documental |
| `On(sync_check)` | `/sync` | Reporte de drift entre código y docs |
| `On(resume_project)` | `/hoy` | Briefing en 10 líneas con sugerencia del día |
| `On(revisit_decision)` | `/decidir` | Nueva ADR + propagación a docs afectados |

---

## 6. Garantías del sistema

- **Antidrift:** ninguna tarea se cierra sin checklist de `On(task_complete)` visible.
- **Trazabilidad:** toda decisión no trivial vive en `decisions-log.md` como ADR inmutable.
- **Tests primero:** ningún ticket se marca `[x]` sin tests en verde.
- **Implementación disciplinada:** ningún plan multi-archivo se ejecuta de un tirón — siempre paso a paso con revisión humana.
- **Cobertura ≠ calidad:** la calidad real de los tests se audita con mutation thinking (`/cuestionar`).
- **BD bajo auditoría:** ningún cambio destructivo en BD sin `/revisar-bd` previo.
- **Prompts profesionales:** el usuario consulta `prompt-skill` para tareas complejas; el patrón RACEO reduce alucinaciones.
