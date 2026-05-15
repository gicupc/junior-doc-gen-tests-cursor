# Changelog v4.6 → v4.7.1 (Cursor Edition — Template Detection + Refinements)

Sesión de actualización: 15 de mayo de 2026

Origen: durante el arranque del proyecto final `quellevoyo` del bootcamp AI4Devs (con la versión Windsurf del scaffolding) se detectaron dos problemas reales:

1. **Falso positivo de "trabajo previo"** al copiar el scaffolding a un proyecto nuevo: como los templates en `/docs/` tienen contenido placeholder (`[NOMBRE_DEL_PROYECTO]`, etc.), al ejecutar `/inicio` el sistema los detectaba como "archivos con contenido", asumía trabajo previo y mostraba el prompt defensivo de CASO B en lugar de arrancar BMADT limpio.
2. **Falta de mecanismo anti-residual**: aunque el fix del problema 1 instruía al modelo a eliminar el marcador al rellenar cada doc, no había red de seguridad si el modelo se olvidaba. Un marcador residual hacía que el sistema, en futuras ejecuciones, volviera a tratar el proyecto como template sin rellenar — perdiendo el trabajo previo.

Esta release v4.7.1 introduce el marcador `<!-- TEMPLATE-PLACEHOLDER -->` como ciudadano de primera clase del scaffolding Y refuerza el sistema con verificación automática post-generación, todo en un solo bump.

**Nota de paridad de versiones**: en la versión Windsurf (`junior-doc-gen-tests`) estos dos cambios se publicaron secuencialmente como v4.7 (template marker, esta tarde) y v4.7.1 (refuerzo, este bump). En Cursor se combinan en un solo bump v4.7.1 para mantener paridad de número de versión final. El usuario que actualice desde v4.6 obtiene el mismo estado final que un usuario de Windsurf que aplique v4.7 y v4.7.1 secuencialmente.

---

## Resumen del cambio

El scaffolding pasa a marcar explícitamente cuándo un archivo de `/docs/` es un template sin rellenar y cuándo ya tiene contenido real. La detección + eliminación + verificación viven en tres lugares complementarios:

1. **Marcador en los 9 templates** de `/docs/` — `<!-- TEMPLATE-PLACEHOLDER -->` como primera línea.
2. **REGLA INMEDIATA en `.cursor/rules/architect-brain.mdc`** — detecta 3 casos al iniciar sesión: docs vacíos, docs con todos los marcadores (templates sin rellenar), docs con al menos uno sin marcador (trabajo previo).
3. **`On(project_start)` + `On(task_complete)`** instruyen al modelo a (a) eliminar el marcador al rellenar cada doc, y (b) verificar al cierre que no quedó ningún residual.

Ningún cambio rompe v4.6. v4.7.1 es estrictamente aditivo: si un usuario tiene v4.6 con docs ya rellenados (sin marcador), v4.7.1 no afecta a esos proyectos.

---

## Cambios

### 1. Modificado: los 9 templates de `/docs/`

**Estado:** cada archivo añade `<!-- TEMPLATE-PLACEHOLDER -->` como primera línea, sin alterar el resto del contenido.

Archivos modificados:
- `docs/prd.md`
- `docs/architecture.md`
- `docs/blueprints.md`
- `docs/user-stories.md`
- `docs/roadmap.md`
- `docs/testing-strategy.md`
- `docs/database-strategy.md`
- `docs/decisions-log.md`
- `docs/supply-chain-strategy.md`

Razón: el marcador es invisible en preview de markdown (comentario HTML) y se verifica con `head -1 docs/X.md`, lo que permite detección automática sin contaminar la lectura humana del template.

### 2. Modificado: `.cursor/rules/architect-brain.mdc`

**Cambios**:

- **REGLA INMEDIATA ampliada** (al inicio del archivo) — pasa de un solo caso ("si falta documentación, activa regularización o inicio") a tres casos explícitos:
  1. `/docs/` vacío o inexistente → `On(project_start)` (CASO A).
  2. `/docs/` poblado pero TODOS los `.md` tienen el marcador → templates sin rellenar, también activa `On(project_start)`.
  3. `/docs/` poblado y al menos uno NO tiene el marcador → hay trabajo previo, NO arranca `/inicio` automáticamente; ofrece menú CASO B (regularizar / evolucionar / reparar / regularizar+tests+BD+frontend).
