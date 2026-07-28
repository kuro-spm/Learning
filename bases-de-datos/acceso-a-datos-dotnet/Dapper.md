# Dapper

## ¿Qué es?

Dapper es un **micro-ORM** para .NET: un paquete que añade métodos de extensión a `IDbConnection` para ejecutar SQL y volcar el resultado en objetos C#. Tú escribes la consulta, él se encarga de rellenar los objetos.

## ¿Por qué existe?

Leer datos con el driver a pelo funciona, pero es puro ceremonial. Esto es traer un producto con [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) sin ayuda:

```csharp
using var comando = new SqlCommand("SELECT Id, Nombre, Precio FROM Productos WHERE Id = @id", conexion);
comando.Parameters.AddWithValue("@id", 17);
using var lector = await comando.ExecuteReaderAsync();
var productos = new List<Producto>();
while (await lector.ReadAsync())
{
    productos.Add(new Producto
    {
        Id = lector.GetInt32(0), Nombre = lector.GetString(1), Precio = lector.GetDecimal(2)
    });
}
```

Diez líneas, tres índices de columna que se rompen en silencio si alguien reordena el `SELECT`, y ni una sola decisión interesante. Lo mismo con Dapper:

```csharp
var productos = await conexion.QueryAsync<Producto>(
    "SELECT Id, Nombre, Precio FROM Productos WHERE Id = @id", new { id = 17 });
```

El otro extremo es un ORM completo como [Entity Framework Core](../../desarrollo-web/de-wpf-a-web/Entity-Framework-Core.md), que genera el SQL a partir de LINQ, rastrea los cambios de las entidades y gestiona migraciones. Es cómodo, y a cambio pierdes control sobre la consulta que acaba llegando a la base de datos.

> Si ya conoces EF Core, piensa en Dapper como "EF Core sin el `DbContext`, sin las migraciones y sin el generador de SQL — solo la parte de leer resultados".

## ¿Cuándo y para qué se usa?

En la capa de repositorios de cualquier aplicación que quiera escribir su propio SQL: informes con varios `JOIN`, consultas de lectura que solo necesitan cuatro columnas de las treinta de la tabla, llamadas a procedimientos almacenados heredados, o cualquier caso en que el plan de ejecución importe lo bastante para no delegarlo.

El ejemplo conductor de esta guía es una tienda online sobre la base de datos **`TiendaDB`**, con las tablas `Productos`, `Pedidos`, `LineasPedido` y `Clientes`, y las entidades C# `Producto`, `Pedido`, `LineaPedido` y `Cliente`. El pedido que iremos siguiendo es el **#4711**.

```csharp
public sealed class Producto
{
    public int Id { get; set; }
    public string Nombre { get; set; } = "";
    public decimal Precio { get; set; }
    public string Estado { get; set; } = "";
}
```

Nada de atributos, nada de clase base: Dapper mapea cualquier clase con propiedades públicas escribibles (y también `record` con constructor).

## Instalación y primer ejemplo de principio a fin

```bash
dotnet add package Dapper
dotnet add package Microsoft.Data.SqlClient
```

Dapper no trae driver: solo extiende `IDbConnection`, así que el paquete del proveedor va aparte. Un repositorio completo y funcional:

```csharp
using Dapper;
using Microsoft.Data.SqlClient;

public sealed class RepositorioProductos(string cadenaConexion)
{
    public async Task<IReadOnlyList<Producto>> ObtenerActivosAsync()
    {
        using var conexion = new SqlConnection(cadenaConexion);
        var productos = await conexion.QueryAsync<Producto>(
            "SELECT Id, Nombre, Precio, Estado FROM Productos WHERE Estado = @estado",
            new { estado = "Activo" });
        return productos.ToList();
    }
}
```

Tres cosas que no están y podrían sorprender:

- **No hace falta `conexion.OpenAsync()`.** Dapper abre la conexión si está cerrada y la deja como la encontró. Si ya estaba abierta, no la cierra.
- **El `using` es imprescindible.** Devuelve la conexión al pool; sin él, el pool se agota. Los detalles del pool están en [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md).
- **`QueryAsync<T>` devuelve un `IEnumerable<T>` ya materializado**, no una consulta perezosa. Volveremos a esto al hablar de *buffered*.

