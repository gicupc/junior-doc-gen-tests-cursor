---
name: prompt-skill
description: Manual operativo para el USUARIO sobre como redactar prompts profesionales siguiendo el patron RACEO (Role + Objective + Context + Constraints + Expected Output). Usar cuando el usuario quiera redactar prompts complejos para onboarding a codebases ajenos, planificacion de features, implementacion paso a paso, o auditorias. Incluye plantilla base, 3 ejemplos completos, antipatrones a evitar y cuando NO usar RACEO.
---

# Skill: Escritura de Prompts Profesionales (RACEO)

Esta skill enseña al **usuario** cómo redactar prompts de calidad cuando interactúa con el Agent durante el ciclo de un proyecto. No la consume el Agent automáticamente para ejecutar tareas: es un manual operativo para la persona que escribe.

Se usa especialmente durante:
- Onboarding a un codebase desconocido (`/onboarding`).
- Planificación de features complejas (`/nueva`).
- Implementación paso a paso (cuando el usuario reparte trabajo al Agent).
- Auditorías y reviews (`/cuestionar`, `/sync`, `/revisar-bd`).

---

## El patrón RACEO

Un prompt profesional tiene cinco bloques en este orden:

```
# Role          — quién debe ser el Agent en esta tarea
# Objective     — qué resultado concreto quieres
# Context       — qué información comparte la conversacion
# Tasks         — pasos numerados a ejecutar (opcional, segun complejidad)
# Constraints   — qué NO debe hacer el Agent
# Expected Output — formato exacto de la respuesta
```

No siempre necesitas los cinco. Para un prompt corto basta con Role + Objective + Constraints. Para un prompt de implementación los cinco son obligatorios.

---

## Por qué cada bloque

### Role
La diferencia entre "explícame este código" y "Eres un senior haciendo onboarding rápido a un codebase legacy" es enorme. El rol moldea tono, profundidad y prioridades. Sin rol, el Agent tiende a sobre-explicar lo trivial e infra-explicar lo importante.

### Objective
Una frase con el resultado que quieres, no la tarea. "Producir un mapa accionable del módulo de pagos" es un objetivo. "Mira el módulo de pagos" es una tarea sin destino.

### Context
Lo que el Agent necesita saber pero que no está obvio en el código: stack, restricciones del entorno, decisiones previas tomadas, qué partes del código son intocables. **Si en el chat anterior ya quedó claro algo, no lo repitas** — el Agent conserva el contexto del turno previo.

### Tasks
Cuando el prompt requiere múltiples acciones, numéralas. La numeración fuerza orden de ejecución y permite parar al Agent entre pasos.

### Constraints
La parte más infravalorada. Decir "no escribas código" o "no propongas refactors" o "no inventes, pregunta si dudas" reduce alucinaciones más que ningún otro truco.

### Expected Output
Define el formato. "Markdown con 5 secciones, máximo 15 líneas cada una" es mejor que "explícamelo bien". Si no defines formato, el Agent improvisa.

---

## Plantilla base

```markdown
# Role
[Quién es el Agent en esta tarea — un rol concreto, no genérico]

# Objective
[Qué resultado quieres en una frase]

# Context
[Stack, decisiones previas, lo que NO se debe asumir conocido]

# Tasks (si aplica)
1. [Acción 1 con detalles]
2. [Acción 2 con detalles]
3. ...

# Constraints
- [Lo que NO debe hacer]
- [Si encuentra ambigüedad, qué hacer]
- [Restricciones técnicas: no tocar X, no modificar Y]

# Expected Output
[Formato exacto: estructura, secciones, longitud]
```

---

## Ejemplos por situación

### Ejemplo A — Onboarding a codebase desconocido

```markdown
# Role
Eres un ingeniero senior backend (Node.js + TypeScript + Prisma)
haciendo onboarding tecnico a un codebase desconocido. Tu trabajo
no es producir codigo, es leer rapido y devolver un mapa accionable.

# Objective
Producir un mapa conciso del proyecto que me permita, en la
siguiente iteracion, planificar una feature nueva sin sorpresas
arquitectonicas.

# Context
- Stack: Node.js + TypeScript + Express + Prisma + PostgreSQL
- Arquitectura por capas: presentation, application, domain, infrastructure
- Existen ya endpoints A y B que voy a tomar como referencia

# Tasks
1. Modelo de datos relevante (entidades clave + relaciones).
2. Patron de capas: traza un endpoint existente desde la ruta hasta la BD.
3. Tests existentes: framework, ubicacion, estrategia de mocks.
4. Convenciones: naming, validacion, manejo de errores.

# Constraints
- No leas ni cites schemas completos. Solo lo relevante.
- No propongas mejoras ni refactors. Onboarding, no review.
- No escribas codigo nuevo.
- Si una seccion no la puedes responder con confianza, escribe
  "no encontrado" y dime donde miraste — no inventes.

# Expected Output
Markdown con 4 secciones (una por task), maximo 15 lineas por seccion.
Cierra con un bloque "TODOs para mi" con 3 puntos concretos a verificar.
```

