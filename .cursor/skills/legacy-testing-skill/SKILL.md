---
name: legacy-testing-skill
description: Cargar cuando se generen tests sobre codigo legacy o potencialmente incorrecto. Distingue SPEC (comportamiento correcto) vs CHARACTERIZATION (comportamiento actual a congelar antes de cambiar). Filosofia "primero congelar, luego cambiar".
---

# Skill: Testing sobre código legacy

Esta skill se activa cuando hay que generar tests sobre código que YA existe y puede contener bugs, inconsistencias o comportamientos cuestionables. Es distinta de `tests-skill` (que asume código nuevo y correcto).

---

## Principio fundamental

**En código legacy, el primer error es cambiar sin red de seguridad.**

Si corriges un comportamiento raro antes de tener tests, nunca sabrás si lo que arreglaste era un bug o una feature de la que dependía otro módulo. Los tests primero capturan el comportamiento actual — bueno o malo — para poder modificarlo después con evidencia.

---

## Secuencia correcta

1. **Capturar comportamiento actual** con characterization tests.
2. **Separar lo correcto de lo legacy** (lo que funciona bien vs lo que huele a bug).
3. **Elegir un caso legacy puntual** a arreglar.
4. **Cambiar el test** al comportamiento deseado → se pone en rojo.
5. **Cambiar el código lo mínimo** para que vuelva a verde.
6. **Refactorizar** con todo en verde.

Nunca saltarse el paso 1 ni el 2.

---

## Qué es un characterization test

Un test que documenta lo que el código hace HOY, **no lo que debería hacer**.

Ejemplo: si `validateEmail("")` hoy devuelve `true` (lo cual es claramente un bug), el characterization test escribe:

```ts
// CHARACTERIZATION: comportamiento actual probablemente incorrecto.
// Un email vacío pasa la validación. Debería rechazarse.
// Ver ticket futuro para corrección.
test("accepts empty string (legacy behavior)", () => {
    expect(validateEmail("")).toBe(true);
});
```

El test pasa en verde. No arreglamos nada todavía. Solo **congelamos** el comportamiento para que cualquier cambio futuro sea detectado.

---

## Cómo marcar los tests

Dos categorías visibles en el código:

- **`// CHARACTERIZATION: ...`** — captura comportamiento actual, posiblemente incorrecto. Cuando se arregle el código, este test cambiará.
- **`// SPEC: ...`** — captura comportamiento correcto y deseado. No debería cambiar salvo refactor del contrato.

La IA genera ambos tipos mezclados en el mismo archivo. La diferencia visible evita confusión futura.

---

## Cuándo sospechar que algo es legacy / bug

Señales que hay que marcar como sospechoso:

- Una rama `if` que nunca se puede alcanzar (código muerto).
- Validaciones que permiten valores vacíos, nulos o negativos sin razón documentada.
- Regex más permisivos o más restrictivos de lo que el contrato/API dice.
- Errores que se tragan silenciosamente con `catch (e) {}`.
- Magic numbers sin constante o comentario.
- Bifurcaciones que tratan `undefined` como valor válido.
- Funciones con nombres que no coinciden con lo que hacen.

La IA debe **listar las sospechas** al usuario antes de empezar a generar tests, para que el humano confirme qué es legacy legítimo y qué es intencional.

---

## Contract vs implementación

Si el proyecto tiene un contrato formal (OpenAPI, JSON Schema, Protobuf, tipos TypeScript estrictos), contrastarlo con la implementación es una forma poderosa de detectar legacy oculto:

1. Leer el contrato (ej. `api-spec.yaml`).
2. Leer la implementación (ej. `validator.ts`).
3. Listar divergencias:
   - Lo que el contrato permite y el código rechaza.
   - Lo que el código acepta y el contrato prohíbe.
   - Campos requeridos en uno y opcionales en el otro.
4. Cada divergencia es candidato a characterization test.

El contrato es **una fuente de verdad alternativa** — útil precisamente porque los docs internos pueden estar desalineados con el código.

---

## Orden de los tests generados

Para un módulo legacy, generar en este orden:

1. **SPEC** — caso feliz con un input obviamente válido (siempre debe pasar).
2. **SPEC** — casos de error que son claramente correctos (rechazar null, empty, etc. si es evidente).
3. **CHARACTERIZATION** — comportamientos sospechosos marcados como tales.
4. **NOTAS al final del archivo** — lista breve de las sospechas detectadas, con referencia a líneas concretas del código fuente.

---

## Regla de oro final

> **La IA propone. QA (el humano) decide qué conservar y qué corregir.**

La IA no decide qué es bug y qué es feature. Su trabajo es hacer visible el comportamiento y señalar sospechas. La decisión sobre qué arreglar es del usuario.