## Los métodos de extensión: cuál usar en cada caso

Toda la API útil de Dapper son seis métodos. La diferencia entre ellos no es de estilo: es **qué pasa cuando la base de datos devuelve algo distinto de lo que esperabas**.

| Filas que esperas | Método | 0 filas | 2+ filas |
|---|---|---|---|
| Varias | `QueryAsync<T>` | Lista vacía | Todas |
| 0 o 1, y me da igual si hay más | `QueryFirstOrDefaultAsync<T>` | `null` / `default` | Devuelve la primera |
| Exactamente 1 | `QueryFirstAsync<T>` | `InvalidOperationException` | Devuelve la primera |
| Exactamente 1, y 2 es un bug | `QuerySingleAsync<T>` | `InvalidOperationException` | `InvalidOperationException` |
| 0 o 1, y 2 es un bug | `QuerySingleOrDefaultAsync<T>` | `null` / `default` | `InvalidOperationException` |
| Ninguna (`INSERT`/`UPDATE`/`DELETE`) | `ExecuteAsync` | Devuelve filas afectadas | — |
| Un valor único | `ExecuteScalarAsync<T>` | `default` | Ignora el resto |

La pareja que se confunde siempre es `First` y `Single`. `QueryFirstOrDefaultAsync` sobre una clave primaria funciona, pero **oculta el bug**: si un día hay dos filas con el mismo `Id`, devuelve una y sigue como si nada. `QuerySingleAsync` avisa:

```csharp
var pedido = await conexion.QuerySingleAsync<Pedido>(
    "SELECT Id, Estado, Total FROM Pedidos WHERE Id = @id", new { id = 4711 });
```

Con dos filas, `Sequence contains more than one element`; con ninguna, `Sequence contains no elements`, ambas como `InvalidOperationException`:

```
System.InvalidOperationException: Sequence contains more than one element
```

Regla práctica: **`Single*` cuando consultas por una clave única** (el error te está diciendo que el esquema o la consulta están mal), y `First*` solo cuando de verdad pides "el más reciente" de un conjunto con `ORDER BY`.

`ExecuteScalarAsync<T>` es para un único valor, típicamente el `Id` recién insertado:

```csharp
int nuevoId = await conexion.ExecuteScalarAsync<int>(
    "INSERT INTO Pedidos (ClienteId, Estado, Total) OUTPUT INSERTED.Id VALUES (@clienteId, 'Pendiente', @total)",
    new { clienteId = 92, total = 149.90m });
```

Usa `OUTPUT INSERTED.Id` y no `SCOPE_IDENTITY()`: funciona igual con inserciones múltiples y no depende del ámbito de los triggers.

## Parámetros

### Objetos anónimos

Los parámetros se pasan como un objeto anónimo. Dapper recorre sus propiedades y crea un `SqlParameter` por cada `@nombre` que aparezca en el SQL, **vinculando por nombre de propiedad**, no por posición:

```csharp
await conexion.ExecuteAsync(
    "UPDATE Productos SET Estado = @estado WHERE Id = @id",
    new { id = 17, estado = "Descatalogado" });   // el orden no importa
```

Esto es lo que hace a Dapper inmune a la inyección de SQL: el valor **nunca se concatena al texto de la consulta**. Viaja aparte, en la lista de parámetros del protocolo, y el motor lo trata como dato. Compara:

```csharp
// ❌ vulnerable: el valor entra en el texto del SQL
var sql = $"SELECT * FROM Productos WHERE Nombre = '{nombre}'";
// con nombre = "x'; DROP TABLE Productos; --" el motor ejecuta dos sentencias

// ✅ seguro: el valor va como parámetro
await conexion.QueryAsync<Producto>(
    "SELECT * FROM Productos WHERE Nombre = @nombre", new { nombre });
```

El segundo caso, con ese mismo valor, simplemente no encuentra ningún producto llamado `x'; DROP TABLE Productos; --`.

### `DynamicParameters`

Cuando los parámetros se construyen condicionalmente, o necesitas controlar el tipo exacto, el objeto anónimo no sirve:

