# Changelog v4.5 → v4.6 (Cursor Edition — Supply Chain Security)

Sesión de actualización: 15 de mayo de 2026

Origen: ataque a TanStack del 11 de mayo de 2026 (84 versiones maliciosas en 42 paquetes `@tanstack/*` con propagación a Mistral AI, UiPath, OpenSearch y >170 paquetes). Es el primer caso documentado de paquetes maliciosos en npm con **provenance SLSA Build Level 3 válido** (GHSA-g7cv-rxg3-hmpx, CVE-2026-45321). El ataque encadenó tres vulnerabilidades en GitHub Actions (`pull_request_target` "Pwn Request" + cache poisoning + extracción de OIDC token de memoria del runner); no se robaron credenciales npm, todas las defensas criptográficas estándar (2FA, OIDC, provenance) funcionaron como debían y aun así no lo evitaron. Atribución a TeamPCP (mismo grupo que Trivy marzo 2026, Bitwarden abril 2026, SAP abril 2026).

Respuesta del ecosistema previa al ataque: pnpm 11 (28 abril 2026) activó por defecto `minimumReleaseAge: 1440` (1 día) y `blockExoticSubdeps: true`; Yarn Berry 4.10 (septiembre 2025) añadió `npmMinimalAgeGate` con default de 3 días; Bun 1.3 (octubre 2025) añadió `minimumReleaseAge` opt-in; npm CLI 11.10 (febrero 2026) añadió `min-release-age` opt-in.

Esta release mantiene **paridad total** con la versión Windsurf v4.6, publicada en el mismo bump.

---

## Resumen del cambio

La cadena de suministro pasa a ser ciudadano de primera clase del scaffolding, al mismo nivel que tests, base de datos y frontend. Cuando el proyecto usa npm registry (Node, JS/TS, frontend con npm), el sistema activa automáticamente **4 capas de defensa** desde `On(testing_setup)`:

1. **Dependency resolution** — `minimumReleaseAge`, `blockExoticSubdeps`, `trustPolicy`.
2. **Install-time execution** — `allowBuilds` (deny by default), `ignore-scripts`.
3. **CI execution** — GitHub Actions hardening (NO `pull_request_target` con checkout de fork, acciones pineadas a SHA, permisos mínimos).
4. **Publish path** — OIDC trusted publishing pineado a `workflow + branch`, branch protection en main, 2FA obligatorio.

La filosofía es la misma de versiones anteriores: **detectar señales, configurar entorno automáticamente, registrar ADRs, validación humana en los puntos críticos**. No se rompe nada de v4.5; las defensas de supply chain son opt-in para proyectos existentes (vía `/revisar-supply-chain` o `/regularizar` [4]) y opt-out documentado vía ADR para proyectos nuevos.

---

## Cambios

### 1. Skill nueva: `.cursor/skills/supply-chain-skill/SKILL.md`

**Estado:** archivo nuevo (~20 KB) con frontmatter `name: supply-chain-skill` + `description:` extensa (sin tildes, estilo Cursor).

Auto-invocable cuando la IA detecta trabajo con dependencias, lockfiles, workflows de GitHub Actions o publicación de paquetes. Cubre la cadena de suministro end-to-end:

- **Principio fundamental**: 2FA + OIDC + SLSA provenance son necesarios pero NO suficientes (lo demostró el ataque TanStack con provenance SLSA Build Level 3 VÁLIDO sobre paquetes maliciosos). La defensa real es asumir que cualquier versión recién publicada puede estar comprometida y dejarla "enfriar" antes de instalarla.
- **Contexto de incidentes 2025-2026**: Shai-Hulud (sep 2025, ~796 paquetes con 132M descargas/mes), axios marzo 2026 (RAT en `postinstall`), Trivy marzo 2026 (76 tags repointed), Bitwarden + SAP abril 2026 (TeamPCP), Mini Shai-Hulud / TanStack mayo 2026 (84 versiones + propagación + provenance válido + persistence daemon con `rm -rf ~/`).
- **Las 4 capas de defensa** con tabla operativa: pregunta que responde cada una + herramienta principal.
- **Capa 1 detallada**: setup por package manager (pnpm 11, npm 11.10+, Yarn Berry 4.10+, Bun 1.3+, Renovate, Dependabot). Distinción entre `minimumReleaseAge` (todos los upgrades) y security updates (fast-track).
- **Capa 2 detallada**: "deny by default" con `allowBuilds` (pnpm) o `--ignore-scripts` (npm), comandos para identificar paquetes con build scripts (`pnpm install --dry-run`, `npm explain`), defensa contra lockfile injection con `--frozen-lockfile` en CI.
- **Capa 3 detallada**: patrón "Pwn Request" como antipatrón NUNCA usar, patrones seguros, mitigación de cache poisoning, pineado de acciones a SHA en lugar de tag (caso Trivy: 76 tags repointed), permisos mínimos por job.
- **Capa 4 detallada**: tabla PAT clásico vs OIDC, configuración de trusted publisher en npmjs.com con pinning a `Repository + Workflow + Branch`, branch protection con "Block force pushes", 2FA obligatorio.
- **Respuesta a incidente con orden CRÍTICO** (6 pasos):
  1. NO revocar token de GitHub todavía (vector `rm -rf ~/` al revocar).
  2. Limpiar daemon de persistencia (`gh-token-monitor`).
  3. Buscar payloads residuales (`router_runtime.js`, `setup.mjs`, `tanstack_runner.js`).
  4. Aislar la máquina de la red.
  5. Rotar TODAS las credenciales accesibles.
  6. Auditar blast radius y documentar ADR.
