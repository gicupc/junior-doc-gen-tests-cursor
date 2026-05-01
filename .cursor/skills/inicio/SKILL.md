---
name: inicio
description: Arrancar proyecto nuevo desde cero. Lanza la entrevista BMADT, genera todos los docs del SSOT, ejecuta setup de tests y, si aplica, setup de base de datos. Bloquea escritura de codigo hasta aprobacion.
disable-model-invocation: true
---

# /inicio — Proyecto nuevo

Voy a arrancar un proyecto nuevo siguiendo el protocolo Architect-Brain.

Ejecuta estas acciones en orden:

1. Carga `@prompts/architect-brain.md`.
2. Ejecuta el protocolo `On(project_start)`:
   - Lanza la entrevista BMADT (Beneficio, Mecanismo, Alcance, Datos, Testing).
   - Haz las 5 preguntas UNA A UNA, no todas de golpe. Espera a que responda cada una antes de pasar a la siguiente.
3. Al cerrar la entrevista:
   - Genera los docs en `/docs/`: prd.md, architecture.md, user-stories.md, roadmap.md, testing-strategy.md, decisions-log.md.
   - Registra las respuestas BMADT como ADRs iniciales en `docs/decisions-log.md`.
4. Ejecuta `On(testing_setup)` ANTES del primer codigo funcional:
   - Instala framework de tests segun el stack elegido en M.
   - Instala soporte de linter del framework (phpstan-phpunit, @types/jest, etc.).
   - Crea configuracion minima.
   - Verifica con smoke test en verde y borralo.
   - Marca como ticket inicial `[x]` en roadmap.md.
5. **Si la respuesta M (Mecanismo) o D (Datos) implica base de datos**, ejecuta tambien `On(database_setup)`:
   - Carga `@.cursor/skills/database-skill/SKILL.md`.
   - Detecta motor de BD apropiado (Postgres por defecto, salvo razon especifica).
   - Detecta ORM apropiado segun stack (Prisma, Eloquent, SQLAlchemy, etc.).
   - **Pregunta al usuario que entidades de la respuesta D contienen datos personales (PII)** y como debe protegerse cada uno.
   - Genera `docs/database-strategy.md` rellenando la plantilla con todas las decisiones.
   - Configura entorno: pg_stat_statements para Postgres, Testcontainers si el nivel de testing es 2 o 3.
   - Registra ADRs por cada decision (motor, ORM, PII).
   - Anade tickets al `roadmap.md`: `[x] Setup database environment` y `[ ] Definir y revisar schema inicial`.
6. Muestra un resumen final con todos los docs generados y los tickets iniciales, y pregunta al usuario si aprueba antes de empezar con el primer ticket funcional.

No escribas codigo funcional hasta que el usuario apruebe los docs, el entorno de tests, y el setup de BD si aplica.