```csharp
var parametros = new DynamicParameters();
parametros.Add("@estado", "Pendiente", DbType.AnsiString, size: 20);

if (fechaDesde is not null)
    parametros.Add("@fechaDesde", fechaDesde, DbType.DateTime2);

var pedidos = await conexion.QueryAsync<Pedido>(sql, parametros);
```

`DynamicParameters` acepta también un objeto anónimo en el constructor y luego añadirle más, lo que va bien para combinar filtros fijos y opcionales.

### Listas en cláusulas `IN`

Aquí Dapper hace algo que parece magia y conviene entender. Se escribe **sin paréntesis**:

```csharp
var ids = new[] { 4711, 4712, 4715 };

var pedidos = await conexion.QueryAsync<Pedido>(
    "SELECT Id, Estado, Total FROM Pedidos WHERE Id IN @ids", new { ids });
```

Dapper detecta que `ids` es una colección y **reescribe el texto del SQL** antes de enviarlo, creando un parámetro por elemento:

```sql
SELECT Id, Estado, Total FROM Pedidos WHERE Id IN (@ids1,@ids2,@ids3)
```

Dos consecuencias prácticas. La primera: si escribes `IN (@ids)` con paréntesis, el SQL resultante queda `IN ((@ids1,@ids2,@ids3))` y SQL Server lo rechaza. La segunda: con la lista **vacía**, Dapper genera algo que no da error de sintaxis pero tampoco devuelve nada:

```sql
SELECT Id, Estado, Total FROM Pedidos WHERE Id IN (SELECT @ids WHERE 1 = 0)
```

Es decir, cero filas. Razonable, pero si tu código esperaba "sin filtro = todos", el resultado te sorprenderá.

### Procedimientos almacenados y parámetros de salida

Con `CommandType.StoredProcedure` el primer argumento pasa a ser el nombre del procedimiento, no SQL:

```csharp
var parametros = new DynamicParameters();
parametros.Add("@ClienteId", 92);
parametros.Add("@TotalGastado", dbType: DbType.Decimal, direction: ParameterDirection.Output);

var pedidos = await conexion.QueryAsync<Pedido>(
    "dbo.ObtenerPedidosDeCliente", parametros, commandType: CommandType.StoredProcedure);

decimal total = parametros.Get<decimal>("@TotalGastado");
```

El `Get<T>` del parámetro de salida solo tiene valor **después** de que la consulta se haya ejecutado y los resultados se hayan consumido. Con `buffered: false` (más abajo) leerlo antes de terminar de enumerar devuelve `null`. Para el valor de retorno del procedimiento (`RETURN`), el `direction` es `ParameterDirection.ReturnValue`.

## Mapeo de columnas a propiedades

La convención es simple: **el nombre de la columna debe coincidir con el nombre de la propiedad**, sin distinguir mayúsculas. `PrecioUnitario` mapea a `PrecioUnitario`, y también a `preciounitario`.

Cuando no coinciden, hay tres salidas. La primera y mejor es un **alias en el SQL**:

```sql
SELECT precio_unitario AS PrecioUnitario, cantidad AS Cantidad FROM LineasPedido;
```

La segunda, si toda la base de datos usa `snake_case`, es un interruptor global que se activa una vez al arrancar:

```csharp
Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true;
```

Con eso, `precio_unitario` mapea a `PrecioUnitario` sin alias. Es **estático y global** para todo el proceso: si la aplicación habla con dos bases de datos de convenciones distintas, esta opción no te sirve.

Lo peligroso es la tercera situación: **una columna que no encaja con ninguna propiedad se ignora en silencio, y una propiedad sin columna se queda en su valor por defecto**. No hay excepción ni aviso.

```csharp
// La tabla tiene la columna Estado, pero el SELECT la olvida
var producto = await conexion.QuerySingleAsync<Producto>(
    "SELECT Id, Nombre, Precio FROM Productos WHERE Id = @id", new { id = 17 });

Console.WriteLine(producto.Estado);   // "" — no null, no error: el valor por defecto
```