- **Nota explicativa** añadida al final de la regla inmediata: "El marcador `<!-- TEMPLATE-PLACEHOLDER -->` se elimina de cada archivo al rellenarlo con contenido real. Una vez eliminado, el archivo deja de considerarse template."

- **Sección CIERRE DE TAREA OBLIGATORIO (ANTIDRIFT)** amplía con un nuevo párrafo al final:
  > **Verificacion anti-template-residual (v4.7.1)**: si la tarea creo o modifico algun archivo del SSOT en `/docs/`, antes de cerrar la tarea verifica que ningun archivo de `/docs/*.md` empiece con `<!-- TEMPLATE-PLACEHOLDER -->`. Equivalente a `head -1 docs/*.md | grep -c "TEMPLATE-PLACEHOLDER"` debe devolver `0`. Si encuentras algun archivo con marcador residual, eliminalo silenciosamente antes de mostrar el checklist final. Esta verificacion es obligatoria porque un marcador residual hace que el sistema vuelva a tratar el proyecto como "template sin rellenar" en futuras ejecuciones de `/inicio`, perdiendo el trabajo previo.

### 3. Modificado: `.cursor/skills/inicio/SKILL.md`

**Estado:** reescrito (mismo archivo, contenido reforzado, número de pasos pasa de 6 a 8).

Cambios:

- **Paso 1 nuevo — "Detecta el estado de `/docs/`"** antes de cualquier otra acción. Replica la lógica de los 3 casos de la REGLA INMEDIATA. Si encuentra trabajo previo (caso 3), muestra al usuario un aviso con 4 opciones: cancelar, `/onboarding`, `/sync`, reset destructivo. Espera respuesta antes de continuar.
- **Paso 4 (antes 3) ampliado** con la instrucción explícita: "al rellenar cada archivo, ELIMINA la primera linea `<!-- TEMPLATE-PLACEHOLDER -->`. Una vez eliminado el marcador, el archivo deja de considerarse template y queda como SSOT real del proyecto."
- **Paso 6 (antes 5) ampliado** con recordatorio en el bloque de `On(database_setup)`: "(recuerda eliminar el marcador TEMPLATE-PLACEHOLDER de la primera linea)".
- **Paso 7 nuevo — "VERIFICACION FINAL OBLIGATORIA — antiresiduo de templates"**. Antes de mostrar el resumen final, Cascade ejecuta una verificación automática: para cada archivo en `/docs/*.md`, comprueba que la primera línea NO sea `<!-- TEMPLATE-PLACEHOLDER -->`. Si encuentra alguno, lo repara y avisa al usuario en el resumen final con `⚠️ Reparado: marcador residual en docs/X.md eliminado tras la generación.`
- **Frase clave añadida**: "Esta verificacion NO es opcional. Un solo doc con marcador residual hace que el sistema vuelva a tratar el proyecto como 'template sin rellenar' la proxima vez que alguien ejecute /inicio, perdiendo el trabajo previo."

### 4. Modificado: `prompts/architect-brain.md`

**Cambios**:

- **Header bump**: `v4.6 (Cursor Edition — Supply Chain Security + ...)` → `v4.7.1 (Cursor Edition — Template Detection + Supply Chain Security + ...)`.

- **`On(project_start)` paso 1 ampliado** con dos sub-instrucciones:
  - "Si los archivos de `/docs/` ya existian con la primera linea `<!-- TEMPLATE-PLACEHOLDER -->` (templates del scaffolding), ELIMINA esa primera linea al rellenarlos. Un archivo con contenido real NO debe conservar el marcador."
  - "**VERIFICACION POST-GENERACION (no opcional)**: tras rellenar todos los docs, comprueba que ningun archivo de `/docs/*.md` empieza con `<!-- TEMPLATE-PLACEHOLDER -->`. Si encuentras alguno con marcador residual, eliminalo inmediatamente."

  Razón: las instrucciones también tienen que existir en el architect-brain porque es el protocolo que se aplica cuando el usuario invoca `On(project_start)` directamente (no solo desde `/inicio`), por ejemplo en proyectos donde alguien lanza la entrevista BMADT manualmente.

