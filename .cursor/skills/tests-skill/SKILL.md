---
name: tests-skill
description: Convenciones y patrones para generar tests unitarios. Usar siempre que se generen, modifiquen o auditen tests. Cubre PHP+PHPUnit, JS/TS+Jest, JS/TS+Vitest, Python+pytest. Incluye buenas practicas, mocks, linter-friendly y antipatrones a evitar.
---

# Skill: Generación de Tests

Esta skill contiene las convenciones y patrones para generar tests unitarios en cualquier proyecto bajo el protocolo Architect-Brain.

---

## Principios

1. **Un test responde a una pregunta concreta** sobre el comportamiento del código.
2. **Caso feliz + casos de error + bordes.** Por cada unidad de lógica, cubrir como mínimo:
   - 1 test del camino exitoso.
   - 1 test por cada regla de validación o error esperado.
   - 1-2 tests de valores en el borde (vacío, null, límites de longitud/rango).
3. **Testear la lógica, no la fontanería.** Validadores, servicios, cálculos → sí. Renderizado HTML, getters/setters, wiring de frameworks → no.
4. **AAA visible**: Arrange (preparar datos), Act (ejecutar), Assert (verificar).
5. **Nombres descriptivos**: `throws an error when email format is invalid` > `test1`.
6. **Independencia**: cada test debe poder ejecutarse aislado. Usar `beforeEach` para limpiar estado compartido o mocks.

---

## Qué mockear

Todo lo que sea "mundo exterior":
- **Bases de datos** (Prisma, PDO, Eloquent, TypeORM, mongoose).
- **APIs HTTP** (fetch, axios, guzzle).
- **Sistema de archivos** (lectura/escritura).
- **Tiempo y aleatoriedad** (`Date.now()`, `Math.random()`, `uniqid()`) si el test depende de ellos.
- **Servicios de terceros** (Stripe, SendGrid, SMTP, S3).

Lo que NO mockear:
- Código puro del propio módulo (validadores, parsers, calculadoras) — si lo mockeas dejas de testear nada.

---

## Linter-friendly (importante)

Algunos frameworks de test crean métodos dinámicamente, lo que hace que los linters estáticos marquen falsos positivos. Hay que instalarse las extensiones correctas durante el setup (el protocolo `On(testing_setup)` lo hace) y saber reconocer estos falsos positivos si aparecen:

| Stack | Problema típico | Solución |
|---|---|---|
| **PHP + PHPUnit** | Intelephense marca error en `$mock->method(...)`, `$mock->expects(...)`. Esos métodos se inyectan dinámicamente via `__call()`. | Instalar `phpstan/phpstan-phpunit` como dev dependency. |
| **JS/TS + Jest** | Si falta `@types/jest`, marca `describe`/`it`/`expect` como no definidos. | `npm i -D @types/jest` (el protocolo ya lo hace). |
| **Vitest** | Tipos nativos incluidos. | Nada que hacer. |
| **pytest** | Los fixtures pueden marcarse como "unused parameter" en IDEs. | Instalar `pytest-mock` y añadir `# type: ignore` donde aplique. |

**Regla de oro**: si los tests pasan en verde pero el linter marca error, casi siempre es falso positivo de este tipo. Verifica con el protocolo y sigue.

---

## Patrones por stack

### PHP + PHPUnit

```php
// tests/ValidatorTest.php
use PHPUnit\Framework\TestCase;

class ValidatorTest extends TestCase
{
    public function test_accepts_valid_email(): void
    {
        $this->assertTrue(Validator::email('ana@test.com'));
    }

    public function test_rejects_email_without_at_sign(): void
    {
        $this->expectException(InvalidArgumentException::class);
        Validator::email('no-arroba');
    }
}
```

Mock de BD típico: `$pdo = $this->createMock(PDO::class);`

### JavaScript/TypeScript + Jest

```ts
describe('validateEmail', () => {
    it('accepts valid email', () => {
        expect(() => validateEmail('ana@test.com')).not.toThrow();
    });

    it('throws when email has no domain', () => {
        expect(() => validateEmail('ana@')).toThrow('Invalid email');
    });
});
```

Mock de Prisma:

```ts
jest.mock('@prisma/client', () => ({
    PrismaClient: jest.fn().mockImplementation(() => ({
        user: { create: jest.fn(), findUnique: jest.fn() },
    })),
}));
```

### Python + pytest

```python
def test_accepts_valid_email():
    assert validate_email("ana@test.com") is True

def test_rejects_email_without_domain():
    with pytest.raises(ValueError, match="Invalid email"):
        validate_email("ana@")
```

Mock con `unittest.mock.patch` o la fixture `mocker` de pytest-mock.

---

## Estructura de carpetas recomendada

| Stack | Ubicación |
|-------|-----------|
| PHP | `tests/` en la raíz, espejo de `src/` |
| JS backend | `src/tests/` o `backend/src/tests/` |
| JS frontend | `src/__tests__/` junto a componentes, o `tests/` global |
| Monorepo | `packages/<pkg>/tests/` por paquete |

Decisión documentada en `docs/testing-strategy.md`.

---

## Al generar tests para código existente (Regularización, opción [4])

1. **Lee primero** el archivo a testear completo, entiende la intención.
2. **Identifica las funciones públicas** (exportadas) y qué reglas de negocio encapsulan.
3. **Genera el caso feliz primero**. Si no pasa, algo está mal (entendimiento o mock).
4. **Añade un test por cada `throw`, `return false`, o rama de validación** que veas en el código.
5. **Mockea dependencias externas** según las reglas de arriba.
6. **Ejecuta toda la suite**. Si algo falla, NO lo marques verde en el roadmap.
7. **Documenta lo que NO cubriste** en `testing-strategy.md` — honestidad sobre el estado real.

---

## Antipatrones a evitar

- **Tests que replican la implementación**: si tu test comprueba "el método llamó a X y luego a Y", estás acoplando al código, no a la conducta. Mejor: verificar el resultado observable.
- **Datos irreales**: `firstName: 'aaa'` hace tests menos legibles que `firstName: 'Ana'`. Usa datos realistas pero mínimos.
- **Tests gigantes con 20 asserts**: descomponer en varios tests con un foco claro cada uno.
- **`.skip` olvidados**: un test saltado durante más de una semana debe convertirse en ticket del roadmap.
- **Cobertura por cobertura**: 90% de coverage con tests triviales es peor que 60% con tests que capturan la lógica real.
- **Silenciar warnings del linter a ciegas**: si aparece un error en un test que pasa en verde, primero verifica si es falso positivo del framework (ver tabla "Linter-friendly") antes de añadir `@phpstan-ignore`, `// @ts-ignore` o similares.