Ese `""` viaja por la aplicación hasta que alguien compara `Estado == "Activo"` y no cuadra nada. Es el fallo más común y más difícil de encontrar de todo Dapper.

El mapeo a **tipos primitivos** funciona igual, sin clase intermedia:

```csharp
var nombres = await conexion.QueryAsync<string>("SELECT Nombre FROM Productos");
int total = await conexion.ExecuteScalarAsync<int>("SELECT COUNT(*) FROM Pedidos");
```

## Multi-mapping: un pedido con su cliente dentro

Un `JOIN` devuelve una fila plana, pero tú quieres un `Pedido` con un `Cliente` dentro. Para eso hay una sobrecarga que recibe **los tipos de cada trozo de la fila** y una función que los une:

```csharp
const string sql = """
    SELECT p.Id, p.Estado, p.Total,   c.Id, c.Nombre, c.Email
    FROM Pedidos p JOIN Clientes c ON c.Id = p.ClienteId
    WHERE p.Id = @id;
    """;

var pedidos = await conexion.QueryAsync<Pedido, Cliente, Pedido>(
    sql,
    (pedido, cliente) => { pedido.Cliente = cliente; return pedido; },
    new { id = 4711 },
    splitOn: "Id");

var pedido = pedidos.Single();   // pedido.Cliente ya viene relleno
```

**`splitOn` nombra la columna donde empieza el siguiente objeto.** Dapper recibe una lista plana de seis columnas y necesita saber que las tres primeras son del `Pedido` y las tres siguientes del `Cliente`; el punto de corte es la segunda columna llamada `Id`. Su valor por defecto es precisamente `"Id"`, así que aquí podríamos habérnoslo ahorrado.

De esto se deducen las dos reglas que hay que respetar: **el orden de las columnas en el `SELECT` debe coincidir con el orden de los tipos genéricos**, y la columna de corte debe estar presente. Si las claves se llaman de otra forma —`p.PedidoId`, `c.ClienteId`— y no lo dices, el error es explícito:

```
System.ArgumentException: When using the multi-mapping APIs ensure you set the splitOn
param if you have keys other than Id
Parameter name: splitOn
```

Se arregla nombrando la columna por la que cortar, y con varios objetos se listan separadas por comas: `splitOn: "ClienteId"`, o `splitOn: "ClienteId,LineaId"` para tres tipos.

Ojo con el fallo silencioso hermano: si el `splitOn` apunta a una columna que **sí existe pero en el sitio equivocado**, no hay excepción. Dapper corta donde le dijiste y el segundo objeto sale con las propiedades a su valor por defecto.

### El patrón 1-N: un pedido con sus líneas

Con una colección el `JOIN` repite el pedido en cada línea, así que la función de unión recibe el mismo pedido tres veces. La solución es un diccionario que acumule:

```csharp
const string sql = """
    SELECT p.Id, p.Estado, p.Total,   l.Id, l.ProductoId, l.Cantidad, l.PrecioUnitario
    FROM Pedidos p JOIN LineasPedido l ON l.PedidoId = p.Id
    WHERE p.Id = @id;
    """;

var pedidosPorId = new Dictionary<int, Pedido>();

await conexion.QueryAsync<Pedido, LineaPedido, Pedido>(
    sql,
    (pedido, linea) =>
    {
        if (!pedidosPorId.TryGetValue(pedido.Id, out var acumulado))
        {
            acumulado = pedido;
            pedidosPorId.Add(acumulado.Id, acumulado);
        }
        acumulado.Lineas.Add(linea);
        return acumulado;
    },
    new { id = 4711 },
    splitOn: "Id");

var pedido = pedidosPorId[4711];   // con sus 3 líneas
```

El resultado que devuelve `QueryAsync` contiene el mismo pedido repetido una vez por línea; se descarta y se lee del diccionario. `Pedido.Lineas` debe estar inicializada (`public List<LineaPedido> Lineas { get; } = new();`), porque Dapper crea el `Pedido` pero no toca esa propiedad.

Este patrón deja de escalar cuando el "1" tiene muchas columnas y el "N" muchas filas: el `JOIN` repite todos los campos del pedido en cada línea. Ahí es mejor `QueryMultipleAsync`.

