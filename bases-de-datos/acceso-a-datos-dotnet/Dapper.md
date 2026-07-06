# Dapper

## ¿Qué es?

Dapper es un micro-ORM para .NET que extiende `IDbConnection` con métodos de extensión para ejecutar SQL y mapear los resultados directamente a objetos C# de forma sencilla y eficiente.

## ¿Por qué existe?

**Entity Framework Core** (EF Core) genera el SQL por ti, lo que es cómodo pero sacrifica control y, en consultas complejas, rendimiento. Dapper toma el camino contrario: tú escribes el SQL, y Dapper se encarga solo del mapeo. El resultado es código predecible, consultas optimizadas y sin la capa de abstracción pesada del ORM completo.

> Si vienes de EF Core, piensa en Dapper como "EF Core sin el DbContext, sin las migraciones y sin el generador de SQL — solo la parte de leer resultados".

## ¿Cuándo y para qué se usa?

Dapper se usa en la capa de repositorios de cualquier aplicación que necesite ejecutar consultas SQL directas sobre la base de datos: una tienda online que consulta su catálogo de productos, un sistema de facturación o una app de gestión de tareas. Permite escribir joins y filtros específicos del dominio sin depender de un grafo de entidades, lo que facilita consultas de lectura optimizadas sin cargar objetos innecesarios.

## Lo mínimo que necesitas saber

**1. Consulta simple con mapeo**

```csharp
using var connection = new SqlConnection(connectionString);
var productos = await connection.QueryAsync<Producto>(
    "SELECT Id, Nombre, Estado FROM Productos WHERE Estado = @estado",
    new { estado = "Activo" });
```

**2. Consulta que devuelve un único resultado**

```csharp
var producto = await connection.QueryFirstOrDefaultAsync<Producto>(
    "SELECT * FROM Productos WHERE Id = @id",
    new { id = productoId });
```

**3. Ejecutar INSERT / UPDATE / DELETE**

```csharp
int filas = await connection.ExecuteAsync(
    "UPDATE Productos SET Estado = @estado WHERE Id = @id",
    new { estado = "Descatalogado", id = productoId });
```

**4. Parámetros anónimos**

Los parámetros se pasan como objetos anónimos (`new { ... }`). Dapper los vincula por nombre con los `@parametres` del SQL, evitando SQL injection.

**5. Transacciones**

```csharp
using var tx = connection.BeginTransaction();
await connection.ExecuteAsync("INSERT INTO ...", data, transaction: tx);
tx.Commit();
```

## Lo que NO hace

- **No genera SQL** — tú escribes cada consulta a mano.
- **No gestiona migraciones** de base de datos.
- **No rastrea cambios** en entidades (no hay change tracking como en EF Core).
- **No valida el esquema** en tiempo de compilación — un error tipográfico en el SQL solo aparece en ejecución.

---

*En resumen: Dapper es la capa mínima entre tu SQL y tus objetos C# — tú controlas la consulta, él se encarga del mapeo.*