- Integración con el harness Architect-Brain, checklist obligatorio antes de mergear cambios en dependencias (6 puntos), antipatrones operativos (9 entradas), regla de oro final.

### 2. Skill nueva: `.cursor/skills/revisar-supply-chain/SKILL.md`

**Estado:** archivo nuevo con frontmatter `name: revisar-supply-chain` + `description:` extensa + `disable-model-invocation: true`.

Slash command `/revisar-supply-chain`. Pasos:

1. Verificar aplicabilidad (existe `package.json`).
2. Cargar la skill `supply-chain-skill`.
3. Auditoría Capa 1 (lockfile commiteado, `minimumReleaseAge`, `blockExoticSubdeps`, URLs exóticas, `trustPolicy`).
4. Auditoría Capa 2 (`strictDepBuilds`, `allowBuilds`, `ignore-scripts`, paquetes con build scripts no aprobados).
5. Auditoría Capa 3 (`pull_request_target` con checkout de fork = CRÍTICA, cache compartida, acciones con tag mutable, permisos por defecto, secrets en workflows de PR).
6. Auditoría Capa 4 (OIDC vs PAT, trusted publisher pineado a workflow+branch, branch protection, 2FA, tokens en lockfile).
7. Verificación de incidentes conocidos: cross-referencia con TanStack GHSA-g7cv-rxg3-hmpx, Shai-Hulud, axios 1.14.1/0.30.4, Trivy.
8. Generar `docs/supply-chain-audit.md` con hallazgos por severidad (crítica/alta/media/baja) y validación manual pendiente.
9. Compartir hallazgos críticos/altos con `docs/investigacion_seguridad.md`.
10. Crear tickets en `roadmap.md` por cada hallazgo crítico o alto.
11. Cierre con `On(task_complete)`.

NUNCA modifica `.npmrc`, `pnpm-workspace.yaml`, workflows ni dependencias. La IA detecta, el humano decide, el humano arregla.

### 3. Doc nuevo en SSOT: `docs/supply-chain-strategy.md`

**Estado:** plantilla nueva.

Plantilla que se rellena durante `On(supply_chain_setup)`. Secciones: resumen ejecutivo, Capa 1 (package manager, cooldown, exclusiones, bloqueo exotic, trust policy), Capa 2 (política, paquetes en `allowBuilds`, ejecución de scripts en CI), Capa 3 (workflows auditados con tabla de trigger/riesgo/estado, patrones prohibidos verificados), Capa 4 (trusted publisher OIDC, branch protection, 2FA), Renovate/Dependabot, **plan de respuesta a incidente paso a paso**, excepciones documentadas, próximas revisiones, referencias a skill y workflow.

### 4. Modificado: `prompts/architect-brain.md`

- Versión `v4.5 (Cursor Edition — Testing 2026: BDD + Integration + AI-Assisted + Frontend + MCP + Database)` → `v4.6 (Cursor Edition — Supply Chain Security + Testing 2026 BDD + Integration + AI-Assisted + Frontend + MCP + Database)`.
- `On(project_start)`: añadido paso 6 que dispara `On(supply_chain_setup)` si el stack es JS/TS o usa npm registry.
- `On(testing_setup)`: ampliado de 13 a 14 pasos. Paso 12 NUEVO dispara `On(supply_chain_setup)` cuando hay `package.json`. Paso 14 (antes 13) documenta defensas supply chain en `testing-strategy.md` y `decisions-log.md`.
- Insertado protocolo NUEVO `On(supply_chain_setup)` antes de `On(database_setup)` (10 puntos, idéntico contenido que Windsurf con paths `.cursor/skills/supply-chain-skill/SKILL.md`).
- Insertado protocolo NUEVO `On(supply_chain_audit)` después de `On(supply_chain_setup)` (10 puntos).
- `On(task_complete)` checklist: añadidas líneas `supply-chain-strategy.md` y "Defensas supply chain intactas".
- `On(sync_check)`: añadidas 5 divergencias supply chain.
- `On(ticket_close)`: paso 8 nuevo verifica `supply-chain-strategy.md` si el ticket tocó dependencias, lockfile, `.npmrc`, `pnpm-workspace.yaml` o workflows.
- "Restricciones y Calidad": añadida consulta a `docs/supply-chain-strategy.md` antes de tocar dependencias, ejecución de `On(supply_chain_audit)` si encuentra proyecto JS/TS sin defensas, y 4 reglas operativas.

