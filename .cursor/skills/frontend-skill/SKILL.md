---
name: frontend-skill
description: Diseno y auditoria de frontend. Usar cuando el proyecto tenga frontend - en proyectos nuevos para configurar el stack canonico 2026 (Next.js + TypeScript estricto + Tailwind v4 + shadcn/ui + Biome + Zod) con accesibilidad WCAG 2.2 AA y Core Web Vitals desde dia 1, en proyectos existentes para auditar stack/seguridad/CWV/a11y. Cubre MCP servers (Figma Dev Mode, shadcn, Chrome DevTools, Playwright).
---

# Skill: Frontend (diseño y auditoría)

Esta skill se activa cuando el proyecto tiene frontend. Cubre dos escenarios:

1. **Proyecto NUEVO**: cómo arrancar un frontend siguiendo el stack convergente de 2026, configurando TypeScript estricto, accesibilidad desde día 1 y los MCP servers que aceleran el flujo diseño-a-código.
2. **Proyecto EXISTENTE**: cómo auditar un frontend que ya está hecho — detectar problemas de rendimiento real (Core Web Vitals), accesibilidad (WCAG 2.2 AA), seguridad (secretos en cliente, validación ausente) y mantenibilidad (componentes gigantes, `any` extendido).

---

## Principio fundamental

**El frontend es donde el usuario y la accesibilidad legal viven.** Un bug de backend lo ve un log; un bug de frontend lo sufre cada usuario y, desde el 28 de junio de 2025, puede ser objeto de sanción legal en la UE bajo la European Accessibility Act. Por eso:

- En proyectos nuevos: arranca con TypeScript estricto, accesibilidad WCAG 2.2 AA y un stack convergente que la IA conoce bien.
- En proyectos existentes: audita Core Web Vitals reales, no Lighthouse local. Audita accesibilidad con axe-core en CI más prueba manual con lector de pantalla.

---

## ESCENARIO 1: Proyecto nuevo

### Decisiones obligatorias antes de escribir código

Cuando la entrevista BMADT (respuesta M = Mecanismo) menciona frontend, tomar estas decisiones explícitamente y documentarlas:

#### 1. Stack convergente recomendado (2026)

| Capa | Recomendación por defecto | Cuándo desviarse |
|---|---|---|
| Framework | **Next.js 15 / 16** (App Router, RSC, Server Actions) | Astro si es sitio de contenido/marketing (envía cero JS por defecto). SvelteKit si bundle mínimo es crítico. |
| Lenguaje | **TypeScript estricto** (`strict: true`, `noUncheckedIndexedAccess: true`) | Nunca JavaScript plano para proyecto nuevo. |
| Estilado | **Tailwind CSS v4** (config CSS-first, container queries en core, motor Oxide) | CSS Modules si ya hay design system propio en CSS. |
| Componentes | **shadcn/ui** (copy-in, no instalable, basado en Radix + Tailwind) | MUI si proyecto enterprise con catálogo amplio. HeroUI si quieres ligero sobre React Aria. |
| Forms y validación | **react-hook-form + Zod** (schemas compartibles front/back) | Valibot si bundle mínimo es crítico. |
| i18n (preventivo) | **next-intl** (Next.js) o **paraglide** (type-safe, tree-shakeable) | i18next si proyecto multi-framework. |
| Linter/format | **Biome v2** (todo en uno, ~35× más rápido) o **ESLint 9 flat config + Prettier** | Biome para proyectos nuevos; ESLint si necesitas reglas type-aware muy específicas. |
| Bundler | Por defecto del meta-framework (**Turbopack** en Next 16, **Vite** en el resto) | No tocar salvo razón fuerte. |

Por defecto: **Next.js 15/16 + TypeScript + Tailwind v4 + shadcn/ui + Biome + Zod**. Cualquier desviación se registra como ADR.

Razón: este stack es el que las herramientas IA (Cursor, Claude Code, v0, Lovable, Bolt) generan mejor por defecto en 2026.

#### 2. Estructura de carpetas (Next.js App Router)

```
app/                    # Routing + páginas (RSC por defecto)
  (marketing)/
  (app)/
    layout.tsx
    page.tsx
components/
  ui/                   # Primitivas shadcn
  shared/               # Componentes compuestos
  features/             # Componentes de dominio (por feature)
hooks/
lib/
  utils.ts
  validations/          # Schemas Zod compartidos
types/
public/
tests/
  unit/
  integration/
  e2e/
  visual/
```

Regla: si una carpeta supera ~10 archivos, considerar partición por feature.

