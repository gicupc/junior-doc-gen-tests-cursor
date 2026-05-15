<!-- TEMPLATE-PLACEHOLDER -->
# Supply Chain Strategy

> Este documento lo rellena el Architect-Brain durante el protocolo `On(supply_chain_setup)` o al elegir la opción [4] en proyectos existentes con `package.json`.
> No editar a mano sin avisar a la IA — es parte del SSOT.

---

## Resumen ejecutivo

*Una línea con el estado general de las defensas. Ejemplo:*
- Capas activas: 4/4 (Resolution, Execution, CI, Publish).
- Package manager: pnpm 11.
- Última auditoría: 15 de mayo de 2026.
- Incidentes públicos cross-referenciados: ninguno detectado.

---

## Capa 1: Dependency resolution

### Package manager
*Ejemplo:* **pnpm 11.0.5** declarado en `packageManager` field de `package.json`.

### Cooldown de release (`minimumReleaseAge`)
- Valor activo: *1440 minutos (1 día) — default pnpm 11*.
- Excepciones (`minimumReleaseAgeExclude`):
  - `@types/*` — tipos puros, riesgo mínimo.
  - `typescript` — toolchain confiable.

### Bloqueo de fuentes exóticas
- `blockExoticSubdeps: true` — activado (default pnpm 11).
- Direct dependencies con URLs exóticas (`github:`, tarball URLs): *ninguna* o lista justificada con ADR.

### Trust policy
- `trustPolicy: no-downgrade` — *activado / no activado*.

---

## Capa 2: Install-time execution

### Política de lifecycle scripts
*Ejemplo:* deny by default (`strictDepBuilds: true`), aprobaciones explícitas en `allowBuilds`.

### Paquetes en `allowBuilds`
| Paquete | Razón |
|---|---|
| `electron` | Binarios nativos por plataforma |
| `esbuild` | Binario nativo para parseo rápido |
| `sharp` | libvips nativo para procesado de imagen |
| `@swc/core` | Compilador Rust |

### Ejecución de scripts en CI
- `npm ci` / `pnpm install --frozen-lockfile` / `yarn install --immutable` — confirmar.
- NUNCA `npm install` a secas en CI.

---

## Capa 3: CI execution (GitHub Actions)

### Workflows auditados
*Lista de los workflows en `.github/workflows/` con su estado:*

| Workflow | Trigger | Riesgo | Estado |
|---|---|---|---|
| `ci.yml` | `pull_request` | Bajo | ✅ Patrón seguro |
| `release.yml` | `push: main` | Alto | ✅ OIDC + branch pineado |

### Patrones prohibidos verificados
- [ ] NO hay `pull_request_target` con checkout del fork.
- [ ] Acciones de terceros pineadas a SHA, no a tag.
- [ ] `permissions: {}` por defecto en jobs.
- [ ] Secrets NO accesibles desde workflows que corren código de PR.
- [ ] Cache no compartida entre PR workflows y release workflow.

---

## Capa 4: Publish path

*Rellenar solo si el proyecto publica paquetes al registro.*

### Trusted publisher (OIDC)
- Proveedor: npmjs.com / GitHub Packages.
- Workflow pineado: `.github/workflows/release.yml`.
- Branch pineada: `refs/heads/main`.
- ¿OIDC en lugar de PAT?: *sí / no*.

### Branch protection en `main`
- [ ] Require PR review.
- [ ] Require status checks (tests passing).
- [ ] Block force pushes.
- [ ] Restrict who can push (CODEOWNERS).

### 2FA y credenciales
- [ ] 2FA activado en npm (`auth-and-writes`).
- [ ] 2FA activado en GitHub (TOTP o hardware key, NO SMS).
- [ ] Recovery codes guardados en gestor de contraseñas.

---

## Renovate / Dependabot

*Si el proyecto usa bot de actualizaciones:*

- Cooldown configurado: *3 días en Renovate / 3 días en Dependabot*.
- Excepciones para security updates: *sí, security updates fast-track*.

---

## Plan de respuesta a incidente

En caso de detectar paquete comprometido instalado:

1. **NO revocar token de GitHub todavía** (algunos payloads ejecutan `rm -rf ~/` al detectar revocación).
2. Buscar y eliminar daemon de persistencia (`gh-token-monitor` en `~/Library/LaunchAgents/` macOS, `~/.config/systemd/user/` Linux, Task Scheduler en Windows).
3. Buscar payloads residuales (`router_runtime.js`, `setup.mjs`, `tanstack_runner.js`) en `~/.claude`, `~/.vscode`, home directory.
4. Aislar la máquina de la red (desconectar Wi-Fi/Ethernet).
5. Rotar TODAS las credenciales accesibles desde la máquina: npm tokens, GitHub PATs, AWS/GCP/Azure keys, Kubernetes/Vault, SSH keys, CI/CD secrets.
6. Auditar versiones publicadas por la cuenta en el rango de la compromisión (worm self-propagation).
7. Documentar en `docs/decisions-log.md` como ADR de incidente.

---

## Excepciones documentadas

*Si alguna defensa fue desactivada o relajada, razón aquí.*

Ejemplo:
- `minimumReleaseAge: 0` durante migración inicial — revertido tras estabilizar el lockfile (ADR-014).

---

## Próximas revisiones

*Cuándo y cómo se re-audita la cadena de suministro:*

- Cada release a producción → ejecutar `/revisar-supply-chain`.
- Tras cualquier incidente público de npm (TanStack, Shai-Hulud, etc) → cross-referenciar lockfile y rotar si aplica.
- Mensualmente como parte de la rutina de seguridad.

---

## Referencias

- Skill: `.cursor/skills/supply-chain-skill/SKILL.md`.
- Workflow de auditoría: `.cursor/skills/revisar-supply-chain/SKILL.md`.
- Incidentes 2025-2026 conocidos: Shai-Hulud (sep 2025), axios (mar 2026), Trivy (mar 2026), Bitwarden (abr 2026), SAP (abr 2026), TanStack Mini Shai-Hulud (11 may 2026).