## `QueryMultipleAsync`: varios result sets en una sola ida

Un SQL con varias sentencias separadas por `;` devuelve varios conjuntos de resultados, y se leen en orden:

```csharp
const string sql = """
    SELECT Id, Estado, Total FROM Pedidos WHERE Id = @id;
    SELECT Id, ProductoId, Cantidad, PrecioUnitario FROM LineasPedido WHERE PedidoId = @id;
    SELECT COUNT(*) FROM LineasPedido WHERE PedidoId = @id;
    """;

using var multi = await conexion.QueryMultipleAsync(sql, new { id = 4711 });

var pedido = await multi.ReadSingleAsync<Pedido>();
var lineas = (await multi.ReadAsync<LineaPedido>()).ToList();
int numeroLineas = await multi.ReadSingleAsync<int>();
```

Una sola ida y vuelta a la base de datos, sin duplicar las columnas del pedido. El precio es que **el orden de las llamadas `Read*` tiene que corresponder exactamente al orden de las sentencias**, y el compilador no lo comprueba: intercambiar dos `Read` da un error de mapeo en ejecución, o algo peor, un mapeo silencioso a valores por defecto. El `using` del `GridReader` es obligatorio.

## Buffered y unbuffered

Por defecto Dapper es **buffered**: lee el `DataReader` de principio a fin, construye la lista completa en memoria, cierra el lector y te la devuelve. Es lo que quieres el 95 % de las veces, porque la conexión se libera cuanto antes.

Con un result set grande —exportar el histórico entero de `Pedidos`— esa lista es el problema: un millón de objetos en memoria a la vez. En la API síncrona se desactiva con `buffered: false`; en la asíncrona, el equivalente es `QueryUnbufferedAsync<T>`, que devuelve un `IAsyncEnumerable<T>`:

```csharp
// un objeto vivo a la vez, no un millón
foreach (var p in conexion.Query<Pedido>("SELECT Id, Total FROM Pedidos", buffered: false))
    EscribirFilaCsv(p);

await foreach (var p in conexion.QueryUnbufferedAsync<Pedido>("SELECT Id, Total FROM Pedidos"))
    await EscribirFilaCsvAsync(p);
```

El riesgo es este: **sin buffer, la conexión y el lector siguen abiertos mientras enumeras**. Si devuelves el `IEnumerable` desde un método cuyo `using` cierra la conexión, la enumeración ocurre cuando ya no hay conexión:

```csharp
// ❌ el using cierra la conexión antes de que nadie itere
public IEnumerable<Pedido> Historico()
{
    using var conexion = new SqlConnection(cadenaConexion);
    return conexion.Query<Pedido>("SELECT ... FROM Pedidos", buffered: false);
}
```

```
System.InvalidOperationException: Invalid attempt to call Read when reader is closed.
```

Con `buffered: true` ese mismo código funciona, porque la lista ya está completa cuando el `using` cierra. La consecuencia: sin buffer, **consume dentro del mismo ámbito** que abre la conexión, o devuelve un `IAsyncEnumerable` desde un método `async` que mantenga el `using` vivo.

## Transacciones

Dapper no gestiona transacciones: usa las del driver. `BeginTransaction` devuelve un objeto que hay que pasar **en cada llamada**:

```csharp
using var conexion = new SqlConnection(cadenaConexion);
await conexion.OpenAsync();                       // aquí sí, explícito
using var tx = await conexion.BeginTransactionAsync();

await conexion.ExecuteAsync(
    "UPDATE Productos SET Stock = Stock - @cantidad WHERE Id = @productoId",
    new { cantidad = 2, productoId = 17 }, transaction: tx);

await conexion.ExecuteAsync(
    "UPDATE Pedidos SET Estado = 'Confirmado' WHERE Id = @id",
    new { id = 4711 }, transaction: tx);

await tx.CommitAsync();   // sin Commit, el Dispose del using hace ROLLBACK
```

Nota el `OpenAsync()` explícito: `BeginTransaction` necesita la conexión ya abierta, y aquí Dapper no va a abrirla por ti.

