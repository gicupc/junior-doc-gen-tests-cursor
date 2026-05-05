# Decisions Log

> Registro historico de decisiones tecnicas y de producto del proyecto.  
> Cada entrada es inmutable: NUNCA se edita el contenido de una decision existente.  
> Si una decision cambia, se anade una nueva entrada que la supersede (ver campo "Estado").

---

## Formato

Cada decision se registra con:

- **ID**: secuencial (ADR-001, ADR-002, ...).
- **Fecha**: YYYY-MM-DD.
- **Titulo**: frase corta que resume la decision.
- **Estado**: `Vigente` | `Superseded por ADR-XXX` | `Deprecated`.
- **Contexto**: por que se tomo esta decision (situacion, restricciones, limitaciones conocidas).
- **Decision**: que se eligio hacer.
- **Alternativas descartadas** (opcional): que opciones se consideraron y por que no se eligieron.
- **Consecuencias** (opcional): que implica la decision a corto/medio plazo.

---

## Cuando se anade una decision

- Al cerrar la entrevista BMADT del `On(project_start)` → se registran las 5 respuestas como ADR-001 a ADR-005 (o una sola ADR con las cinco dimensiones, segun complejidad).
- Al invocar `On(revisit_decision)` → se anade una nueva ADR con el cambio y se marca la anterior como `Superseded por ADR-XXX`.
- Al tomar cualquier decision no trivial durante el desarrollo (cambio de libreria, refactor estructural, cambio de convencion) → el usuario o la IA anaden una ADR.

---

## Decisiones de comportamiento ad-hoc

Durante la planificacion de una feature compleja (en `On(implementation_phase)` o tras un plan del Agent), aparecen a menudo preguntas semanticas que el enunciado no aclara: "¿que devuelve el avg si todos los scores son nulos?", "¿este endpoint solo actualiza el puntero o tambien crea un registro?", "¿que filtro implicito hay detras de palabras como 'en proceso'?".

Estas decisiones se etiquetan como `C1`, `C2`, `C3`... durante el plan, se confirman explicitamente con el usuario antes de implementar, y se documentan aqui como ADR. Formato sugerido:

```
ADR-XXX — Decisiones de comportamiento de [feature]

Estado: Vigente

Contexto: durante la planificacion de [feature] surgieron N preguntas
semanticas no cubiertas por el enunciado original. Se confirmaron con
el usuario en sesion antes de implementar.

Decisiones:
- C1 (titulo corto): decision tomada. Razon: ...
- C2 (titulo corto): decision tomada. Razon: ...
- C3 (titulo corto): decision tomada. Razon: ...
```

Esto evita que el "porque la IA lo decidio asi" sea la justificacion en code reviews. La justificacion correcta es "porque tomamos esta decision por estas razones, registrada en ADR-XXX".

---

## ADR-000 (plantilla - NO BORRAR)

**Fecha**: YYYY-MM-DD  
**Titulo**: [Una frase que resuma la decision]  
**Estado**: Vigente

### Contexto
[Situacion que motivo la decision. Que problema o pregunta se estaba resolviendo. Restricciones conocidas.]

### Decision
[Que se eligio hacer, en lenguaje claro y accionable.]

### Alternativas descartadas
- **[Opcion A]** — descartada por [razon].
- **[Opcion B]** — descartada por [razon].

### Consecuencias
[Implicaciones conocidas de la decision: que se vuelve facil, que se vuelve dificil, que deudas tecnicas se asumen.]

---

<!--
  Las decisiones reales del proyecto se anaden debajo de esta linea,
  en orden cronologico (mas reciente al final).
  
  Cuando una decision sea superseded, edita SOLO el campo "Estado" de la entrada antigua
  para anadir la referencia a la nueva ADR. El contenido original se conserva intacto.
-->