### 5. Modificado: `.cursor/rules/architect-brain.mdc`

- INTERRUPCION DE SEGURIDAD: añadida pregunta 9 sobre cambios en `package.json`, lockfile, `.npmrc`, `pnpm-workspace.yaml` o `.github/workflows/`.
- CASO A (proyecto nuevo): añadido paso 7 que dispara `On(supply_chain_setup)` si el proyecto usa npm registry.
- CASO B (proyecto existente) opción [4]: añadido bloque "Si el proyecto usa npm registry → `On(supply_chain_audit)`" después del bloque de frontend.
- Tabla de comandos: añadida fila para `/revisar-supply-chain`.
- Bloque NUEVO "SUPPLY CHAIN (obligatorio si el proyecto usa npm registry)" después del bloque FRONTEND.
- REGLAS DE ORO: añadidas 3 entradas finales (Defensa en 4 capas, Lifecycle scripts denegados por defecto, Respuesta a incidente con orden crítico).

### 6. Modificado: `docs/testing-strategy.md`

Añadida sección "Supply chain (si el proyecto usa npm registry)" al final con: configuración del package manager, tests con dependencias muy nuevas, CI hardening, respuesta a incidente.

### 7. Modificado: `CLAUDE.md`

- Versión `v4.5` → `v4.6`. Mención explícita a cadena de suministro.
- Lista de protocolos `On(...)` actualizada (añadidos `supply_chain_setup`, `supply_chain_audit`).
- SSOT amplía con `supply-chain-strategy.md`.
- Slash commands: añadido `/revisar-supply-chain`.
- Skills auxiliares: añadida `supply-chain-skill`.
- Reglas de oro 14 y 15 nuevas (Supply chain en 4 capas + Respuesta a incidente con orden crítico).
- Interrupción de seguridad: añadida pregunta 9.

### 8. Modificado: `AGENTS.md`

- Versión `v4.5` → `v4.6`. Mención explícita a cadena de suministro.
- Slash commands: añadido `/revisar-supply-chain`.
- Skills auxiliares: añadida `supply-chain-skill`.
- Reglas de oro 14 y 15 nuevas.
- SSOT amplía con `supply-chain-strategy.md`.

### 9. Modificado: `README.md`

- Cabecera amplía descripción con "cadena de suministro (npm + GitHub Actions + OIDC publishing)".
- Sección "Qué resuelve": 8 → 9 problemas (punto 9 nuevo sobre `npm install` y ataques tipo Shai-Hulud/TanStack).
- Sección "Slash commands disponibles": 13 → 14 commands; fila nueva `/revisar-supply-chain`.
- Sección "Qué hace el sistema": 9 → 10 pilares; pilar 10 nuevo "Defensa de cadena de suministro en 4 capas".
- Estructura de archivos: skills 21 → 23 (14 commands + 9 auxiliares), docs con `supply-chain-strategy.md`.
- Reglas de oro: 17 → 20 con entradas 18, 19 y 20 nuevas.
- Sección "Sobre este sistema": añadida entrada v4.6 actual marcando v4.5 como histórica.
- Soporte para Claude Code: versión actualizada a v4.6.

### 10. Modificado: `inicio-chat.txt`

- Cabecera versión y subtítulo actualizados a v4.6 (Cursor Edition — Supply Chain Security).
- Tabla de comandos: 13 → 14, fila `/revisar-supply-chain`.
- Sección detallada nueva `/revisar-supply-chain` con descripción de las 4 capas, cross-referencia con incidentes públicos, y bloque de respuesta a incidente con orden crítico.
- Reglas de oro: 17 → 20 con entradas 18, 19 y 20 nuevas.
- Estructura de archivos: skills 21 → 23, docs con `supply-chain-strategy.md`.
- Protocolo de defensa: amplía menciones para incluir supply chain y `/revisar-supply-chain`.

---

## Migración

### Archivos nuevos en este bump

