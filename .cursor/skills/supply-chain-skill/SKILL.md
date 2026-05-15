---
name: supply-chain-skill
description: Seguridad de la cadena de suministro en proyectos JS/TS y stacks que usan npm registry. Cubre las 4 capas de defensa - dependency resolution con minimumReleaseAge (pnpm 11, Yarn Berry, Bun, npm 11.10), install-time execution con allowBuilds y --ignore-scripts, CI execution con hardening de GitHub Actions (NO pull_request_target con checkout de fork, acciones pineadas a SHA, permisos minimos), publish path con OIDC trusted publishing pineado a workflow+branch. Incluye contexto de incidentes 2025-2026 (Shai-Hulud, axios, Trivy, TanStack Mini Shai-Hulud del 11 mayo 2026), respuesta a incidente con orden critico (NO revocar token de GitHub antes de limpiar persistencia, daemon que ejecuta rm -rf), y antipatrones. Complementaria a tests-skill, integration-testing-skill, ai-testing-skill.
---

# Skill: Supply Chain Security (npm, GitHub Actions, OIDC publishing)

Esta skill cubre la **seguridad de la cadena de suministro** en proyectos JavaScript/TypeScript: defensas client-side al instalar dependencias, hardening de GitHub Actions, publicación con OIDC trusted publishing, y respuesta a incidentes cuando un paquete del ecosistema es comprometido.

Es transversal a `tests-skill`, `integration-testing-skill`, `visual-testing-skill`, `bdd-skill` y `ai-testing-skill`: aplica siempre que el proyecto consuma o publique paquetes npm.

---

## Principio fundamental

> **2FA, OIDC trusted publishing, SLSA provenance y npm audit son necesarios pero NO suficientes. El ataque a TanStack del 11 de mayo de 2026 publicó paquetes maliciosos con provenance SLSA Build Level 3 VÁLIDO desde el workflow legítimo del repositorio. Toda la criptografía funcionó como debía y aun así no lo evitó.**

La conclusión operativa es dura: **no puedes confiar en que un paquete es seguro porque pasa las verificaciones criptográficas**. La defensa real es asumir que cualquier versión recién publicada puede estar comprometida y dejarla "enfriar" antes de instalarla. Ese es el principio de `minimumReleaseAge`.

---

## Cuándo activar esta skill

- Vas a arrancar un proyecto NUEVO con Node.js / TypeScript / cualquier stack que use npm registry.
- Vas a auditar un proyecto EXISTENTE que ya consume paquetes npm.
- Vas a publicar paquetes propios al registry (npm, GitHub Packages).
- Vas a configurar pipelines de CI/CD con GitHub Actions que ejecuten `pull_request_target` o que publiquen a npm.
- Has detectado o sospechas un incidente de cadena de suministro (paquete comprometido, instalación maliciosa).

NO actives esta skill si:

- El proyecto es 100% PHP, Python, Java o cualquier stack que no consuma npm registry (los principios son aplicables pero las herramientas son otras).
- Es un script throwaway en un sandbox aislado donde no hay credenciales que perder.

---

## Contexto: por qué esta skill existe en v4.6

Tres incidentes recientes definen el panorama 2025-2026:

1. **Shai-Hulud** (septiembre 2025 + ondas posteriores): worm que comprometió ~796 paquetes con 132M descargas mensuales combinadas usando scripts `postinstall` para robar credenciales.
2. **axios + Trivy + Bitwarden + SAP** (marzo-abril 2026, grupo TeamPCP): hijack progresivo de paquetes populares con RAT (remote access trojan) en `postinstall`.
3. **Mini Shai-Hulud / TanStack** (11 de mayo de 2026): 84 versiones maliciosas en 42 paquetes `@tanstack/*`, con propagación a Mistral AI, UiPath, OpenSearch y >170 paquetes. **Primer caso documentado con SLSA Build Level 3 provenance VÁLIDO sobre paquetes maliciosos**. Carga útil con persistence daemon que ejecuta `rm -rf ~/` cuando el usuario revoca el token de GitHub (la respuesta "correcta" es el trigger del ataque).

La respuesta del ecosistema:

