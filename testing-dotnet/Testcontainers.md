# Testcontainers

## ¿Qué es?

Testcontainers es una librería que levanta **contenedores Docker efímeros desde el propio test**: un PostgreSQL real que nace al empezar la clase de tests y muere al terminar. El paquete `Testcontainers.PostgreSql` trae el módulo preconfigurado para PostgreSQL (hay módulos para Redis, RabbitMQ, SQL Server...).

## ¿Por qué existe?

Los tests que necesitan base de datos tenían dos malas opciones: usar una base de datos *in-memory* (rápida, pero no es PostgreSQL: sin índices parciales, sin `SqlState`, sin el mismo SQL) o apuntar a un PostgreSQL instalado en la máquina (real, pero compartido: los tests se pisan entre sí, y "en mi máquina funciona"). Testcontainers da la tercera vía: **base de datos real, pero efímera y privada** — cada ejecución de la suite estrena la suya, en un puerto aleatorio, y la destruye al acabar.

> Es el `docker run postgres` que harías a mano para probar algo, pero automatizado, con puerto aleatorio para no colisionar, y con destrucción garantizada al terminar.

## ¿Cuándo y para qué se usa?

- **Probar el SQL de verdad**: consultas de repositorios, joins, funciones específicas de PostgreSQL.
- **Probar el esquema y sus reglas**: que las migraciones se aplican y son idempotentes, que un índice único rechaza duplicados, que el seed inicial deja los datos esperados. Estas reglas *viven* en la base de datos: mockear el repositorio no las prueba.
- **Tests de API con datos reales**: combinado con `WebApplicationFactory`, la API arranca apuntando al contenedor y los tests ejercitan el flujo completo, de la petición HTTP a la fila en disco.

Requisito único: Docker corriendo en la máquina (o en el runner de CI).

## Lo mínimo que necesitas saber

**1. Declarar y arrancar el contenedor**

El contenedor se arranca en `InitializeAsync` (es asíncrono) y se destruye en `DisposeAsync` — el patrón `IAsyncLifetime` de xUnit:

```csharp
public class ProductRepositoryTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _db =
        new PostgreSqlBuilder("postgres:16-alpine").Build();

    public Task InitializeAsync() => _db.StartAsync();
    public Task DisposeAsync() => _db.DisposeAsync().AsTask();

    [Fact]
    public async Task Add_ConProductoNuevo_LoPersiste()
    {
        var repository = new ProductRepository(_db.GetConnectionString());
        // ...
    }
}
```

Fija la etiqueta de la imagen (`postgres:16-alpine`): los tests deben correr contra la misma versión que producción, no contra "la última".

**2. Compartirlo con un fixture (es caro)**

Arrancar un contenedor tarda segundos; no lo hagas por test. Con `IClassFixture` se arranca una vez por clase (ver la ficha de fixtures):

```csharp
public class MigratedDbFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _db =
        new PostgreSqlBuilder("postgres:16-alpine").Build();

    public string ConnectionString => _db.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _db.StartAsync();
        MigrationRunner.Run(_db.GetConnectionString()); // esquema listo para los tests
    }

    public Task DisposeAsync() => _db.DisposeAsync().AsTask();
}
```

**3. Conectar la API al contenedor**

Una factory que es a la vez `WebApplicationFactory` y dueña del contenedor — la connection string se inyecta antes de que la app lea su configuración:

```csharp
public class ApiWithDbFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _db =
        new PostgreSqlBuilder("postgres:16-alpine").Build();

    public Task InitializeAsync() => _db.StartAsync();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.UseSetting("ConnectionStrings:Default", _db.GetConnectionString());
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await base.DisposeAsync();  // 1º el host web
        await _db.DisposeAsync();   // 2º el contenedor
    }
}
```

**4. Verificar reglas que viven en el esquema**

Con la base de datos real puedes asertar sobre el comportamiento del propio esquema — aquí, que un índice único rechaza duplicados con el `SqlState` de PostgreSQL:

```csharp
var ex = Assert.Throws<PostgresException>(() => connection.Execute(
    """INSERT INTO "User" (email) VALUES ('ANA@EXAMPLE.COM')"""));
Assert.Equal("23505", ex.SqlState); // unique_violation
```

## Lo que NO hace

- **No funciona sin Docker** — si el daemon no está corriendo, los tests fallan al arrancar. En CI necesitas un runner con Docker disponible.
- **No es gratis en tiempo** — cada contenedor tarda segundos en arrancar; por eso se comparte por clase o colección, nunca por test.
- **No limpia los datos entre tests** — el contenedor es el mismo para toda la clase; si los tests se ensucian entre sí, o diseñas datos que no colisionen, o recreas el esquema, o usas una herramienta de reseteo como Respawn.
- **No sustituye a los tests unitarios** — la lógica pura sigue siendo más rápida y clara de probar sin base de datos.

## Buenas prácticas avanzadas

- **Prueba la idempotencia de tus migraciones** — un test que ejecuta el runner de migraciones *dos veces* sobre una base de datos limpia y verifica que la segunda pasada no hace nada es barato de escribir y caza una clase entera de incidentes de despliegue.
- **Aserta sobre `SqlState`, no sobre el mensaje de error** — los mensajes de PostgreSQL cambian entre versiones e idiomas; los códigos (`23505` unique_violation, `23503` foreign_key_violation) son estables. Guárdalos como constantes con nombre.
- **Inyecta explícitamente lo que el test valida** — si en producción las migraciones se auto-descubren por reflexión, en el test pásalas a mano: el test debe declarar exactamente qué esquema está validando, no "el que haya".
- **Usa la imagen `-alpine` y déjala precacheada en CI** — la variante alpine pesa una fracción y arranca antes; un paso de `docker pull` cacheado en el pipeline evita que el primer test pague la descarga.
- **Reutiliza el contenedor, recrea el estado** — la estrategia barata es un contenedor por clase con datos diseñados para no colisionar; la cara pero infalible, contenedor nuevo por clase. Contenedor por *test* casi nunca compensa.

---

*En resumen: Testcontainers te da la base de datos real sin el dolor de tenerla — nace con la suite, vive en Docker y muere sin dejar rastro.*