- `.cursor/skills/supply-chain-skill/SKILL.md`
- `.cursor/skills/revisar-supply-chain/SKILL.md`
- `docs/supply-chain-strategy.md`
- `CHANGELOG-v4.6.md`

### Archivos modificados

- `prompts/architect-brain.md`
- `.cursor/rules/architect-brain.mdc`
- `docs/testing-strategy.md`
- `CLAUDE.md`
- `AGENTS.md`
- `README.md`
- `inicio-chat.txt`

### Comportamiento para proyectos existentes

Los proyectos existentes en v4.5 siguen funcionando sin cambios. Para activar las defensas de supply chain en uno de ellos, ejecutar `/revisar-supply-chain` (audit only) o pedir explícitamente "configura las defensas de supply chain del proyecto" (que dispara `On(supply_chain_setup)`).

---

## Verificación post-migración

Tras aplicar v4.6, comprobar:

1. **Slash commands**: en el chat de Agent, escribir `/` y verificar que aparece `/revisar-supply-chain` en la lista (total 14 comandos).
2. **Carga de skill**: invocar `@.cursor/skills/supply-chain-skill/SKILL.md` y comprobar que se carga sin errores.
3. **Carga del workflow**: ejecutar `/revisar-supply-chain` en un proyecto sin `package.json` y verificar que sale con mensaje informativo ("no aplica").
4. **Pregunta sobre TanStack**: preguntar "¿me afecta el ataque a TanStack?" en un proyecto cualquiera y verificar que la IA aplica el contexto correcto (sí afecta si hay `@tanstack/react-router` en lockfile, no afecta si no hay `package.json`).
5. **Orden crítico de respuesta a incidente**: preguntar "¿qué hago si he instalado un paquete comprometido?" y verificar que la IA responde con "NO revocar token de GitHub antes de limpiar persistencia" como primer paso.
6. **Restricciones**: pedir "desactiva minimumReleaseAge" en un proyecto con la defensa activada, y verificar que la IA pide ADR documentando el motivo antes de proceder.

---

## Paridad con Windsurf v4.6

Esta release tiene **paridad total** con la versión Windsurf (`junior-doc-gen-tests`) v4.6, publicada en el mismo bump:

- Mismos 4 archivos nuevos (skill, workflow como skill en `.cursor/skills/`, doc SSOT, changelog).
- Mismas modificaciones en los 7 archivos meta.
- Mismos protocolos `On(supply_chain_setup)` y `On(supply_chain_audit)`.
- Mismas 4 capas de defensa y mismo orden crítico de respuesta a incidente.

Las únicas diferencias son las propias de cada motor:

- **Windsurf**: skill en `prompts/skills/X.md`, workflow en `.windsurf/workflows/X.md` con frontmatter `description:`, reglas globales en `.windsurfrules`.
- **Cursor (esta versión)**: skill en `.cursor/skills/X/SKILL.md` con frontmatter `name:` + `description:` extensa (sin tildes en `description`), reglas globales en `.cursor/rules/architect-brain.mdc`. Las referencias a otras skills usan nombres sin extensión (`supply-chain-skill`) en lugar de paths.

---

## Lo que NO entró en v4.6 (intencionado)

- **Herramienta automática de remediación**. La filosofía sigue siendo "la IA detecta, el humano decide, el humano arregla". `/revisar-supply-chain` NUNCA modifica configuración ni dependencias. Igual que `/revisar-bd` y `/revisar-frontend`.
- **MCP server específico de supply chain**. Considerado pero descartado: las herramientas SaaS del ecosistema (Socket, Snyk, Aikido) ya tienen integraciones nativas que sería redundante envolver. La skill apunta a ellas como referencia.
- **Integración por defecto con herramientas SaaS**. Socket / Snyk / Aikido se mencionan como opciones, pero la activación es opt-in (no se asume budget ni cuenta).
- **Cambios en defaults de v4.5**. Las defensas de supply chain son opt-in para proyectos existentes (vía `/revisar-supply-chain`) y opt-out documentado vía ADR para proyectos nuevos (porque sería raro pero legítimo desactivar `minimumReleaseAge` en un sandbox temporal).
- **Soporte para registros privados / mirrors**. El scaffolding asume npm registry público o GitHub Packages. Verdaccio, Artifactory, Nexus y similares quedan para una v4.7 si surge la necesidad.

---

## Próxima parada

v4.7 candidatos (sin compromiso):

- Si el módulo 12 del curso AI4Devs introduce algo nuevo (deployment, observabilidad, security operations), bump correspondiente.
- Si surge un caso real en PromptVault o LTI-GFS que requiera nuevo skill o workflow, bump dedicado.
- Mantenimiento de IOCs y bases de paquetes comprometidos si aparecen incidentes nuevos del ecosistema npm.