- **pnpm 11** (28 abril 2026): activa por defecto `minimumReleaseAge: 1440` (1 día) y `blockExoticSubdeps: true`.
- **Yarn Berry 4.10** (septiembre 2025): añade `npmMinimalAgeGate` con default de 3 días.
- **Bun 1.3** (octubre 2025): añade `minimumReleaseAge` (opt-in).
- **npm CLI 11.10** (febrero 2026): añade `min-release-age` (opt-in).

**Estado actual**: todos los gestores principales tienen cooldown de release. Solo pnpm lo trae activado por defecto. Es trabajo del developer activarlo en los demás.

---

## Las 4 capas de defensa

Pensar la cadena de suministro como cuatro capas separadas ayuda a no dejar huecos:

| Capa | Pregunta que responde | Herramienta principal |
|---|---|---|
| **1. Dependency resolution** | ¿Qué versiones pueden instalarse? | `minimumReleaseAge`, lockfile, `blockExoticSubdeps` |
| **2. Install-time execution** | ¿Qué código se ejecuta durante `install`? | `allowBuilds`, `--ignore-scripts`, `strictDepBuilds` |
| **3. CI execution** | ¿Cómo se ejecutan los workflows? | Hardening de GitHub Actions, OIDC con branch/workflow pinning |
| **4. Publish path** | ¿Cómo se publican mis paquetes? | OIDC trusted publishing, branch protection, 2FA |

Las capas 1 y 2 son **client-side**: protegen tu máquina y tu CI cuando instalas dependencias de terceros. Las capas 3 y 4 son **publish-side**: protegen el código que tú publicas para que otros no se vean afectados desde ti.

---

## Capa 1: Dependency resolution (cooldown de release)

La defensa más efectiva contra ataques zero-day a paquetes populares. La lógica es simple: la mayoría de paquetes maliciosos se detectan y retiran del registro en horas (Shai-Hulud: ~12h; debug/chalk: ~2.5h; TanStack: ~20 minutos). Un cooldown de 24 horas habría bloqueado los tres.

### pnpm (recomendado por defaults)

```yaml
# pnpm-workspace.yaml
minimumReleaseAge: 1440  # minutos = 1 día (default en pnpm 11)
minimumReleaseAgeExclude:
  - '@types/*'           # tipos suelen ser seguros y se actualizan rápido
  - 'typescript'         # toolchain confiable
  # NO añadir aquí paquetes solo porque "los necesitas ya"
blockExoticSubdeps: true  # default en pnpm 11
trustPolicy: no-downgrade  # detecta si un paquete pierde provenance entre versiones
```

`minimumReleaseAge` está en **minutos**. Valores típicos:

- `1440` = 1 día (default pnpm 11, mínimo recomendado).
- `4320` = 3 días (default Yarn Berry, equilibrio velocidad/seguridad).
- `10080` = 1 semana (proyectos críticos en producción).
- `0` = desactivado (NO recomendado).

### npm (CLI 11.10+)

```ini
# .npmrc
min-release-age=2d
min-release-age-exclude=@types/*,typescript
```

`min-release-age` en npm acepta sufijos `d` (días), `h` (horas), `m` (minutos).

### Yarn Berry (4.10+)

```yaml
# .yarnrc.yml
npmMinimalAgeGate: "3d"
npmPreapprovedPackages:
  - "@types/*"
  - "typescript"
```

### Bun (1.3+)

```toml
# bunfig.toml
[install]
minimumReleaseAge = 259200  # segundos = 3 días
minimumReleaseAgeExcludes = ["@types/bun", "typescript"]
```

### Renovate / Dependabot

Si usas bot de actualización automática, configura cooldown ahí también:

```json5
// renovate.json
{
  "minimumReleaseAge": "3 days"
}
```

```yaml
# .github/dependabot.yml
updates:
  - package-ecosystem: "npm"
    cooldown:
      default-days: 3
```

**Importante**: Dependabot aplica cooldown solo a actualizaciones de versión, NO a security updates. Esto es bueno: una vulnerabilidad conocida tiene fix prioritario sobre el riesgo de zero-day en versión nueva.

---

## Capa 2: Install-time execution (lifecycle scripts)

El vector clásico de los worms de npm: `preinstall` y `postinstall` que se ejecutan al instalar la dependencia. Defenderse aquí es tan importante como el cooldown.

### Política recomendada: deny by default

