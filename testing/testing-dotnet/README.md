# Testing en .NET — Guía de metodologías

Guía introductoria de testing en .NET para desarrolladoras con experiencia backend que quieren escribir tests con criterio: qué herramienta usar en cada capa, cómo estructurar los tests y qué prácticas distinguen una suite fiable de una que solo pinta verde.

Cada ficha explica qué es la tecnología o el patrón, por qué existe, cuándo se usa y lo mínimo para no perderse — sin depender de ningún proyecto concreto.

---

## Orden de lectura recomendado

### 1. Fundamentos

Antes de tocar herramientas, el mapa: qué tipos de test existen y cómo se estructura cualquiera de ellos.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Tipos de tests](Tipos-de-tests.md) | El mapa general: unitario, integración y end-to-end, y cuándo usar cada uno. |
| 2 | [xUnit](xUnit.md) | El framework sobre el que se escribe todo lo demás: `[Fact]`, `[Theory]`, `Assert`. |
| 3 | [Shouldly](Shouldly.md) | Aserciones más legibles que el `Assert` de xUnit — mismos tests, mejor lectura. |
| 4 | [Arrange-Act-Assert](Arrange-Act-Assert.md) | La estructura interna de cada test. Dos minutos de lectura. |

### 2. Compartir recursos caros

Cuando los tests necesitan un host web o una base de datos, arrancarlos por test es inviable: aquí entra el ciclo de vida de xUnit.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 5 | [Fixtures y ciclo de vida](Fixtures-y-ciclo-de-vida.md) | `IClassFixture`, `IAsyncLifetime` y compañía: la base de los tests de integración. |

### 3. Tests de API e integración

Las dos herramientas que permiten probar la aplicación real: el host en memoria y la base de datos efímera.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 6 | [WebApplicationFactory](WebApplicationFactory.md) | Arranca tu API completa en memoria y la ataca por HTTP. |
| 7 | [Testcontainers](Testcontainers.md) | PostgreSQL real y efímero en Docker para probar SQL, esquema y migraciones. |

### 4. Aislamiento y medición

El complemento de los tests de integración: aislar la lógica pura y medir qué queda sin cubrir.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 8 | [NSubstitute](NSubstitute.md) | Mocking de dependencias para tests unitarios de lógica de negocio. |
| 9 | [Coverlet](Coverlet.md) | Cobertura de código: encontrar los huecos que ningún test pisa y fijar umbrales. |

---

> Si vienes del frontend, la ficha de [Vitest](../../desarrollo-web/frontend-react/Vitest.md) de la guía de React es el equivalente de xUnit en TypeScript — muchos conceptos de esta colección (runner, aserciones, mocking) tienen allí su reflejo.
