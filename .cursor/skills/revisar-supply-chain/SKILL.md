---
name: revisar-supply-chain
description: Auditar la cadena de suministro de un proyecto Node.js o JS/TS - detecta lockfile sin commitear, ausencia de minimumReleaseAge, lifecycle scripts no controlados, patrones peligrosos en GitHub Actions (pull_request_target con checkout de fork, acciones sin pinear a SHA), OIDC sin pinning a branch+workflow, y dependencias sospechosamente nuevas. Cross-referencia el lockfile con bases publicas de paquetes comprometidos (TanStack, Shai-Hulud, axios, Trivy). Solo reporta, NO modifica codigo ni configuracion.
disable-model-invocation: true
---

# /revisar-supply-chain — Auditar la cadena de suministro

Este comando audita las defensas de supply chain de un proyecto Node.js / JS / TS en las 4 capas (dependency resolution, install-time execution, CI execution, publish path). Genera un reporte con hallazgos clasificados por severidad y crea tickets en el roadmap para los críticos.

**Solo reporta. NUNCA modifica código, configuración ni dependencias automáticamente.**

---

## Cuándo usar este comando

- Auditoría inicial al tomar control de un proyecto existente (`/regularizar` opción [4] lo dispara automáticamente si detecta `package.json`).
- Después de un incidente público de cadena de suministro (TanStack, Shai-Hulud, axios) para verificar exposición.
- Periódicamente (cada release importante o cada N semanas) como parte de la rutina de seguridad.
- Antes de un release a producción para verificar que las defensas siguen activas.

---

## Pasos del protocolo

### Paso 1 — Verificar aplicabilidad

Antes de auditar, confirmar que el proyecto usa npm registry:

- ¿Existe `package.json` en el proyecto? Si NO, este comando no aplica. Salir con mensaje informativo.
- Si existe, leer `package.json` para identificar el package manager (`packageManager` field), tipo de proyecto (app, librería, monorepo), y si publica al registro (`name` no scoped privado).

### Paso 2 — Cargar la skill de referencia

Cargar `@.cursor/skills/supply-chain-skill/SKILL.md` como guía obligatoria para criterios de auditoría.

### Paso 3 — Auditoría Capa 1: Dependency resolution

Verificar:

| Comprobación | Cómo | Severidad si falla |
|---|---|---|
| Lockfile commiteado | `git ls-files | grep -E '(package-lock\|pnpm-lock\|yarn\|bun)'` | 🔴 Alta |
| `minimumReleaseAge` activo | leer `pnpm-workspace.yaml`, `.npmrc`, `.yarnrc.yml`, `bunfig.toml` | 🟠 Media (alta si publica a prod) |
| `blockExoticSubdeps` activo (pnpm) | leer `pnpm-workspace.yaml` | 🟡 Baja |
| Dependencias desde URLs exóticas | grep `"github:"`, `"git+"`, tarball URLs en `package.json` | 🔴 Alta si transitivas |
| `trustPolicy: no-downgrade` (pnpm) | leer `pnpm-workspace.yaml` | 🟢 Informativo |

Para cada paquete del lockfile, comprobar fecha de publicación de la versión instalada con `npm view <pkg>@<version> time`. Listar paquetes con menos de 24h de vida como sospechosos (no necesariamente maliciosos, pero merecen revisión).

### Paso 4 — Auditoría Capa 2: Install-time execution

Verificar:

| Comprobación | Cómo | Severidad si falla |
|---|---|---|
| `strictDepBuilds` o equivalente | leer config del package manager | 🟠 Media |
| `allowBuilds` definido (pnpm 11) | leer `pnpm-workspace.yaml` | 🟠 Media |
| `ignore-scripts` activo (npm clásico) | leer `.npmrc` | 🟠 Media |
| Paquetes con build scripts no aprobados | comparar lifecycle scripts del lockfile vs `allowBuilds` | 🔴 Alta si hay desconocidos |

Listar todos los paquetes que ejecutan `preinstall`, `install`, `postinstall`, `prepublish`, `prepare` con su justificación (si es legítima: binarios nativos, compilación obligatoria).

### Paso 5 — Auditoría Capa 3: CI execution (GitHub Actions)

Solo si existe `.github/workflows/`. Verificar cada workflow:

| Patrón peligroso | Detección | Severidad |
|---|---|---|
| `pull_request_target` con checkout de fork | grep `pull_request_target` + `refs/pull/.*/merge` | 🔴 Crítica |
| `actions/cache` compartida entre PRs y release | analizar workflows que escriben y leen mismo key de cache | 🔴 Alta |
| Acciones pineadas a tag mutable (`@v6`, `@main`) | grep `uses:` sin SHA de 40 chars | 🟠 Media |
| Permisos por defecto (no `permissions: {}`) | leer cabecera de workflow | 🟡 Baja |
| Secrets accesibles desde workflows de PR | revisar `secrets.*` en `pull_request` triggers | 🔴 Alta |