```yaml
# pnpm-workspace.yaml
strictDepBuilds: true  # falla si hay build scripts no aprobados
allowBuilds:
  electron: true       # binarios nativos legítimos
  esbuild: true
  sharp: true
  "@swc/core": true
  # cualquier paquete que NO esté aquí, NO ejecuta scripts
```

En npm clásico no hay `allowBuilds`; lo más cerca es:

```bash
npm install --ignore-scripts
# o globalmente:
npm config set ignore-scripts true
```

Esto **rompe paquetes que legítimamente necesitan compilación** (binarios nativos). El compromiso es: deshabilitar por defecto y aprobar caso a caso.

### Cómo identificar paquetes con build scripts

```bash
# pnpm
pnpm install --dry-run
# muestra qué scripts intentarían ejecutarse

# npm
npm explain <pkg>
# muestra la cadena que llega a un paquete y sus scripts
```

### Lockfile correcto

El lockfile NO es defensa contra paquetes maliciosos publicados, pero SÍ contra **lockfile injection** (un PR que cambia el tarball URL en el lockfile a uno malicioso):

- **Commitea siempre** `package-lock.json` / `pnpm-lock.yaml` / `yarn.lock` / `bun.lock`.
- En CI usa `npm ci` / `pnpm install --frozen-lockfile` / `yarn install --immutable` / `bun install --frozen-lockfile`. NUNCA `npm install` en CI: regenera el lockfile silenciosamente.
- Yarn Berry trae `enableHardenedMode: true` por defecto en PRs de GitHub públicos, que valida el lockfile contra el registro. Mantenlo activado.

---

## Capa 3: CI execution (GitHub Actions hardening)

El ataque a TanStack no robó credenciales: explotó vulnerabilidades en GitHub Actions encadenadas. Esta capa es la más infravalorada y la que más dolor ahorra.

### El patrón "Pwn Request" — NUNCA usar

```yaml
# ❌ PELIGROSO: pull_request_target con checkout del fork
on:
  pull_request_target:
    paths: ['packages/**']

jobs:
  build:
    steps:
      - uses: actions/checkout@v6
        with:
          ref: refs/pull/${{ github.event.pull_request.number }}/merge
      - run: npm install  # ejecuta código del FORK con permisos del BASE
```

`pull_request_target` se ejecuta en el contexto del repo base (con sus secrets) pero el checkout trae código del fork. Si combinas ambos, **estás ejecutando código no revisado del fork con tus secrets**. Es exactamente el vector del TanStack.

### Patrón seguro

```yaml
# ✅ Para operaciones de solo lectura (etiquetas, comentarios):
on:
  pull_request_target:
permissions:
  pull-requests: write
  contents: read  # explícito, mínimo necesario
jobs:
  label:
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.addLabels({ ... });
      # NO hacer checkout del fork. NO ejecutar npm install.
```

```yaml
# ✅ Para CI normal de PRs:
on:
  pull_request:  # NO target
jobs:
  test:
    steps:
      - uses: actions/checkout@v6
      - run: npm ci && npm test
      # corre con permisos limitados, sin acceso a secrets del repo base
```

### Cache poisoning

`actions/cache@v5` usa un token interno del runner, NO `GITHUB_TOKEN`. Restricciones de permisos vía `permissions:` NO previenen escritura en cache. Mitigación:

- NO compartir cache entre workflows con distinto trust boundary.
- Usar `actions/cache/restore` (sin save) en workflows que ejecutan código no confiable.
- Limpiar cache después de cada PR de fork con `gh cache delete`.

### Pinear acciones a SHA, no a tag

```yaml
# ❌ tag puede ser repointed por el atacante (caso Trivy: 76 tags repointed)
- uses: actions/checkout@v6

# ✅ SHA inmutable
- uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v6.0.2
```

Renovate puede mantener los SHAs actualizados con `pinDigests: true`.

### Permisos mínimos por job

```yaml
permissions: {}  # default: NADA, opt-in explícito
jobs:
  build:
    permissions:
      contents: read  # solo lo que necesita
```

---

## Capa 4: Publish path (OIDC trusted publishing)

Aplica solo si publicas paquetes propios al registry. Si solo consumes, omítelo.

### OIDC vs PAT clásico

| | PAT clásico | OIDC trusted publishing |
|---|---|---|
| Vida del token | Largo plazo (90 días, 1 año) | Minutos |
| Robable de máquina del maintainer | Sí | No (se mintea en CI) |
| Robable de runner CI | Sí, si exfiltras | Sí (memoria del runner, como TanStack) |
| Audit trail por publicación | Difícil | Sí, con run_id y workflow |

