---
name: regularizar
description: Tomar control de un proyecto existente sin documentacion. Escanea codigo y BD, ofrece menu de opciones, genera blueprints, tests y auditoria de BD si aplica. La opcion [4] hace todo el ciclo completo de regularizacion + tests + auditoria.
disable-model-invocation: true
---

# /regularizar — Proyecto existente

Voy a tomar control de un proyecto que ya tiene codigo pero documentacion incompleta o inexistente.

Pasos:

1. Escanea el proyecto actual:
   - Estructura de carpetas.
   - Archivos fuente principales (ignora `node_modules`, `vendor`, `dist`, `build`).
   - `package.json` / `composer.json` para detectar stack y dependencias.
   - Tests existentes si los hay.
   - **Detecta si hay base de datos**: schema declarativo (`schema.prisma`, modelos Eloquent, etc.), migraciones, conexiones en codigo, dependencias tipo `pg`, `mysql2`, `mongodb`.

2. Presenta al usuario un resumen de lo detectado:
   - Stack tecnico identificado.
   - Modulos principales.
   - Framework de tests (si existe) y su estado.
   - **Base de datos detectada (si aplica): motor, ORM, numero de migraciones existentes**.
   - Documentacion existente en `/docs/` y que falta.

3. Ofrece el menu de 4 opciones:
   ```
   Segun lo detectado, puedes:
   [1] Regularizar documentacion — generar blueprints sin tests.
   [2] Evolucionar — anadir funcionalidad nueva (usa /nueva mejor).
   [3] Reparar — corregir un bug puntual (usa /reparar mejor).
   [4] Regularizar + Cubrir con tests + Auditar BD — blueprints + setup de tests + tests para logica critica + auditoria de BD si aplica.
   
   ¿Que opcion eliges?
   ```

4. Segun la eleccion:

   **Si elige [1] Regularizar:**
   - Crea `docs/blueprints.md` extrayendo patrones reales del codigo.
   - Crea `docs/architecture.md` documentando la estructura actual.
   - Crea `docs/prd.md` con lo que puedas inferir (preguntale al usuario si no esta claro).
   - Crea `docs/user-stories.md` identificando las funcionalidades existentes.
   - Crea `docs/roadmap.md` con tickets retroactivos marcados `[x]` para lo ya construido.
   - Crea `docs/decisions-log.md` registrando las decisiones tecnicas inferibles (eleccion de stack, patron arquitectonico).
   - Si hay BD: crea tambien `docs/database-strategy.md` documentando lo que se infiere del schema actual (motor, ORM, tablas principales, audit fields presentes/ausentes). Sin auditar todavia.

   **Si elige [4] Regularizar + Cubrir con tests + Auditar BD:**
   - Todo lo de [1], mas:
   - Carga `@.cursor/skills/tests-skill/SKILL.md` como guia.
   - **Si detectas codigo legacy sospechoso o divergencias con contrato formal, carga ADICIONALMENTE `@.cursor/skills/legacy-testing-skill/SKILL.md`** para distinguir SPEC vs CHARACTERIZATION en los tests generados.
   - Identifica "logica critica" (validadores, servicios, reglas de negocio) ignorando fontaneria.
   - Detecta o propone framework de tests segun el stack.
   - Instala dependencias (framework + soporte de linter) y crea configuracion.
   - Genera tests para la logica critica priorizada.
   - Documenta en `docs/testing-strategy.md` que esta cubierto y que no.
   - Anade tickets al roadmap por cada zona NO cubierta para abordar mas adelante.
   - **Si el proyecto tiene base de datos**, ejecuta tambien `On(database_audit)`:
     - Carga `@.cursor/skills/database-skill/SKILL.md`.
     - Realiza las 4 auditorias (modelo, seguridad, rendimiento, calidad de datos).
     - Si esta disponible un MCP de BD, usalo para inspeccion vivo. Si no, analisis estatico desde codigo y schema.
     - Genera `docs/database-audit.md` con el reporte ejecutivo.
     - Genera `docs/investigacion_seguridad.md` si hay hallazgos de seguridad.
     - Crea tickets en `roadmap.md` por cada hallazgo critico o de severidad alta.
     - **NO arregles nada automaticamente**. La IA detecta, el humano decide, el humano repara con `/reparar`.

5. Al terminar, ejecuta `On(task_complete)` mostrando el checklist completo (incluyendo `database-strategy.md` y `database-audit.md` si aplica).

No toques el codigo existente excepto lo estrictamente necesario para instalar/configurar tests.
NUNCA modifiques el schema de BD ni los datos directamente desde aqui.