Si olvidas el `transaction: tx` en una de las llamadas no ocurre nada silencioso — SQL Server lo rechaza con un mensaje bastante claro:

```
System.InvalidOperationException: ExecuteReader requires the command to have a transaction
when the connection assigned to the command is in a pending local transaction.
The Transaction property of the command has not been initialized.
```

Esa obligación de arrastrar `tx` por todas las firmas es la diferencia más visible con EF Core, donde el `SaveChangesAsync` ya envuelve todo en una transacción implícita.

## Type handlers: tipos que Dapper no sabe mapear

Dapper conoce los primitivos, `string`, `decimal`, `DateTime`, `Guid`, `byte[]` y sus versiones nullable. Con un tipo propio —un *value object* como `Sku`— no sabe qué hacer:

```
System.Data.DataException: Error parsing column 1 (Sku=ZAP-42 - String)
```

Se resuelve con un `SqlMapper.TypeHandler<T>`, que traduce en las dos direcciones:

```csharp
public readonly record struct Sku(string Valor);

public sealed class SkuTypeHandler : SqlMapper.TypeHandler<Sku>
{
    public override Sku Parse(object value) => new Sku((string)value);

    public override void SetValue(IDbDataParameter parameter, Sku value)
    {
        parameter.DbType = DbType.AnsiString;   // la columna es varchar(20)
        parameter.Size = 20;
        parameter.Value = value.Valor;
    }
}
```

Se registra **una sola vez** al arrancar la aplicación, antes de la primera consulta:

```csharp
SqlMapper.AddTypeHandler(new SkuTypeHandler());
```

Desde ese momento, `new { sku = new Sku("ZAP-42") }` funciona como parámetro y una propiedad `public Sku Sku { get; set; }` se rellena al mapear. Los casos típicos son *value objects*, tipos de terceros como `NodaTime.Instant`, y columnas `nvarchar` que guardan JSON y quieres deserializar al vuelo.

## Dapper frente a EF Core

No es una elección moral. Cada uno gana en cosas distintas:

| | Dapper | EF Core |
|---|---|---|
| Quién escribe el SQL | Tú | El proveedor, desde LINQ |
| Change tracking | ❌ No existe | ✅ `SaveChanges` calcula el `UPDATE` |
| Migraciones de esquema | ❌ Aparte ([DbUp](../migraciones-de-esquema/DbUp.md), [Flyway](../migraciones-de-esquema/Flyway.md)) | ✅ `dotnet ef migrations` |
| Errores de nombre de columna | En ejecución | En compilación (si es LINQ) |
| Consulta de lectura compleja | Escribes el `JOIN` exacto | LINQ traducible, o `FromSql` |
| Sobrecarga por fila | Casi nula | Notable con tracking activado |
| Grafo de objetos con relaciones | A mano, con `splitOn` | `Include` |
| Renombrar una columna del esquema | Buscar en cadenas de texto | El compilador te avisa |

La comparativa de rendimiento que se cita siempre es real —Dapper está muy cerca del driver a pelo— pero rara vez es el argumento decisivo. Lo que de verdad pesa es el **coste de mantenimiento del SQL a mano**: añadir una columna a `Productos` significa tocar todos los `INSERT` y `SELECT` que la mencionan, y nadie te avisa de los que has olvidado.

Por eso el patrón más habitual en proyectos serios es **usar los dos**, y aprovechar que EF Core expone su propia conexión:

```csharp
// EF Core para escrituras: tracking, validación, transacción implícita
var producto = await contexto.Productos.FindAsync(17);
producto.Precio = 82.50m;
await contexto.SaveChangesAsync();

// Dapper para la consulta del informe, sobre la MISMA conexión y transacción
var resumen = await contexto.Database.GetDbConnection()
    .QueryAsync<ResumenVentas>(sqlDelInforme, new { desde });
```

La división que funciona: **EF Core manda en las escrituras y en el modelo de dominio; Dapper en las lecturas que se resisten a LINQ** (agregaciones, funciones de ventana, `PIVOT`, CTEs recursivas). Si compartes conexión dentro de una transacción de EF Core, pasa también `transaction: contexto.Database.CurrentTransaction?.GetDbTransaction()`.