#### 3. TypeScript estricto desde día 1

`tsconfig.json` mínimo aceptable:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true
  }
}
```

Regla del proyecto: **cero `any` en código de producción**. Si aparece un `any`, debe documentarse el motivo y tener un ticket de eliminación en `roadmap.md`. Tipo `unknown` con narrowing es preferible.

Razón: los LLMs generan código mucho más preciso cuando leen tipos. Un `any` rompe ese contrato.

#### 4. Componentes pequeños y de responsabilidad única

- **Tamaño máximo orientativo**: ~150 líneas por componente. Si supera, dividir.
- **Una responsabilidad por componente**. Si maneja "datos + UI + side effects + estado global", extraer custom hooks y subcomponentes.
- **Custom hooks** para lógica reusable y testeable por separado.
- **Compound components** para APIs componibles (ejemplo: `<Tabs><Tabs.List/><Tabs.Panel/></Tabs>`).

Antipatrón frecuente generado por IA: un único componente de 400 líneas que mezcla fetching, transformación, formulario y render. Refactor inmediato.

#### 5. Accesibilidad WCAG 2.2 AA desde día 1

**Mínimo legal en la UE desde junio 2025** para muchos productos B2C. No es opcional.

Reglas no negociables del proyecto:
- **Semántica HTML antes que ARIA**. ARIA solo cuando no exista equivalente nativo.
- **Contraste mínimo 4.5:1** texto normal, 3:1 texto grande (≥18pt o 14pt bold).
- **Navegación completa por teclado**, focus visible siempre.
- **Labels en formularios**, `alt` en imágenes informativas, `aria-hidden="true"` en imágenes decorativas.
- **Landmarks**: `<nav>`, `<main>`, `<aside>`, `<header>`, `<footer>` en cada página.
- **`prefers-reduced-motion`** respetado para usuarios sensibles al movimiento.

Setup obligatorio en `On(testing_setup)` cuando hay frontend:
- `eslint-plugin-jsx-a11y` (o equivalente Biome) — detecta ~30% de issues estáticamente.
- `@axe-core/react` o `@axe-core/playwright` para validación runtime.
- `vitest-axe` para tests de accesibilidad por componente.

shadcn/ui parte de primitivas Radix accesibles, pero **lo que tú compones por encima puede romperlo**. Validar con axe-core.

#### 6. Core Web Vitals como criterio de aceptación

| Métrica | Significado | Objetivo |
|---|---|---|
| **LCP** (Largest Contentful Paint) | Tiempo hasta el elemento visible más grande | < 2.5 s |
| **INP** (Interaction to Next Paint) | Responsividad ante clic/tecla (sustituye a FID desde marzo 2024) | < 200 ms |
| **CLS** (Cumulative Layout Shift) | Estabilidad visual (saltos inesperados) | < 0.1 |

Esto entra en el Definition of Done de cualquier ticket que toque rendimiento. Se mide con:
- **Lab**: Lighthouse (DevTools) y PageSpeed Insights (combina lab + CrUX).
- **Real**: Vercel Speed Insights, Cloudflare Web Analytics o Sentry — Real User Monitoring (RUM).

Regla operativa: para la imagen LCP, `fetchpriority="high"` y `preload`. Para el resto, `loading="lazy"`. Bundlers modernos hacen code-splitting automático; no hace falta optimizar a mano.

#### 7. Seguridad frontend (no negociable)

- **Cero secretos en el cliente**. En Next.js, solo variables con prefijo `NEXT_PUBLIC_` son públicas. Auditar `NEXT_PUBLIC_*` que parezcan secretos.
- **Sanitización con DOMPurify** para HTML inyectado. Nunca `dangerouslySetInnerHTML` sin sanitizar.
- **Validación cliente Y servidor con Zod** (schemas compartidos). Cliente es UX; servidor es seguridad.
- **CSP, SRI, Strict-Transport-Security, X-Frame-Options, Referrer-Policy** configurados. Auditar con `securityheaders.com` y `Mozilla Observatory`.
- **`npm audit`, Snyk o Socket.dev** en CI. Bloquear PRs con vulnerabilidades altas/críticas.
- **Cuidado con código generado por IA**: estudios 2025-2026 reportan ~40-45% de vulnerabilidades en outputs de v0/Lovable/Bolt sin revisión.

#### 8. MCP servers recomendados (cuando aplica)

| MCP server | Cuándo activarlo | Qué da |
|---|---|---|
| **Figma Dev Mode MCP** | Equipo de diseño en Figma | `get_code`, `get_image`, `get_variable_defs` (tokens), `get_design_context` (jerarquía + props mapeados) |
| **shadcn MCP** | Stack incluye shadcn/ui | Búsqueda e instalación de bloques (>6.000 disponibles) |
| **Chrome DevTools MCP** | Auditoría DOM/CSS/performance/networking | 26 tools en 6 categorías (oficial Google) |
| **Playwright MCP** | Validación visual y tests E2E | Automatización sobre accessibility tree (determinista) |

Configuración en Cursor: `.cursor/mcp.json` (ya incluye plantillas comentadas para los anteriores). Diferencia clave de Figma MCP vs screenshot-a-código: el LLM ve **tokens estructurados** (spacing, color, typography) y jerarquía. No adivina, lee.

Acceso: requiere Dev o Full seat en planes pagos de Figma.

#### 9. Vibe coding: cuándo SÍ y cuándo NO

- **SÍ usar** v0/Lovable/Bolt para: prototipos clicables, MVPs muy tempranos, validación de hipótesis con stakeholders.
- **NO usar** para: producción seria sin pasar antes por revisión humana en Cursor/Claude Code.
- **Patrón profesional**: prototipo en v0/Lovable → exportar a GitHub → continuar en Cursor con el sistema Architect-Brain → producción.

Honestidad sobre Figma Make (LogRocket, abril 2026): el código no es production-ready. Útil para inspiración y user-testing; no para shipping. Tratar como "validación rápida".

### Setup obligatorio

Tras la entrevista BMADT con frontend en M:

1. Crear `tsconfig.json` con strict mode.
2. Inicializar el linter elegido (Biome o ESLint+Prettier) con reglas de a11y activas.
3. Configurar Tailwind v4 con `@theme` directive y tokens base.
4. Si shadcn: `npx shadcn@latest init` (genera `components.json`, deja primitivas en `components/ui/`).
5. Configurar headers de seguridad (CSP, HSTS, etc.).
6. Configurar `npm audit` o Snyk en CI.
7. Configurar Real User Monitoring si despliegue en plataforma compatible.
8. Generar `docs/frontend-strategy.md`.
9. Registrar ADR del stack y de las decisiones de a11y/CWV.
10. Añadir tickets al `roadmap.md`.

### Loop visual (recomendación)

Para proyectos con diseño de partida (Figma o capturas), montar el loop visual desde el principio:

1. Agente genera código → 2. Playwright toma screenshot → 3. Compara con baseline (pixelmatch) → 4. Ajusta → 5. Re-compara.

Detalle completo en `visual-testing-skill`.

---

## ESCENARIO 2: Proyecto existente — Auditoría

Cuando se invoca `/revisar-frontend` (o se llega aquí desde `/regularizar` opción [4]) sobre un proyecto con frontend ya construido.

### Las 4 auditorías

#### Auditoría 1: Stack y mantenibilidad

- **TypeScript no estricto o ausente** (proyecto en JS plano sin migración planificada).
- **`any` extendido** sin justificación (>5 ocurrencias en código de dominio = ALTA).
- **Componentes gigantes** (>300 líneas, >5 responsabilidades en uno).
- **Estado global mal usado** (Redux/Zustand para datos de un solo componente).
- **Dependencias obsoletas o no auditadas**.
- **Mezcla de librerías de UI** (shadcn + MUI + Chakra = Frankenstein, ALTA).
- **Linter ausente o configuración mínima** (sin reglas de a11y).
- **Tests de componentes ausentes**.

Output: tabla clasificada por severidad con archivos referenciados.

#### Auditoría 2: Seguridad frontend

- **Secretos en cliente**: `NEXT_PUBLIC_*`/`VITE_*`/`PUBLIC_*` con sufijos sospechosos (`*_KEY`, `*_TOKEN`, `*_SECRET`). CRÍTICO si confirma exposición.
- **`dangerouslySetInnerHTML` sin sanitización** previa con DOMPurify. ALTA.
- **Validación solo en cliente**: formularios sin validación servidor. CRÍTICO.
- **Headers de seguridad ausentes**: probar con `securityheaders.com`. Buscar CSP, HSTS, X-Frame-Options, Referrer-Policy.
- **CORS abierto** (`origin: '*'`) en endpoints sensibles. ALTA.
- **Dependencias vulnerables**: `npm audit --production`.
- **Código generado por IA sin revisión**: rastros de v0/Lovable/Bolt con patrones inseguros.

Output: documento `docs/investigacion_seguridad.md` (compartido con auditoría de BD si aplica).

#### Auditoría 3: Rendimiento (Core Web Vitals)

Si hay despliegue accesible:
- Lanzar **PageSpeed Insights** sobre URLs clave.
- Recoger LCP, INP, CLS reales (CrUX) y de laboratorio (Lighthouse).
- Reportar páginas que NO pasan los umbrales.

Si NO hay despliegue accesible — análisis estático:
- **Imágenes sin optimizar**: `<img>` plano en vez de `<Image>` de Next/Astro.
- **Imágenes sin atributo `loading`** y sin `fetchpriority` para LCP.
- **Bundles grandes**: dependencias pesadas innecesarias.
- **Sin code-splitting**: rutas sin `dynamic` import donde aplica.
- **Web fonts sin `font-display: swap`**.
- **Animaciones costosas** que no respetan `prefers-reduced-motion`.

Output: las 3-5 mejoras de mayor impacto, priorizadas.

#### Auditoría 4: Accesibilidad (WCAG 2.2 AA)

Tooling:
- `npx @axe-core/cli https://tu-sitio.com` si hay despliegue.
- `axe-playwright` si hay E2E configurado.
- Análisis estático: leer JSX en busca de patrones problemáticos.