- **`On(task_complete)` checklist amplía con un bloque nuevo** entre "Documentacion actualizada" y "Tests":
  ```
  Integridad del scaffolding (anti-template-residual):
  - [x] Ningun archivo `/docs/*.md` empieza con `<!-- TEMPLATE-PLACEHOLDER -->` (verificado con `head -1 docs/*.md`). Si esta tarea creo o modifico algun doc del SSOT, esta verificacion es obligatoria.
  ```
  Razón: cualquier tarea que toque docs del SSOT (no solo el setup inicial) puede dejar marcadores residuales si el modelo recrea un doc desde el template. El check en `On(task_complete)` lo previene como rutina.

### 5. Modificado: `AGENTS.md`

- Versión `v4.6` → `v4.7.1` en la línea de cabecera.

### 6. Modificado: `CLAUDE.md`

- Versión `v4.6` → `v4.7.1` en la línea de cabecera.

### 7. Archivo nuevo: `CHANGELOG-v4.7.1.md`

Este archivo.

---

## Archivos NO modificados en este bump (sin cambios funcionales necesarios)

- `README.md` — la mención a versión está en el cuerpo del documento; bumpear es mantenimiento estético no crítico.
- `inicio-chat.txt` — manual operativo, similar al README.
- Resto de skills (`bdd`, `database-skill`, `frontend-skill`, etc.) — no afectadas por el cambio.
- `.cursor/mcp.json` — sin cambios.

Si el usuario quiere paridad textual de versión en el README y el inicio-chat.txt, puede bumpear a mano (búsqueda y reemplazo de `v4.6` por `v4.7.1` en cabeceras y secciones de versión). No es crítico para el funcionamiento del sistema.

---

## Migración

### Archivos modificados

- `docs/prd.md`
- `docs/architecture.md`
- `docs/blueprints.md`
- `docs/user-stories.md`
- `docs/roadmap.md`
- `docs/testing-strategy.md`
- `docs/database-strategy.md`
- `docs/decisions-log.md`
- `docs/supply-chain-strategy.md`
- `.cursor/rules/architect-brain.mdc`
- `.cursor/skills/inicio/SKILL.md`
- `prompts/architect-brain.md`
- `AGENTS.md`
- `CLAUDE.md`

### Archivos nuevos

- `CHANGELOG-v4.7.1.md`

### Comportamiento para proyectos existentes

- **Proyecto en v4.6 con docs ya rellenados (sin marcador)**: v4.7.1 no afecta nada. El sistema detectará "trabajo previo" en cada sesión (caso 3 de la REGLA INMEDIATA) y ofrecerá el menú CASO B si el usuario invoca `/inicio` por error.
- **Proyecto en v4.6 que usa el scaffolding para iniciar uno nuevo HOY**: actualizar a v4.7.1 antes de copiar el scaffolding. El proyecto nuevo arrancará limpio con BMADT al ejecutar `/inicio`.
- **Proyecto que se inició con scaffolding viejo v4.6 y dejó marcadores residuales (no debería pasar, pero por si acaso)**: el siguiente `/cerrar`, `/sync` o cualquier tarea que dispare `On(task_complete)` los detectará y eliminará silenciosamente.

---

## Verificación post-migración

Tras aplicar v4.7.1, comprobar:

1. **Templates con marcador**: `head -1 docs/*.md` debería mostrar `<!-- TEMPLATE-PLACEHOLDER -->` en las 9 líneas.
2. **REGLA INMEDIATA con 3 casos**: en `.cursor/rules/architect-brain.mdc`, las primeras 15 líneas tras `# PROTOCOLO ...` describen los 3 casos.
3. **`On(task_complete)` con verificación**: en `.cursor/rules/architect-brain.mdc`, buscar la cadena `Verificacion anti-template-residual (v4.7.1)`.
4. **Header de `prompts/architect-brain.md`**: empieza con `# Architect-Brain v4.7.1 (Cursor Edition — Template Detection + ...)`.
5. **`AGENTS.md` y `CLAUDE.md`**: línea de cabecera dice `Architect-Brain v4.7.1`.
6. **Smoke test funcional**: copiar el scaffolding a una carpeta nueva, ejecutar `/inicio` en Cursor, completar BMADT con respuestas mínimas, comprobar que el resumen final NO contiene `⚠️ Reparado: marcador residual...` (porque el sistema debería haberlo limpiado durante la generación). Si aparece `⚠️ Reparado:`, el modelo se olvidó del paso de eliminar el marcador al rellenar y la verificación final lo arregló — esto es ACEPTABLE (la red de seguridad funcionó) pero indica que el refuerzo de `On(project_start)` no fue suficiente.

---

## Paridad con Windsurf v4.7.1

Esta release tiene **paridad funcional total** con la versión Windsurf (`junior-doc-gen-tests`) v4.7.1, publicada en el mismo bump:

- Mismos 9 templates con marcador.
- Mismas 3 detecciones en REGLA INMEDIATA.
- Mismo paso de verificación final en `inicio`.
- Mismo refuerzo en `On(project_start)` y `On(task_complete)`.

Las únicas diferencias estructurales son las propias de cada motor:

- **Windsurf**: workflow en `.windsurf/workflows/inicio.md` con frontmatter `description:`, reglas globales en `.windsurfrules`, protocolos en `prompts/architect-brain.md`.
- **Cursor**: skill en `.cursor/skills/inicio/SKILL.md` con frontmatter `name:` + `description:` + `disable-model-invocation: true`, reglas globales en `.cursor/rules/architect-brain.mdc`, protocolos en `prompts/architect-brain.md` (igual nombre que Windsurf, contenido equivalente con paths adaptados a `.cursor/skills/`).

Adicionalmente, Cursor sostiene dos archivos auxiliares (`AGENTS.md` y `CLAUDE.md`) que Windsurf no tiene; ambos reciben el bump de versión.

---

## Lo que NO entró en v4.7.1 (intencionado)

- **Separación de `docs/architecture.md` en `SYSTEM-INFO.md` (meta-doc del scaffolding) + `architecture.md` (template real)**. Detectado durante la sesión de quellevoyo: el `docs/architecture.md` actual contiene meta-documentación del propio sistema Architect-Brain, no es un template puro. Esto puede confundir a Cascade/Cursor al rellenarlo. Apuntado como ítem para v4.8.
- **Hook git pre-commit que verifica marcadores residuales**. Considerado pero descartado: añade dependencia de git hooks (no todos los usuarios los activan), y la verificación en `On(task_complete)` cubre el caso real.
- **Renombre del marcador**. `<!-- TEMPLATE-PLACEHOLDER -->` es funcional y descriptivo. No se renombra.
- **Cambio en el formato del marcador a algo no-HTML** (por ejemplo `# TEMPLATE-PLACEHOLDER` como heading). Descartado: como comentario HTML, el marcador es invisible en preview de markdown, lo que es exactamente lo que queremos (no contamina la lectura del usuario si abre el template).
- **Bump de README.md e inicio-chat.txt**. Estéticamente recomendable pero no crítico para el funcionamiento. Si el usuario quiere paridad completa, puede bumpear a mano (búsqueda y reemplazo).

---

## Próxima parada

v4.8 candidatos (sin compromiso):

- Refactor del meta-doc: separar `architecture.md` de SSOT en `SYSTEM-INFO.md` (sistema) + `architecture.md` (template real del proyecto).
- Bump cosmético de `README.md` e `inicio-chat.txt` si surge una versión con cambios funcionales suficientes para justificarlo.
- Si el módulo 12 del bootcamp introduce algo nuevo, bump correspondiente.
- Si surge un caso real en `quellevoyo` o `PromptVault` que requiera nuevo skill o workflow.