## Cuándo NO usar Dapper

- **CRUD masivo y repetitivo.** Si la aplicación son treinta pantallas de "listar, crear, editar, borrar" sobre tablas planas, escribir a mano los cuatro SQL de cada tabla es trabajo sin recompensa. Ahí EF Core gana claramente.
- **Cuando necesitas un modelo de dominio con estado.** Sin change tracking, cada modificación es un `UPDATE` explícito que alguien tiene que acordarse de escribir.
- **Cuando el esquema cambia a menudo.** El SQL en cadenas de texto no se refactoriza; en un esquema inestable, cada migración deja consultas rotas que solo aparecen en ejecución.
- **Para inserciones masivas.** `ExecuteAsync` con una lista hace **un viaje por elemento**: 50 000 productos son 50 000 `INSERT`. Eso es `SqlBulkCopy` o un *table-valued parameter*, y se trata en [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md).
- **Si el equipo no está cómodo escribiendo SQL.** Dapper no te protege de un `SELECT *` sin índice ni de un `JOIN` cartesiano. Delega en ti justo la parte que un ORM hace razonablemente bien de serie.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| Una propiedad llega con `0`, `null` o `""` sin motivo | La columna no está en el `SELECT` o se llama distinto. Dapper ignora en silencio lo que no encaja: revisa la lista de columnas y pon un alias |
| `Must declare the scalar variable "@id"` | El SQL usa `@id` pero el objeto de parámetros no tiene una propiedad `id` (típico al renombrarla: `new { productoId = 17 }`) |
| `Sequence contains more than one element` | `QuerySingleAsync` sobre una consulta que devuelve varias filas: o falta un filtro, o el `JOIN` multiplica |
| `When using the multi-mapping APIs ensure you set the splitOn param...` | Las claves del `JOIN` no se llaman `Id`; indica `splitOn: "ClienteId"` |
| El segundo objeto del multi-mapping viene vacío | `splitOn` corta en el sitio equivocado, o el orden de columnas del `SELECT` no coincide con el de los tipos genéricos |
| `Data is Null. This method or property cannot be called on Null values.` | Una columna nullable mapeada a un tipo no nullable (`DateTime` en vez de `DateTime?`). Dapper lo envuelve como `Error parsing column 3 (FechaEnvio=<null>)` |
| `Invalid attempt to call Read when reader is closed.` | Un `IEnumerable` sin buffer enumerado después de cerrar la conexión |
| `The ConnectionString property has not been initialized` o `Connection must be valid and open` | La conexión se dispuso antes (un `using` mal colocado, o guardar la conexión en un campo de un servicio *singleton*) |
| Un método tarda segundos y la base de datos registra cientos de consultas idénticas | N+1: una consulta dentro de un bucle. Sustituye por un `JOIN`, un `IN @ids` o `QueryMultipleAsync` |
| `Incorrect syntax near ','` en una cláusula `IN` | Escribiste `IN (@ids)` con paréntesis; con la expansión de listas va sin ellos |
| Una consulta con `IN @ids` no devuelve nada aunque haya datos | La lista llegó vacía, y Dapper genera `IN (SELECT @ids WHERE 1 = 0)` |
| `Error parsing column 1 (Sku=ZAP-42 - String)` | Tipo propio sin `TypeHandler` registrado |

El N+1 merece el cálculo concreto, porque es el error que más rendimiento cuesta y el más fácil de escribir sin darse cuenta:

```csharp
// ❌ 1 consulta + 1 por pedido
var pedidos = await conexion.QueryAsync<Pedido>("SELECT Id, ClienteId FROM Pedidos");
foreach (var pedido in pedidos)
    pedido.Cliente = await conexion.QuerySingleAsync<Cliente>(
        "SELECT Id, Nombre FROM Clientes WHERE Id = @id", new { id = pedido.ClienteId });
```

Con 500 pedidos son 501 idas y vueltas. A 2 ms de latencia de red cada una —optimista, en la misma red local— eso es **algo más de un segundo consumido solo en esperar**, con la base de datos casi ociosa. El `JOIN` con multi-mapping hace el mismo trabajo en 2 ms.