Detectar:
- **Imágenes sin `alt`** (o con `alt=""` cuando deberían tener descripción).
- **Botones sin texto accesible** (solo iconos sin `aria-label`).
- **Inputs sin `<label>`** asociado.
- **Mensajes de error de formulario no asociados al input** (sin `aria-describedby`).
- **Contraste insuficiente**.
- **Navegación rota por teclado** (focus traps).
- **Landmarks ausentes**.
- **Headings desordenados**: `h1` ausente o múltiple.
- **Colores como única fuente de información** (estados solo por color).
- **Animaciones sin respeto a `prefers-reduced-motion`**.

Honestidad: tooling automatizado detecta ~30% de issues. El resto requiere testing manual con NVDA/VoiceOver.

### Filosofía de la auditoría

> **La IA detecta. El humano decide. El humano repara.**

`/revisar-frontend` NUNCA modifica código. Solo genera reportes.

### Output estándar

1. **`docs/frontend-audit.md`**: informe ejecutivo.
2. **`docs/investigacion_seguridad.md`** (compartido con BD).
3. **Tickets en `roadmap.md`**: uno por hallazgo crítico/alto.
4. **Tests de regresión** si aplica.

---

## Referencia rápida del stack 2026

```
Framework:     Next.js 15/16 (App Router, RSC, Server Actions)
Lenguaje:      TypeScript estricto (sin any)
Estilado:      Tailwind v4 (container queries, OKLCH, @theme)
Componentes:   shadcn/ui (Radix + Tailwind, copy-in)
Forms:         react-hook-form + Zod (schemas compartidos front/back)
i18n:          next-intl o paraglide (type-safe)
Linter:        Biome v2 (todo-en-uno) o ESLint 9 + Prettier
Testing:       Vitest/Jest + RTL + jest-axe + Playwright + axe-core
RUM:           Vercel Speed Insights, Cloudflare Web Analytics o Sentry
Deploy:        Vercel, Cloudflare Pages, Netlify
```

---

## Antipatrones a evitar

- **`any` por todas partes**: hace que la IA genere código peor en sucesivas iteraciones.
- **Componentes monolíticos**: 400 líneas con 5 responsabilidades.
- **`dangerouslySetInnerHTML` sin DOMPurify**: vector XSS directo.
- **Variables `NEXT_PUBLIC_*` con secretos**: el prefijo expone al cliente.
- **Validación solo en cliente**: el servidor es la trinchera.
- **Mezcla de librerías de UI** (shadcn + MUI + Chakra).
- **Saltarse axe-core porque "ya parece accesible"**: la mitad de los issues son invisibles.
- **Animaciones sin `prefers-reduced-motion`**: WCAG 2.2 (criterio 2.3.3).
- **Confiar ciegamente en código generado por IA**: 40-45% de vulnerabilidades sin revisión.
- **Optimización prematura**: medir primero (PageSpeed Insights, RUM).

---

## Regla de oro final

> **En proyectos nuevos: TypeScript estricto + accesibilidad WCAG 2.2 AA + Core Web Vitals desde día 1.**
> **En proyectos existentes: medir CWV reales antes de optimizar; auditar a11y con axe + lector de pantalla; nunca aceptar código IA sin revisión de seguridad.**