OIDC es claramente mejor pero **no es inmune** (el ataque a TanStack lo demuestra). La defensa real es **pinear OIDC a branch + workflow específicos**:

```yaml
# Configuración de trusted publisher en npmjs.com:
Repository: gicupc/junior-doc-gen-tests-cursor
Workflow:   .github/workflows/release.yml  # ✅ NO solo "release.yml"
Branch:     refs/heads/main                 # ✅ NO solo "main"
```

Sin `Workflow` y `Branch` pineados, **cualquier workflow** del repo puede publicar usando la identity OIDC. Con el pinning, solo `release.yml` corriendo desde `main` puede publicar.

### Branch protection sobre `main`

- Require PR review (al menos 1 reviewer humano).
- Require status checks (tests passing).
- Restrict who can push (solo CODEOWNERS).
- Require signed commits si el proyecto es crítico.
- **Block force pushes**: el ataque a TanStack incluyó force-pushes para borrar el commit malicioso del fork.

### 2FA obligatorio en cuentas

- npm: `npm profile enable-2fa auth-and-writes`.
- GitHub: 2FA con TOTP o hardware key (NO SMS).
- Recovery codes guardados en gestor de passwords, NO en el disco.

---

## Respuesta a incidente (si ya pasó)

Si sospechas o confirmas que instalaste un paquete comprometido:

### Paso 1: NO revoques el token de GitHub todavía

⚠ **Crítico**: algunos payloads (Mini Shai-Hulud, TanStack) instalan un daemon de persistencia que monitoriza tu token y ejecuta `rm -rf ~/` cuando detecta revocación. Antes de tocar credenciales, busca y elimina el daemon.

### Paso 2: Buscar y eliminar persistencia

```bash
# macOS
ls -la ~/Library/LaunchAgents/com.user.gh-token-monitor.plist
launchctl unload ~/Library/LaunchAgents/com.user.gh-token-monitor.plist 2>/dev/null
rm ~/Library/LaunchAgents/com.user.gh-token-monitor.plist

# Linux
ls -la ~/.config/systemd/user/gh-token-monitor.service
systemctl --user stop gh-token-monitor 2>/dev/null
systemctl --user disable gh-token-monitor 2>/dev/null
rm ~/.config/systemd/user/gh-token-monitor.service

# Windows
Get-ScheduledTask | Where-Object {$_.TaskName -like "*gh-token*"}
# si aparece algo, desinstalar con Unregister-ScheduledTask
```

También buscar payloads residuales:

```bash
# macOS / Linux
find ~/.claude ~/.vscode -name "router_runtime.js" -o -name "setup.mjs" 2>/dev/null
find / -name "tanstack_runner.js" 2>/dev/null

# Windows
Get-ChildItem -Path $env:USERPROFILE -Recurse -Include "router_runtime.js","setup.mjs","tanstack_runner.js" -ErrorAction SilentlyContinue
```

### Paso 3: Aislar la máquina de la red

Desconectar el Wi-Fi / Ethernet para cortar exfiltración. Si el ataque usa Session (Oxen) para C2, bloquear no es trivial: el aislamiento físico es lo más seguro.

### Paso 4: Rotar credenciales DESPUÉS de limpiar persistencia

Asumir que TODA credencial accesible desde la máquina comprometida está expuesta:

- npm tokens: `npm token list` → revocar todos los desconocidos.
- GitHub PATs: Settings → Developer settings → Personal access tokens → revocar.
- AWS / GCP / Azure: rotar Access Keys y Service Account keys.
- Kubernetes / Vault / SSH keys: rotar.
- CI/CD secrets (GitHub Actions, GitLab CI, etc.): rotar todos.
- Si la cuenta puede publicar paquetes: revisar versiones publicadas en ese rango y deprecar las sospechosas.

### Paso 5: Auditar el blast radius

```bash
# Versiones de paquetes en lockfile
grep -E '@tanstack|@uipath|@mistralai' pnpm-lock.yaml package-lock.json yarn.lock 2>/dev/null

# Si encuentras coincidencias, comparar fechas:
npm view @tanstack/react-router time --json | jq

# Lista oficial de paquetes comprometidos (TanStack, mayo 2026):
# https://github.com/advisories/GHSA-g7cv-rxg3-hmpx
```