### Paso 6 — Auditoría Capa 4: Publish path

Solo si el proyecto publica al registro (`private: false` en `package.json` y `name` sin scope privado). Verificar:

| Comprobación | Cómo | Severidad si falla |
|---|---|---|
| Publica con OIDC, no con PAT | revisar workflow de release | 🟠 Media |
| OIDC trusted publisher pineado a workflow+branch | comprobar en npmjs.com (manual o vía API) | 🔴 Alta |
| Branch protection sobre `main` | `gh api repos/:owner/:repo/branches/main/protection` | 🟠 Media |
| 2FA activado en cuenta de publicación | `npm profile get` | 🔴 Crítica |
| Lockfile en repo público no expone tokens | grep `_authToken`, `_password` en lockfile y `.npmrc` | 🔴 Crítica |

### Paso 7 — Verificación de incidentes conocidos

Cross-referenciar lockfile con bases públicas de paquetes comprometidos:

- TanStack mayo 2026: GHSA-g7cv-rxg3-hmpx (42 paquetes `@tanstack/*` + propagación).
- Shai-Hulud septiembre 2025 + ondas: lista en OSSF Malicious Packages.
- axios marzo 2026: versiones 1.14.1 y 0.30.4.
- Trivy marzo 2026: trivy-action y setup-trivy con tags repointed.

Si encuentras coincidencias: **severidad crítica + ticket urgente en roadmap + protocolo de respuesta a incidente** (ver `supply-chain-skill` sección "Respuesta a incidente").

### Paso 8 — Generar reporte

Crear `docs/supply-chain-audit.md` con la siguiente estructura:

```markdown
# Supply Chain Audit — [fecha]

## Resumen ejecutivo
- Total hallazgos: X críticos, Y altos, Z medios, W bajos
- Paquetes comprometidos detectados: [lista o "ninguno"]
- Capas con defensas activas: [1/4, 2/4, 3/4, 4/4]

## Hallazgos por capa

### Capa 1: Dependency resolution
[Tabla con hallazgos]

### Capa 2: Install-time execution
[Tabla con hallazgos]

### Capa 3: CI execution
[Tabla con hallazgos]

### Capa 4: Publish path
[Tabla con hallazgos]

## Recomendaciones priorizadas
1. [Acción más urgente]
2. [Segunda más urgente]
...

## Validación manual pendiente
- [ ] Revisar OIDC trusted publisher en npmjs.com
- [ ] Verificar branch protection en GitHub
- [ ] Confirmar 2FA en cuentas con permisos de publicación
```

### Paso 9 — Compartir hallazgos de seguridad

Si hay hallazgos de severidad crítica o alta, compartir con `docs/investigacion_seguridad.md` (compartido con auditoría de BD y frontend si existen).

### Paso 10 — Crear tickets

Crear tickets en `roadmap.md` por cada hallazgo crítico y alto. Por ejemplo:

```markdown
- [ ] [CRÍTICO supply-chain] Habilitar minimumReleaseAge: 1440 en pnpm-workspace.yaml
- [ ] [CRÍTICO supply-chain] Eliminar pull_request_target + checkout del fork en .github/workflows/bench.yml
- [ ] [ALTO supply-chain] Pinear actions/checkout a SHA en lugar de @v6
```

### Paso 11 — Cierre

Ejecutar `On(task_complete)` con el checklist de sincronización documental.

---

## Reportar sin paranoia

**No todo hallazgo es bloqueante.** Calibrar:

- **Crítico**: bloquea el release o requiere remediación inmediata (paquete comprometido instalado, secrets expuestos, `pull_request_target` mal usado).
- **Alto**: arreglar en el siguiente sprint (sin lockfile, sin `minimumReleaseAge` si publicas a producción).
- **Medio**: añadir a roadmap (sin `blockExoticSubdeps`, acciones sin SHA pinning).
- **Bajo / Informativo**: documentar para revisión futura.

La auditoría es para **reducir riesgo**, no para introducir fricción innecesaria. Un proyecto de aprendizaje no necesita las 4 capas activas el día 1; un proyecto en producción sí.

---

## Regla de oro

> **La auditoría detecta, el humano decide, el humano arregla.**

`/revisar-supply-chain` NUNCA modifica `.npmrc`, `pnpm-workspace.yaml`, workflows ni dependencias. Genera el reporte y crea tickets. La remediación pasa por `/nueva` o `/reparar` después de que el usuario priorice.