### Ejemplo B — Planificación de feature

```markdown
# Role
Eres un tech lead disenando un PR pequeno y limpio. Tu prioridad
es producir un plan ejecutable que un junior pueda implementar
sin tomar decisiones arquitectonicas.

# Objective
Producir un plan de implementacion detallado para [FEATURE].

# Context
[Decisiones ya tomadas que NO se cuestionan. Por ejemplo:
patron de capas elegido, librerias a usar, restricciones de BD.]

# Tasks
1. Estructura de archivos (tabla con ruta + accion + proposito).
2. Pseudo-firmas TypeScript de cada funcion nueva + DTOs.
3. Pseudocodigo de la query principal (cero N+1, justificalo).
4. Lista de validaciones en orden + codigos HTTP.
5. Plan de tests describe/it.

# Constraints
- Una sola query principal. Si crees que necesitas mas, justifica.
- Cero cambios en schema ni nuevas dependencias.
- Reutiliza el validator/repository/middleware existente.
- Si encuentras algo ambiguo, escribe "Necesito confirmacion"
  con preguntas concretas. NO inventes.
- No escribas codigo de implementacion. Solo plan.

# Expected Output
Markdown estructurado: tabla, pseudocodigo, listas, plan de tests,
riesgos y decisiones abiertas.
```

### Ejemplo C — Implementación paso a paso

```markdown
# Role
Eres un ingeniero implementando exactamente el plan acordado.
Cero desviaciones. Si encuentras algo no previsto, paras y preguntas.

# Objective
Implementar [FEATURE] siguiendo el plan validado en el turno anterior.

# Context
Plan ya validado en el turno anterior. Decisiones de comportamiento
ya confirmadas: [lista C1, C2, C3...].

# Tasks (orden estricto, NO mezclar)
1. Modificar archivo X. PARA y muestrame antes de seguir.
2. Crear archivo Y. PARA y espera "ok, sigue".
3. Crear archivo Z. PARA y espera "ok, sigue".
4. Modificar tests. Ejecuta `npm test` y pega el output completo.

# Constraints
- Implementa SOLO esta feature. NO toques codigo fuera del scope.
- Despues de cada paso, DETENTE y espera mi confirmacion explicita
  "ok, sigue". No encadenes pasos.
- Si encuentras un hueco que no puedes resolver con el contexto,
  preguntame. No improvises.

# Expected Output (por cada paso)
- Ruta del archivo modificado/creado.
- Codigo completo (no diffs parciales).
- Una linea de justificacion por decision no trivial.
```

---

## Antipatrones del usuario

Errores frecuentes al redactar prompts:

1. **"Hazlo bien"** — sin role, sin formato esperado, sin restricciones. El Agent improvisa.
2. **Pedir 5 cosas en una frase** — mejor 5 prompts cortos que uno largo. El Agent gestiona mejor un foco a la vez.
3. **No detener entre pasos en implementacion** — si pides "implementa A, B y C", el Agent lo hace todo de un tiron y pierdes la oportunidad de revisar B antes de C.
4. **Repetir contexto que ya esta en el chat** — el Agent conserva el turno anterior. No le repitas el README cada vez.
5. **No definir el "no"** — un prompt sin Constraints es una invitacion a hacer mas de lo pedido (scope creep, refactors no solicitados, dependencias nuevas).

---

## Cuando NO hace falta RACEO

Para tareas triviales, RACEO es excesivo. Ejemplos donde basta una frase:

- "Renombra esta variable a `userId` en este archivo."
- "Ejecuta `npm test` y pasame el output."
- "Que dependencias hay en `package.json`?"

**Regla:** si la tarea es de un solo paso, mecanica, y sin decisiones arquitectonicas, una frase basta. Si implica diseno, planificacion o implementacion de logica de negocio, usa RACEO.

---

## Pro tip: usar el plan como meta-prompt

Cuando termines un proyecto/feature, copia los prompts efectivos a `prompts/templates/` para reutilizarlos. Un prompt que funciono bien en un onboarding sirve para el siguiente con cambios minimos. Es la base de tu propio "skill personal" de prompting.
