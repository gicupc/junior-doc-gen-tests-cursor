---
name: inicio
description: Arrancar proyecto nuevo desde cero. Lanza la entrevista BMADT, genera todos los docs del SSOT, ejecuta setup de tests y, si aplica, setup de base de datos. Bloquea escritura de codigo hasta aprobacion.
disable-model-invocation: true
---

# /inicio — Proyecto nuevo

Voy a arrancar un proyecto nuevo siguiendo el protocolo Architect-Brain.

Ejecuta estas acciones en orden:

1. **Detecta el estado de `/docs/`** antes de cualquier otra cosa:
   - Si `/docs/` esta vacio o no existe → procede como proyecto nuevo (continua al paso 2).
   - Si `/docs/` tiene archivos y TODOS los `.md` empiezan con `<!-- TEMPLATE-PLACEHOLDER -->` como primera linea → son templates del scaffolding sin rellenar. Procede como proyecto nuevo (continua al paso 2). Los templates se sobrescribiran al rellenarlos.
   - Si `/docs/` tiene archivos y al menos uno NO tiene el marcador `<!-- TEMPLATE-PLACEHOLDER -->` → HAY TRABAJO PREVIO. NO arranques BMADT automaticamente. Muestra al usuario:
     ```
     ⚠️ Aviso antes de ejecutar /inicio

     Detecto archivos en /docs/ SIN el marcador <!-- TEMPLATE-PLACEHOLDER -->.
     Eso significa que tienen contenido real, no son templates del scaffolding.

     /inicio sobrescribiria ese contenido. ¿Como procedo?
     - [1] Cancelar /inicio y leer los docs existentes (recomendado).
     - [2] Ejecutar /onboarding para entender el codebase.
     - [3] Ejecutar /sync para auditar drift.
     - [4] Reset total: archivar /docs/ en /docs-archive-YYYYMMDD/ y arrancar BMADT desde cero (DESTRUCTIVO).
     ```
     Espera la respuesta del usuario antes de seguir.

2. Carga `@prompts/architect-brain.md`.

3. Ejecuta el protocolo `On(project_start)`:
   - Lanza la entrevista BMADT (Beneficio, Mecanismo, Alcance, Datos, Testing).
   - Haz las 5 preguntas UNA A UNA, no todas de golpe. Espera a que responda cada una antes de pasar a la siguiente.

4. Al cerrar la entrevista:
   - Genera los docs en `/docs/`: prd.md, architecture.md, user-stories.md, roadmap.md, testing-strategy.md, decisions-log.md.
   - **IMPORTANTE: al rellenar cada archivo, ELIMINA la primera linea `<!-- TEMPLATE-PLACEHOLDER -->`**. Una vez eliminado el marcador, el archivo deja de considerarse template y queda como SSOT real del proyecto.
   - Registra las respuestas BMADT como ADRs iniciales en `docs/decisions-log.md`.

5. Ejecuta `On(testing_setup)` ANTES del primer codigo funcional:
   - Instala framework de tests segun el stack elegido en M.
   - Instala soporte de linter del framework (phpstan-phpunit, @types/jest, etc.).
   - Crea configuracion minima.
   - Verifica con smoke test en verde y borralo.
   - Marca como ticket inicial `[x]` en roadmap.md.

6. **Si la respuesta M (Mecanismo) o D (Datos) implica base de datos**, ejecuta tambien `On(database_setup)`:
   - Carga `@.cursor/skills/database-skill/SKILL.md`.
   - Detecta motor de BD apropiado (Postgres por defecto, salvo razon especifica).
   - Detecta ORM apropiado segun stack (Prisma, Eloquent, SQLAlchemy, etc.).
   - **Pregunta al usuario que entidades de la respuesta D contienen datos personales (PII)** y como debe protegerse cada uno.
   - Genera `docs/database-strategy.md` rellenando la plantilla con todas las decisiones (recuerda eliminar el marcador TEMPLATE-PLACEHOLDER de la primera linea).
   - Configura entorno: pg_stat_statements para Postgres, Testcontainers si el nivel de testing es 2 o 3.
   - Registra ADRs por cada decision (motor, ORM, PII).
   - Anade tickets al `roadmap.md`: `[x] Setup database environment` y `[ ] Definir y revisar schema inicial`.

7. **VERIFICACION FINAL OBLIGATORIA — antiresiduo de templates**:
   Antes de mostrar el resumen final al usuario, ejecuta esta verificacion automatica:
   - Para cada archivo en `/docs/*.md`, comprueba que la primera linea NO sea `<!-- TEMPLATE-PLACEHOLDER -->`.
   - Comando equivalente: `head -1 docs/*.md | grep -c "TEMPLATE-PLACEHOLDER"` debe devolver `0`.
   - Si encuentras algun archivo que SIGUE empezando con el marcador (porque te lo dejaste al rellenar): elimina el marcador inmediatamente y avisa al usuario en el resumen final con `⚠️ Reparado: marcador residual en docs/X.md eliminado tras la generacion.`
   - Esta verificacion NO es opcional. Un solo doc con marcador residual hace que el sistema vuelva a tratar el proyecto como "template sin rellenar" la proxima vez que alguien ejecute /inicio, perdiendo el trabajo previo.

8. Muestra un resumen final con todos los docs generados y los tickets iniciales, y pregunta al usuario si aprueba antes de empezar con el primer ticket funcional.

No escribas codigo funcional hasta que el usuario apruebe los docs, el entorno de tests, y el setup de BD si aplica.
