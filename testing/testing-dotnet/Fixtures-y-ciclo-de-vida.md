# Fixtures y ciclo de vida en xUnit

## ¿Qué es?

Los *fixtures* son el mecanismo de xUnit para **compartir recursos caros entre tests** (un host web, un contenedor de base de datos) y controlar cuándo se crean y se destruyen. El ciclo de vida define qué se ejecuta antes y después de cada test, de cada clase y de cada colección de clases.

## ¿Por qué existe?

xUnit crea una instancia nueva de la clase de test para *cada* test: aislamiento perfecto, pero un problema cuando el setup es caro. Arrancar una API en memoria tarda segundos; levantar un contenedor Docker con PostgreSQL, más. Repetirlo por test convertiría una suite de 2 segundos en una de 10 minutos. Los fixtures resuelven el dilema: el recurso caro se crea **una vez** y los tests lo comparten, mientras la clase de test sigue recreándose por test.

> Si conoces la inyección de dependencias de ASP.NET, los fixtures son parecidos a los *lifetimes*: la clase de test es "transient" (una por test) y el fixture es "singleton" dentro de su ámbito (clase o colección).

## ¿Cuándo y para qué se usa?

Siempre que varios tests necesiten el mismo recurso costoso: la `WebApplicationFactory` que arranca la API, el contenedor de PostgreSQL con las migraciones aplicadas, un servidor de mensajería de pruebas. También cuando un recurso necesita inicialización **asíncrona** (arrancar un contenedor es `async`), algo que un constructor de C# no puede hacer.

## Lo mínimo que necesitas saber

**1. El ciclo de vida básico: constructor y Dispose**

Sin fixtures, el constructor es tu "setup" y `Dispose` tu "teardown" — se ejecutan por cada test:

```csharp
public class CartTests : IDisposable
{
    private readonly ShoppingCart _cart = new();   // antes de CADA test

    public void Dispose() { /* después de CADA test */ }

    [Fact]
    public void Add_ConProducto_IncrementaElContador() { /* ... */ }
}
```

**2. `IClassFixture<T>`: compartir entre los tests de una clase**

xUnit crea el fixture una vez, lo inyecta por constructor en cada test de la clase, y lo destruye al acabar la clase:

```csharp
public class ProductApiTests : IClassFixture<TestWebApplicationFactory>
{
    private readonly TestWebApplicationFactory _factory;

    public ProductApiTests(TestWebApplicationFactory factory) => _factory = factory;

    [Fact]
    public async Task GET_Products_DevuelveOk()
    {
        var client = _factory.CreateClient();   // el host ya está arrancado
        // ...
    }
}
```

**3. `IAsyncLifetime`: setup y teardown asíncronos**

Un constructor no puede hacer `await`. `IAsyncLifetime` añade `InitializeAsync` / `DisposeAsync`, y funciona tanto en clases de test como en fixtures:

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _db =
        new PostgreSqlBuilder("postgres:16-alpine").Build();

    public string ConnectionString => _db.GetConnectionString();

    public Task InitializeAsync() => _db.StartAsync();        // arranca el contenedor
    public Task DisposeAsync() => _db.DisposeAsync().AsTask(); // lo destruye al final
}
```

**4. `ICollectionFixture<T>`: compartir entre varias clases**

Si varias clases de test necesitan el mismo recurso, se agrupan en una colección. Bonus: las clases de una misma colección no se ejecutan en paralelo entre sí, lo que evita pisarse los datos:

```csharp
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database")]
public class ProductRepositoryTests { /* recibe DatabaseFixture por constructor */ }

[Collection("Database")]
public class OrderRepositoryTests { /* la MISMA instancia del fixture */ }
```

## Lo que NO hace

- **No limpia los datos entre tests** — el fixture comparte el recurso tal cual; si un test inserta filas, el siguiente las ve. La limpieza (recrear el esquema, borrar tablas) es responsabilidad tuya.
- **No inyecta cualquier cosa por constructor** — xUnit solo inyecta fixtures declarados con `IClassFixture`/`ICollectionFixture`; no es un contenedor de DI genérico.
- **No garantiza orden entre tests** — compartir fixture no significa que los tests se ejecuten en un orden concreto; no encadenes estado entre ellos.

## Buenas prácticas avanzadas

- **Compartir estado mutable es la primera fuente de tests intermitentes** — si los tests de una clase escriben en la misma base de datos compartida, diseña los datos para que no colisionen (IDs propios por test) o limpia entre tests. El síntoma delator: tests que pasan solos y fallan en suite.
- **Cuidado con el orden de destrucción al combinar herencia e `IAsyncLifetime`** — si tu fixture hereda de algo que ya gestiona recursos (como `WebApplicationFactory`, que es `IAsyncDisposable`), implementa `IAsyncLifetime.DisposeAsync` de forma explícita y ordena el apagado a mano: primero el host, luego el contenedor del que depende. Apagar la base de datos antes que la aplicación que la usa produce errores fantasma en el teardown:

```csharp
async Task IAsyncLifetime.DisposeAsync()
{
    await base.DisposeAsync();   // 1º el host web
    await _db.DisposeAsync();    // 2º el contenedor
}
```

- **El fixture es el sitio para las decisiones, no para la magia** — declara en el fixture exactamente lo que el test necesita (qué migraciones se aplican, qué configuración se inyecta) aunque en producción eso se descubra automáticamente. Un test que depende de auto-descubrimiento valida "lo que haya", no "lo que declaro".
- **Un fixture por *estrategia*, no por clase de test** — ten un fixture "API sin base de datos" y otro "API con PostgreSQL real", y que cada clase de test elija. Multiplicar fixtures casi idénticos es la versión testing del copy-paste.

---

*En resumen: los fixtures son la respuesta de xUnit al setup caro — crea el recurso una vez, compártelo con criterio y destrúyelo en el orden correcto.*