## Buenas prácticas avanzadas

- **Fuerza el tipo exacto de los parámetros `string` o perderás los índices.** Dapper envía las cadenas como `nvarchar(4000)` por defecto. Si la columna es `varchar(20)`, SQL Server no puede comparar directamente: aplica una conversión implícita a `nvarchar` **sobre la columna**, y eso convierte un *index seek* en un *index scan* de la tabla entera. Se ve en el plan como `CONVERT_IMPLICIT`. La cura es `new DbString { Value = sku, IsAnsi = true, IsFixedLength = false, Length = 20 }` o un `DynamicParameters` con `DbType.AnsiString` y `size`.
- **Vigila la caché: cada texto de SQL distinto es un plan y un mapeador nuevos.** Dapper indexa sus mapeadores compilados por la terna (texto del SQL, tipo de retorno, forma de los parámetros), y SQL Server hace lo propio con los planes. La expansión de listas genera un texto distinto por cada tamaño (`IN (@ids1,@ids2)` frente a `IN (@ids1,@ids2,@ids3)`), así que un endpoint que recibe listas de longitud variable llena la caché de planes de un solo uso: `SqlMapper.Settings.PadListExpansions = true` redondea el número de parámetros y reduce las variantes a un puñado. Para el resto, `SqlMapper.GetCachedSQLCount()` te dice cuántas entradas hay; si crece sin parar en un servicio estable, tienes SQL construido dinámicamente donde deberías tener constantes.
- **Los parámetros no sirven para identificadores: usa una lista blanca.** `ORDER BY @columna` no funciona —un parámetro es un valor, no un nombre de columna—, así que la ordenación dinámica es el sitio donde casi todo el mundo acaba concatenando. La única forma segura es traducir la entrada del usuario a un literal escrito por ti: `var columna = permitidas.GetValueOrDefault(entrada, "Nombre");` con `permitidas` como diccionario cerrado. Cualquier otra cosa es inyección de SQL con pasos extra.
- **Un `ExecuteAsync` con una colección no es una inserción masiva.** Dapper acepta `ExecuteAsync(sql, listaDeProductos)` y ejecuta la sentencia una vez por elemento, reutilizando el comando. Es cómodo para diez filas y catastrófico para diez mil: son diez mil viajes. Para volumen, *table-valued parameter* o `SqlBulkCopy`.
- **Mapea a tipos inmutables con constructor.** Dapper sabe usar el constructor de un `record` cuando los nombres de los parámetros coinciden con las columnas, y eso elimina de golpe la clase de bug del mapeo silencioso a valor por defecto: un parámetro de constructor que no recibe valor no compila el mapeo, mientras que una propiedad `set` que nadie rellena se queda tan callada como su valor por defecto.

## Recursos didácticos

- [Learn Dapper](https://www.learndapper.com/) — la referencia práctica más completa, organizada por escenario (parámetros, multi-mapping, transacciones) con ejemplos ejecutables. Es corta, y eso dice mucho de la superficie de la librería.
- [Repositorio DapperLib/Dapper](https://github.com/DapperLib/Dapper) — el código fuente cabe en unos pocos ficheros. `SqlMapper.cs` es donde se resuelven la expansión de listas y la caché de mapeadores; leerlo es la forma más rápida de dejar de tratar a Dapper como una caja negra.
- [Dapper Tutorial](https://dappertutorial.net/) — recetario con la variante síncrona y asíncrona de cada método, útil para consultar rápido qué sobrecarga acepta `splitOn` o `commandType`.
- [Benchmarks de micro-ORMs .NET (RawDataAccessBencher)](https://github.com/FransBouma/RawDataAccessBencher) — comparativa reproducible de Dapper, EF Core, el driver a pelo y otros, con memoria asignada además de tiempo. Mira siempre la columna de asignaciones: es la que explica las diferencias bajo carga.

---

*En resumen: Dapper es la capa mínima entre tu SQL y tus objetos C# — elige el método según cuántas filas esperas, parametriza siempre, entiende `splitOn`, y acepta que a cambio de controlar la consulta te quedas sin la red de seguridad que un ORM completo te daba.*
