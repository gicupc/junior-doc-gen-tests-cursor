---
name: onboarding
description: Mapa rapido de un codebase desconocido sin generar docs ni tocar nada. Lectura y reporte en 6 secciones (stack, capas, traza de un endpoint de referencia, modelo de datos, tests, convenciones). Cierra con TODOs verificables. NO genera docs ni codigo. Es la fase 0 antes de decidir si adoptar un proyecto. Usar para ejercicios de bootcamp, PRs de companeros, debugs urgentes en codigo ajeno.
disable-model-invocation: true
---

# /onboarding — Mapa rapido de codebase desconocido

Voy a entrar a un proyecto que no conozco y necesito un mapa accionable antes de planificar nada. **NO generes docs**, **NO toques codigo**. Solo lectura y reporte.

Diferencia con `/regularizar`:
- `/regularizar` → proyecto que adopto a largo plazo. Crea docs completos en `/docs/`, blueprints, ADRs, plan de cobertura. Es un compromiso.
- `/onboarding` → tarea puntual sobre codigo ajeno (ejercicio del bootcamp, PR de un companero, debug urgente). Solo necesito entender lo justo para actuar.

Ejecuta estas acciones en orden:

1. Carga `@.cursor/skills/prompt-skill/SKILL.md` para recordar el patron RACEO de respuesta.

2. **Detecta el stack** rapidamente:
   - Lee `package.json` / `composer.json` / `requirements.txt` / `Cargo.toml` para identificar lenguaje, framework principal, ORM y libreria de tests.
   - Anota la version del runtime si esta declarada.

3. **Mapea las capas existentes** explorando estructura de carpetas:
   - Si hay `src/` con subcarpetas tipo `controllers/`, `services/`, `models/`, `routes/`, `domain/`, `application/`, `infrastructure/`, `presentation/` → reporta el patron arquitectonico (capas, hexagonal, clean architecture, MVC, Active Record, etc.).
   - Si no hay subcarpetas claras → reporta que es flat y advierte.

4. **Traza un endpoint o feature de referencia** desde la entrada hasta la persistencia. Devuelve la cadena exacta de archivos:
   ```
   routes/X.ts -> controller/Y.ts -> service/Z.ts -> domain/W.ts -> BD
   ```
   Anota el rol de cada capa en una linea. **Si encuentras un antipatron** (ej. ruta que llama directamente al servicio bypaseando el controller, o controller huerfano sin invocar), reportalo explicitamente — esto condiciona como hay que escribir codigo nuevo.

5. **Modelo de datos minimo**:
   - Si hay schema (Prisma, SQLAlchemy, migraciones SQL): lista las entidades clave con PK + FK + 2-3 atributos importantes.
   - Si hay relaciones complejas relevantes: bloque aparte en formato `A -[1:N|N:1|N:M]- B`.
   - **No pegues el schema entero**. Solo lo necesario para razonar.

6. **Tests existentes**:
   - Donde viven (`tests/`, `__tests__/`, `spec/`).
   - Framework usado.
   - Estrategia de mocks de dependencias externas (BD, APIs, FS).
   - Si NO hay tests, decirlo claramente — implica que cualquier feature nueva necesita configurar el patron de mocks desde cero.

7. **Convenciones detectadas**:
   - Naming (camelCase / PascalCase / snake_case por carpeta).
   - Validacion de inputs (libreria como Zod / validator manual / sin validacion).
   - Manejo de errores (try/catch en controller / middleware global / mixto / inconsistente).
   - Una frase por convencion, no parrafos.

8. **TODOs para el usuario**: cierra con 3 puntos concretos que debe verificar manualmente antes de planificar trabajo:
   - Decisiones arquitectonicas que no quedan claras solo del codigo.
   - Areas con antipatrones donde habra que decidir si propagar o no.
   - Cualquier cosa que el usuario tenga que confirmar con stakeholders o documentacion externa.

## Restricciones

- **No generes docs en `/docs/`**. No es `/regularizar`.
- **No propongas refactors ni mejoras**. Es onboarding, no review.
- **No escribas codigo**.
- **No leas mas de lo necesario**: si una seccion la puedes responder con 2 archivos, no leas 20.
- **Si una seccion no la puedes responder con confianza**, escribe "no encontrado" y di donde miraste — no inventes.

## Output esperado

Markdown con 6 secciones (stack, capas, traza de referencia, modelo de datos, tests, convenciones), maximo 15 lineas por seccion. Cierra con bloque `## TODOs para mi` con 3 puntos.

---

## Cuando usar este workflow

Casos tipicos:

- Acabo de clonar un repo y tengo que hacer un PR pequeno (ejercicio del bootcamp).
- Voy a revisar el codigo de un companero antes de un code review.
- Tengo que arreglar un bug urgente en un proyecto que no es mio.
- Quiero entender rapido si vale la pena adoptar este codebase como propio (en cuyo caso pasaria luego a `/regularizar`).

Si despues del onboarding decides adoptar el proyecto, ejecuta `/regularizar` para generar los docs completos. El onboarding **no** es sustitutivo: es el paso 0.
