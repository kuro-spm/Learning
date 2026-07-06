# Testcontainers

**¿Qué es?** Una librería (disponible para .NET, Java, Node, Go...) que permite a tus tests arrancar contenedores Docker reales **desde el propio código de test** — una base de datos, una cola de mensajes — y destruirlos automáticamente al terminar.

---

## ¿Por qué existe?

Para probar código que habla con una base de datos tienes tres opciones clásicas, todas con pegas: *mockear* la BD (no pruebas el SQL real), una BD **compartida** de equipo (los tests se pisan entre sí y dejan basura), o una BD **en memoria** tipo SQLite (dialecto distinto: lo que pasa en el test puede fallar en producción).

Testcontainers da la cuarta opción: cada test (o clase de tests) arranca **su propio PostgreSQL de verdad**, idéntico al de producción, vacío, y lo tira al acabar. Sin instalación previa, sin limpieza manual, sin interferencias.

> Es como si cada examen se hiciera en un aula recién construida que se demuele al entregar: imposible copiarse de la pizarra del examen anterior.

---

## ¿Cuándo y para qué se usa?

- **Tests de integración de repositorios**: verificar que tu SQL real funciona contra el motor real (joins, tipos, constraints).
- **Probar migraciones de esquema**: aplicarlas sobre una BD limpia y comprobar el resultado.
- **Cualquier dependencia externa dockerizable**: PostgreSQL, MySQL, Redis, RabbitMQ, MongoDB... hay un módulo por tecnología.

Requiere Docker en marcha allí donde corran los tests (en los runners de GitHub Actions ya viene instalado).

---

## Lo mínimo que necesitas saber

**1. `PostgreSqlContainer`: la BD del test, declarada en el propio test (.NET)**

```csharp
using Testcontainers.PostgreSql;

// El módulo de PostgreSQL expone un builder preconfigurado (usuario, puerto, imagen)
private readonly PostgreSqlContainer _db =
    new PostgreSqlBuilder("postgres:16-alpine").Build();
```

**2. Arrancar antes y destruir después (con xUnit, vía `IAsyncLifetime`)**

```csharp
public class ProductRepositoryTests : IAsyncLifetime
{
    public Task InitializeAsync() => _db.StartAsync();          // arranca el contenedor
    public Task DisposeAsync()    => _db.DisposeAsync().AsTask(); // lo elimina, pase lo que pase
}
```

**3. Conectarte: el contenedor te da su connection string**

```csharp
[Fact]
public async Task Create_PersisteYSeRecupera()
{
    var repo = new ProductRepository(_db.GetConnectionString()); // puerto aleatorio real
    var id = await repo.CreateAsync(new Product { Name = "Camiseta" });

    var guardado = await repo.GetByIdAsync(id);
    Assert.Equal("Camiseta", guardado!.Name);   // roundtrip contra PostgreSQL de verdad
}
```

**4. La imagen se fija explícitamente**

Usa la misma versión que producción (`postgres:16-alpine`, `postgres:17`...) — esa es justamente la gracia: probar contra el motor real, versión incluida.

---

## Lo que NO hace

- **No sustituye a los tests unitarios** — arrancar un contenedor cuesta segundos; la lógica pura se sigue probando sin infraestructura.
- **No funciona sin Docker** — si el demonio no está corriendo, los tests de integración fallan al arrancar.
- **No comparte estado entre tests** — es una característica, no un defecto: cada clase parte de cero y debe sembrar sus propios datos.
- **No gestiona tu esquema** — la BD nace vacía; aplicar migraciones o DDL antes de probar es responsabilidad del test.

---

## Buenas prácticas avanzadas

- **Un contenedor por suite, no por test — y limpia los datos con Respawn** — el error de rendimiento clásico es arrancar un PostgreSQL nuevo por cada clase de tests: la suite pasa de segundos a minutos. Comparte un único contenedor con una *collection fixture* de xUnit (`ICollectionFixture<T>`) y, entre tests, en vez de recrearlo, borra las filas con una librería como **Respawn** (hace TRUNCATE respetando las foreign keys en milisegundos). Aislamiento igual, coste mínimo.
- **`WithReuse(true)` para no pagar el arranque en cada ejecución local** — con `new PostgreSqlBuilder(...).WithReuse(true)` el contenedor sobrevive al final de la suite y la siguiente ejecución lo reutiliza en lugar de arrancar uno nuevo; en el ciclo "toco código, lanzo tests" se nota muchísimo. Requiere activar `testcontainers.reuse.enable=true` en `~/.testcontainers.properties` y no tiene sentido en CI (el runner es desechable) — por eso es opt-in dos veces.
- **Para imágenes sin módulo oficial, define la *wait strategy* tú** — los builders como `PostgreSqlBuilder` ya saben esperar a que la BD acepte conexiones, pero si dockerizas algo a mano con `ContainerBuilder`, Testcontainers solo espera a que el contenedor *arranque*, no a que el servicio esté listo: la causa nº 1 de tests intermitentes con "connection refused". Declara la condición real con `WithWaitStrategy(Wait.ForUnixContainer().UntilMessageIsLogged(...))` o `UntilHttpRequestIsSucceeded(...)`.

---

*En resumen: Testcontainers convierte "necesito una base de datos para este test" en una línea de código — un PostgreSQL real, limpio y desechable por cada ejecución.*