### Paso 6: Reportar y documentar

- Reportar al team de seguridad si trabajas en empresa.
- Documentar en `docs/decisions-log.md` como ADR de incidente con: fecha, paquetes afectados, credenciales rotadas, lecciones aprendidas.
- Si publicas paquetes propios y has corrido CI con la versión maliciosa, asumir que **tu pipeline también está comprometido** (worm self-propagation) y revisar versiones publicadas recientemente.

---

## Integración con el harness Architect-Brain

- **`On(testing_setup)`** detecta el package manager y aplica los defaults de la Capa 1 + Capa 2 al configurar el proyecto (paso nuevo en v4.6).
- **`On(supply_chain_setup)`** (nuevo en v4.6) — disparado automáticamente desde `On(testing_setup)` cuando el proyecto usa npm registry. Configura `minimumReleaseAge`, `allowBuilds`, hardening básico de GitHub Actions, registra ADR.
- **`On(supply_chain_audit)`** (nuevo en v4.6) — disparado automáticamente desde `/regularizar` opción [4] cuando hay `package.json`. Audita lockfile, scripts, workflows.
- **`/revisar-supply-chain`** (nuevo en v4.6) — slash command equivalente al protocolo de audit, sin modificar nada.
- **`On(task_complete)`** verifica que actualizaciones de dependencias no han desactivado defensas (cambios en `.npmrc`, `pnpm-workspace.yaml`, etc).
- **`/sync`** detecta drift en configuración de supply chain (un developer desactivó `minimumReleaseAge` y no lo documentó).

---

## Checklist obligatorio antes de mergear cambios en dependencias

Code review obligatoria cuando un PR toca `package.json`, lockfiles o configuración del package manager:

- [ ] El PR no añade `--ignore-scripts=false` ni `minimumReleaseAge: 0` sin justificación documentada.
- [ ] El PR no añade paquetes con `< 1 día de vida` (cooldown).
- [ ] El PR no añade paquetes desde URLs exóticas (`github:`, tarball URLs) sin ADR.
- [ ] Si el PR añade un paquete a `allowBuilds`, hay justificación de por qué necesita ejecutar scripts.
- [ ] El lockfile commiteado es coherente (no editado a mano).
- [ ] Si toca workflows en `.github/workflows/`, no introduce patrón `pull_request_target` con checkout del fork.

---

## Antipatrones a evitar

- **Desactivar `minimumReleaseAge` "porque molesta"**. El día que llegue el zero-day, te tocará el reloj.
- **Añadir paquetes a `minimumReleaseAgeExclude` sin pensar**. Solo tipos puros y toolchain confiable (TypeScript, ESLint, Prettier, Biome). Nunca librerías de runtime.
- **Usar `npm install` en CI**. Regenera el lockfile y abre la puerta a versiones nuevas no auditadas. Siempre `npm ci` / `--frozen-lockfile`.
- **Confiar en SLSA provenance como sello de seguridad**. El ataque TanStack lo demuestra: el provenance era válido. Provenance es necesario, no suficiente.
- **Revocar token de GitHub sin limpiar persistencia primero**. Es el trigger del `rm -rf ~/`. Limpiar daemon → desconectar red → rotar.
- **Usar `pull_request_target` con checkout del fork**. NUNCA. Si necesitas labels/comentarios, usa `github-script` sin checkout.
- **Pinear acciones a tags en lugar de SHA**. El atacante puede repointear el tag (caso Trivy).
- **Compartir secrets de producción con workflows que corren código de PRs**. Trust boundary roto.
- **Mantener PATs de larga duración cuando OIDC está disponible**. Migrar a trusted publishing.

---

## Regla de oro final

> **Asume que cualquier paquete que instalas hoy puede haber sido secuestrado hoy mismo. Defiende en 4 capas, no en 1.**

`minimumReleaseAge` (resolución) + `allowBuilds` (ejecución) + GitHub Actions hardening (CI) + OIDC pinning (publish) es lo que separa un proyecto en 2026 de un proyecto en 2022. Las cuatro capas son baratas de activar y eliminan el 95% del riesgo. La capa 5 (paranoia plena: air-gap, registros privados, mirroring) queda para casos que ya saben que la necesitan.
