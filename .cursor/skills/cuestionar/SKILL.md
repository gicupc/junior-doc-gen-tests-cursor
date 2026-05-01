---
name: cuestionar
description: Auditar la calidad real de los tests con mutation thinking, sin instalar herramientas. La IA propone mutaciones realistas, evalua si la suite las detectaria, lista las que sobreviven y propone tests faltantes. Filosofia "cobertura no es calidad".
disable-model-invocation: true
---

# /cuestionar — Cuestionar la calidad de los tests

Este comando NO instala herramientas ni ejecuta mutation testing automaticamente. Plantea mentalmente mutaciones sobre el codigo y detecta falsa confianza en la suite actual.

La filosofia: **cobertura no es calidad**. Un test util es el que detecta errores relevantes, no el que simplemente "pasa por encima" del codigo.

---

## Cuando usar este comando

- Has cerrado varios tickets con tests en verde pero algo te escama.
- Quieres validar si la suite actual es robusta o solo aparenta serlo.
- Antes de un release importante, como auditoria de calidad.
- Cuando un bug se colo en produccion a pesar de tener tests "que lo cubrian".

---

## Pasos del protocolo

### Paso 1 — Elegir el alcance
Pregunta al usuario: "¿Sobre que modulo o archivo quieres cuestionar la calidad de los tests?"

Opciones comunes:
- Un archivo especifico (`src/services/AuthService.php`).
- Un modulo completo (`src/Validators/`).
- Todo el backend (solo si el proyecto es pequeno).

### Paso 2 — Leer codigo y tests del alcance
Lee el codigo fuente Y sus tests asociados. Necesitas ambos para el analisis.

### Paso 3 — Generar mutaciones mentales
Para cada funcion importante del codigo, propon al menos 5 mutaciones realistas:

**Mutaciones de operadores:**
- Cambiar `>` por `>=` o por `<`.
- Cambiar `&&` por `||`.
- Cambiar `==` por `!=`.

**Mutaciones de valores:**
- Cambiar `0` por `1` o por `-1`.
- Cambiar `true` por `false`.
- Cambiar un string literal.

**Mutaciones de flujo:**
- Eliminar una llamada a metodo (ej. `db.save()`).
- Duplicar una llamada.
- Cambiar el orden de dos llamadas.
- Hacer que una funcion devuelva el input sin procesarlo.

**Mutaciones de logica de negocio especifica:**
- En un contador: no incrementar.
- En una validacion: aceptar lo que deberia rechazar.
- En un ownership check: ignorar el user_id.
- En un flujo de estados: saltar un paso o repetir uno.

### Paso 4 — Evaluar cada mutacion
Para cada mutacion propuesta, responde a tres preguntas:

1. **¿Los tests actuales la detectarian?** (Si/No/Parcialmente)
2. **¿Por que?** (explica el razonamiento)
3. **Si sobrevive, ¿que test falta?** (propuesta concreta)

Presenta el resultado en una tabla:

```
| Mutacion | Detectada? | Por que | Test faltante |
|----------|-----------|---------|---------------|
| ... | 🔴 Sobrevive | Los tests solo verifican el caso feliz | Test con X = 0 |
| ... | 🟢 Detectada | El test "rechaza valor negativo" la caza | Ninguno |
| ... | 🟡 Parcial | Solo un test de los 5 la detecta | Reforzar asserts |
```

### Paso 5 — Resumen ejecutivo
Al final, presenta:

- **Mutation score estimado** (aproximado: X% de mutaciones serian detectadas).
- **Tests faltantes prioritarios** (los 3-5 mas importantes).
- **Modulos con falsa confianza** (los que parecen bien cubiertos pero fallan en mutacion).

### Paso 6 — Opcion de ir mas alla
Pregunta al usuario: "¿Quieres que implementemos mutation testing real?"

- **Si** — propon el framework segun stack:
  - PHP → Infection
  - JS/TS → StrykerJS (`@stryker-mutator/jest-runner`, `@stryker-mutator/typescript-checker`)
  - Java → PIT
  - Python → mutmut
  - C++ → Mull
  - Registra la decision como ADR en `decisions-log.md`.
  - Anade ticket al `roadmap.md` para configurarlo.
- **No** — deja el analisis como reporte. Pregunta si quiere anadir los tests faltantes prioritarios (paso que SI implementaria tests nuevos).

### Paso 7 — Cierre
Ejecuta `On(task_complete)` con el checklist de sincronizacion documental.

---

## Property-based testing (mencion breve)

Si durante el analisis detectas que el modulo tiene **invariantes claras** (propiedades que siempre deben cumplirse independientemente del input), menciona al usuario la opcion de property-based testing:

- Ejemplos de invariantes:
  - "El resultado siempre es mayor que el input."
  - "Aplicar dos veces la misma operacion devuelve el mismo resultado."
  - "Para cualquier input valido, nunca se lanza excepcion."

Herramientas por stack: fast-check (JS/TS), Hypothesis (Python), jqwik (Java).

NO implementes PBT en este comando. Solo mencionalo si aplica, y sugiere que puede ser un siguiente paso separado.

---

## Regla de oro

> **La mutacion no sustituye al pensamiento. Lo enfoca.**

El valor de este comando no es generar una lista larga de mutaciones, es hacer pensar al usuario sobre que situaciones reales de bug podrian pasar desapercibidas.
