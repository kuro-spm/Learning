# Entity Framework Core

## ¿Qué es?

Entity Framework Core (EF Core) es el **ORM** oficial de .NET: la librería que traduce entre tus clases C# y las tablas de una base de datos. Tú escribes consultas LINQ sobre objetos y modificas propiedades; EF Core genera el `SELECT`, el `INSERT` y el `UPDATE` que hacen falta.

> Un **ORM** (*Object-Relational Mapper*) es una librería que hace de puente entre el mundo de los objetos (clases, propiedades, referencias) y el mundo relacional (tablas, columnas, claves ajenas), para no escribir SQL a mano en cada operación.

## ¿Por qué existe?

Porque buena parte del código de acceso a datos es trabajo repetitivo sin ninguna decisión interesante: copiar columnas a propiedades y, al revés, componer un `UPDATE` con exactamente las columnas que han cambiado. Esto es actualizar el precio de un producto con [Dapper](Dapper.md), que ya ahorra el mapeo manual del driver:

```csharp
var producto = await conexion.QuerySingleAsync<Producto>(
    "SELECT Id, Nombre, Precio, Estado FROM Productos WHERE Id = @id", new { id = 17 });
producto.Precio = 82.50m;
await conexion.ExecuteAsync(
    "UPDATE Productos SET Nombre = @Nombre, Precio = @Precio, Estado = @Estado WHERE Id = @Id",
    producto);
```

El `UPDATE` lo escribes tú, columna a columna, y si mañana `Productos` gana una columna hay que acordarse de añadirla aquí y en los otros doce sitios donde se actualiza. Lo mismo con EF Core:

```csharp
var producto = await contexto.Productos.FindAsync(17);
producto.Precio = 82.50m;
await contexto.SaveChangesAsync();
```

No hay SQL. EF Core sabe qué fila era, compara el objeto con la copia que guardó al cargarlo, ve que solo cambió `Precio` y emite un `UPDATE` de una sola columna. Esa memoria de lo que ha cambiado se llama *change tracking* y es lo que separa un ORM completo de un micro-ORM.

La analogía: [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) es el cable —abre la conexión y manda texto—, [Dapper](Dapper.md) es un traductor que rellena tus objetos con lo que llega por el cable, y EF Core es un intérprete que además **habla por ti**: le dices qué quieres en C# y él decide qué SQL enviar.

## ¿Cuándo y para qué se usa?

Es el camino estándar de acceso a datos en ASP.NET Core cuando quieres productividad: CRUD sobre muchas tablas, agregados con relaciones y un esquema que evoluciona con el código. Cuanto más se parezca tu trabajo a "leer un objeto, modificarlo y guardarlo", más gana EF Core; cuanto más se parezca a "un informe con cuatro `JOIN` y una función de ventana", menos.

El ejemplo conductor es una tienda online sobre la base de datos **`TiendaDB`**, con las tablas `Productos`, `Pedidos`, `LineasPedido` y `Clientes`, las entidades C# `Producto`, `Pedido`, `LineaPedido` y `Cliente`, y un `DbContext` llamado **`TiendaContext`**. El pedido que iremos siguiendo es el **#4711**.

---

## Instalación y el primer ciclo completo

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet tool install --global dotnet-ef
```

El modelo son clases normales, y un `DbSet<T>` por tabla dentro de una clase que hereda de `DbContext`. Un `DbSet` no es una lista: es un punto de partida para construir consultas, y por sí solo no lee nada.

```csharp
public sealed class Producto
{
    public int Id { get; set; }
    public string Nombre { get; set; } = "";
    public decimal Precio { get; set; }
    public string Estado { get; set; } = "";
}

public sealed class TiendaContext(DbContextOptions<TiendaContext> options) : DbContext(options)
{
    public DbSet<Producto> Productos => Set<Producto>();
    public DbSet<Pedido> Pedidos => Set<Pedido>();
    public DbSet<LineaPedido> LineasPedido => Set<LineaPedido>();
    public DbSet<Cliente> Clientes => Set<Cliente>();
}
```

El registro en el contenedor de dependencias va en `Program.cs`. `AddDbContext` lo inscribe con vida **por petición** (*scoped*), que es lo correcto y tiene su propia sección más abajo:

```csharp
builder.Services.AddDbContext<TiendaContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("TiendaDB")));
```

Y ya se recibe por constructor. Una lectura y una escritura de principio a fin:

```csharp
public sealed class ServicioProductos(TiendaContext contexto)
{
    public Task<Producto?> ObtenerAsync(int id, CancellationToken ct) =>
        contexto.Productos.FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task<int> CrearAsync(string nombre, decimal precio, CancellationToken ct)
    {
        var producto = new Producto { Nombre = nombre, Precio = precio, Estado = "Activo" };
        contexto.Productos.Add(producto);
        await contexto.SaveChangesAsync(ct);
        return producto.Id;          // EF Core lo rellena con el Id que asignó la base
    }
}
```

El SQL de las dos (el `INSERT` tal como lo genera EF Core 7 y posteriores sobre SQL Server):

```sql
SELECT TOP(1) [p].[Id], [p].[Nombre], [p].[Precio], [p].[Estado]
FROM [Productos] AS [p] WHERE [p].[Id] = @__id_0;

SET IMPLICIT_TRANSACTIONS OFF;
SET NOCOUNT ON;
INSERT INTO [Productos] ([Nombre], [Precio], [Estado])
OUTPUT INSERTED.[Id]
VALUES (@p0, @p1, @p2);
```

Fíjate en el `OUTPUT INSERTED.[Id]`: EF Core no solo inserta, recupera la clave generada y la escribe de vuelta en tu objeto. Y en el `@__id_0`: el valor viaja **como parámetro**, nunca concatenado, así que la inyección de SQL no existe por construcción.

> **Las migraciones no se tratan aquí.** Cuando cambies tus clases hay que actualizar el esquema, y eso se hace con `dotnet ef migrations add` y `dotnet ef database update`. Tiene guía propia: [EF Core Migrations](../migraciones-de-esquema/EF-Core-Migrations.md).

## El modelo: convenciones, atributos y API fluida

EF Core deduce solo la mayor parte del mapeo. Las **convenciones** que explican por qué funciona sin configurar nada:

| Convención | Qué deduce |
|---|---|
| Nombre de la tabla | El de la propiedad `DbSet` (`Productos` → tabla `Productos`) |
| Clave primaria | Una propiedad `Id` o `<Entidad>Id`; si es `int`, la trata como *identity* |
| Tipo de columna | Del tipo C#: `string` → `nvarchar(max)`, `int` → `int`, `DateTime` → `datetime2` |
| Obligatoriedad | `int` y `decimal` → `NOT NULL`; `int?` y `string?` → `NULL` |
| Clave ajena | `LineaPedido.PedidoId` junto a una navegación `LineaPedido.Pedido` |

Lo que las convenciones no aciertan se configura de dos formas. Los **atributos** van sobre la propia clase; la **API fluida**, en `OnModelCreating`:

```csharp
// atributos, sobre la entidad Cliente
[Required, MaxLength(120)] public string Nombre { get; set; } = "";

// API fluida, en el DbContext
protected override void OnModelCreating(ModelBuilder modelo) =>
    modelo.Entity<Pedido>(pedido =>
    {
        pedido.Property(p => p.Total).HasPrecision(10, 2);
        pedido.Property(p => p.Estado).HasMaxLength(20).IsRequired();
        pedido.HasIndex(p => new { p.ClienteId, p.Fecha });
        pedido.Property(p => p.Version).IsRowVersion();
    });
```

| Necesidad | Atributos | API fluida |
|---|---|---|
| Longitud, obligatoriedad, nombre de columna, precisión | ✅ Sí | ✅ Sí |
| Índices únicos o filtrados | ❌ `[Index]` muy limitado | ✅ `HasIndex(...).IsUnique()` |
| Claves compuestas | ❌ No se puede | ✅ `HasKey(x => new { ... })` |
| Comportamiento del borrado en cascada | ❌ No | ✅ `OnDelete(DeleteBehavior.Restrict)` |
| Filtros globales, conversores de valor, tipos propietarios | ❌ No | ✅ Sí |
| Entidades sin dependencia de EF Core | ❌ Las contamina | ✅ Las deja limpias |

Los atributos son cómodos para tres propiedades en un prototipo. En un proyecto serio gana la API fluida por dos razones que no son de gusto: **cubre toda la superficie de configuración** (con atributos, tarde o temprano necesitas un índice único filtrado y te quedas sin herramienta) y **mantiene las entidades libres de referencias a EF Core**, así que la capa de dominio no depende del ORM. Con muchas entidades, cada configuración va a su clase `IEntityTypeConfiguration<T>` y se registran de golpe con `modelo.ApplyConfigurationsFromAssembly(...)`.

### Relaciones: un pedido con sus líneas

Una relación uno a muchos tiene tres piezas: la **clave ajena** en el lado "muchos" y una **propiedad de navegación** a cada lado.

```csharp
public sealed class Pedido
{
    public int Id { get; set; }
    public int ClienteId { get; set; }              // clave ajena
    public Cliente Cliente { get; set; } = null!;   // navegación al "uno"
    public string Estado { get; set; } = "";
    public decimal Total { get; set; }
    public byte[] Version { get; set; } = [];       // token de concurrencia
    public List<LineaPedido> Lineas { get; } = [];  // navegación al "muchos"
}

public sealed class LineaPedido
{
    public int Id { get; set; }
    public int PedidoId { get; set; }               // clave ajena
    public Pedido Pedido { get; set; } = null!;     // navegación inversa
    public int ProductoId { get; set; }
    public int Cantidad { get; set; }
    public decimal PrecioUnitario { get; set; }
}

// la configuración explícita, opcional pero recomendable por el OnDelete
modelo.Entity<Pedido>()
    .HasMany(p => p.Lineas).WithOne(l => l.Pedido)
    .HasForeignKey(l => l.PedidoId)
    .OnDelete(DeleteBehavior.Cascade);
```

Con eso, `pedido.Lineas.Add(new LineaPedido { ... })` y un `SaveChangesAsync` insertan la línea con el `PedidoId` correcto sin que tú lo asignes: EF Core lo deduce de la relación. Y `OnDelete` decide qué pasa al borrar el pedido — que borre sus líneas, que falle o que las deje huérfanas. Es una decisión de negocio disfrazada de configuración, así que conviene escribirla en lugar de heredar el valor por defecto.

## Consultar con LINQ, y el SQL que sale

Cada operador LINQ tiene un equivalente en SQL, y saber cuál es lo que impide escribir consultas caras sin darse cuenta.

```csharp
var baratos = await contexto.Productos
    .Where(p => p.Precio < precioMaximo && p.Estado == "Activo")
    .OrderBy(p => p.Nombre)
    .ToListAsync(ct);
```

```sql
SELECT [p].[Id], [p].[Nombre], [p].[Precio], [p].[Estado]
FROM [Productos] AS [p]
WHERE [p].[Precio] < @__precioMaximo_0 AND [p].[Estado] = N'Activo'
ORDER BY [p].[Nombre]
```

Un detalle con consecuencias: `precioMaximo` es una variable capturada y se convierte en **parámetro**, mientras que `"Activo"` es una constante del código y se **incrusta literalmente**. Es lo que quieres: los parámetros permiten reutilizar el plan de ejecución, y las constantes dan al motor información para elegir mejor.

Los operadores que devuelven un solo elemento se diferencian en qué hacen cuando la realidad no coincide con lo que esperabas:

| Operador | SQL | 0 filas | 2+ filas |
|---|---|---|---|
| `FirstOrDefaultAsync` | `SELECT TOP(1)` | `null` | Devuelve la primera |
| `FirstAsync` | `SELECT TOP(1)` | `InvalidOperationException` | Devuelve la primera |
| `SingleOrDefaultAsync` | `SELECT TOP(2)` | `null` | `InvalidOperationException` |
| `SingleAsync` | `SELECT TOP(2)` | `InvalidOperationException` | `InvalidOperationException` |
| `FindAsync` | `SELECT TOP(1)` por clave, o **nada** | `null` | No aplica |

El `TOP(2)` de `Single` no es un capricho: EF Core pide dos filas para poder detectar que hay más de una y lanzar. Es una fila extra de red a cambio de que un duplicado en una clave única no pase inadvertido. Usa `Single*` cuando consultas por clave y `First*` cuando de verdad pides "el más reciente" con un `OrderBy`. `FindAsync` es especial: mira primero el *change tracker* y, si la entidad ya está cargada en este contexto, **no va a la base de datos**.

`Any` y `Count` responden a preguntas distintas, y confundirlos cuesta:

```csharp
bool tienePedidos = await contexto.Pedidos.AnyAsync(p => p.ClienteId == 92, ct);
int cuantos      = await contexto.Pedidos.CountAsync(p => p.ClienteId == 92, ct);
```

```sql
-- AnyAsync: para en cuanto encuentra una fila
SELECT CASE WHEN EXISTS (SELECT 1 FROM [Pedidos] AS [p] WHERE [p].[ClienteId] = @__id_0)
       THEN CAST(1 AS bit) ELSE CAST(0 AS bit) END
-- CountAsync: recorre todas las que cumplen
SELECT COUNT(*) FROM [Pedidos] AS [p] WHERE [p].[ClienteId] = @__id_0
```

Si solo quieres saber *si existe*, `AnyAsync`: `CountAsync(...) > 0` obliga al motor a contar la última fila cuando la respuesta ya se sabía con la primera.

### Ejecución diferida: la consulta no viaja hasta que se enumera

`contexto.Productos.Where(...)` **no ejecuta nada**. Devuelve un `IQueryable<Producto>`, que es un árbol de expresiones: una descripción de la consulta que todavía se puede seguir componiendo.

```csharp
IQueryable<Producto> consulta = contexto.Productos.Where(p => p.Estado == "Activo");
if (buscarTexto is not null)
    consulta = consulta.Where(p => p.Nombre.Contains(buscarTexto));   // sigue sin ejecutar

var pagina = await consulta.OrderBy(p => p.Nombre).Skip(20).Take(20).ToListAsync(ct);
```

Solo la última línea manda SQL, y manda **una sola** consulta con filtro, ordenación y paginación dentro. Lo que materializa una consulta es: `ToListAsync`/`ToArrayAsync`/`ToDictionaryAsync`, los `First…`/`Single…`/`Count…`/`Any…`/`Sum…`, un `foreach` (o `await foreach` sobre `AsAsyncEnumerable()`) y un `AsEnumerable()` seguido de enumeración.

El error que sale de aquí es materializar demasiado pronto:

```csharp
// ❌ SELECT sin WHERE: trae la tabla entera y filtra en memoria
var caros = (await contexto.Productos.ToListAsync(ct)).Where(p => p.Precio > 500).ToList();

// ✅ filtra el motor y solo viajan las filas que importan
var caros = await contexto.Productos.Where(p => p.Precio > 500).ToListAsync(ct);
```

Con 200 000 productos, la primera versión construye 200 000 objetos en memoria para quedarse con doce. Regla: **materializa lo más tarde posible**, y desconfía de cualquier `ToList` que no esté en la última línea del método.

## La proyección es la optimización que más rinde

Si la pantalla muestra cuatro campos, traer la entidad entera es pagar por columnas que nadie va a leer.

```csharp
// ❌ trae las 30 columnas de Productos, incluida una Descripcion nvarchar(max)
var listado = await contexto.Productos.Where(p => p.Estado == "Activo").ToListAsync(ct);

// ✅ trae exactamente lo que se muestra
var listado = await contexto.Productos
    .Where(p => p.Estado == "Activo")
    .Select(p => new ProductoResumen(p.Id, p.Nombre, p.Precio))
    .ToListAsync(ct);
```

```sql
-- ❌
SELECT [p].[Id], [p].[Nombre], [p].[Precio], [p].[Estado], [p].[Descripcion],
       [p].[Sku], [p].[Peso], [p].[FechaAlta], ...  -- 30 columnas
FROM [Productos] AS [p] WHERE [p].[Estado] = N'Activo'
-- ✅
SELECT [p].[Id], [p].[Nombre], [p].[Precio]
FROM [Productos] AS [p] WHERE [p].[Estado] = N'Activo'
```

Con una `Descripcion` de 2 KB de media y 5 000 productos activos, la primera versión mueve unos **10 MB** por la red y los materializa en el montón de .NET; la segunda, unos 200 KB. Y hay un segundo efecto menos conocido: **una proyección a un tipo que no es una entidad desactiva el *tracking* por sí sola**, porque no hay entidad que rastrear — no necesita `AsNoTracking`. Una tercera ventaja: si la proyección atraviesa una navegación (`Cliente = p.Cliente.Nombre`), EF Core genera el `JOIN` en la misma consulta sin `Include` y sin traer el `Cliente` completo.

## Consultas del servidor frente a consultas del cliente

EF Core traduce lo que sabe traducir: operadores aritméticos y de comparación, `string.Contains`/`StartsWith`/`ToUpper`/`Substring`, los componentes de `DateTime`, agregaciones, `Contains` sobre una colección en memoria y lo que expone `EF.Functions` (`Like`, `DateDiffDay`, texto completo). Lo que **no** sabe traducir es un método tuyo, porque no tiene forma de saber qué hace:

```csharp
// ❌ EsPromocionable es un método de tu clase: no existe en SQL Server
var promociones = await contexto.Productos.Where(p => EsPromocionable(p)).ToListAsync(ct);
```

Desde EF Core 3.0 esto **no se evalúa en el cliente en silencio** —como hacía EF Core 2, descargando la tabla entera para filtrarla en memoria—, sino que falla:

```
System.InvalidOperationException: The LINQ expression 'DbSet<Producto>()
    .Where(p => EsPromocionable(p))' could not be translated.
Either rewrite the query in a form that can be translated, or switch to client evaluation
explicitly by inserting a call to 'AsEnumerable', 'AsAsyncEnumerable', 'ToList', or 'ToListAsync'.
```

El cambio fue deliberado y es una mejora: un fallo ruidoso es mejor que una consulta que en desarrollo va bien con 50 filas y en producción descarga un millón. Hay dos salidas y solo una es buena:

```csharp
// ✅ reescribir la condición en algo traducible
var promociones = await contexto.Productos
    .Where(p => p.Estado == "Activo" && p.Precio > 100 && p.Stock > 0).ToListAsync(ct);

// ⚠️ forzar la evaluación local a sabiendas: primero se filtra en SQL todo lo que se pueda
var promociones = (await contexto.Productos.Where(p => p.Estado == "Activo").ToListAsync(ct))
    .Where(EsPromocionable)    // esto ya es LINQ to Objects, en memoria
    .ToList();
```

La segunda solo es aceptable cuando el filtro traducible ya reduce el resultado a algo pequeño. Si el `AsEnumerable()` está antes de cualquier `Where`, has escrito un `SELECT *` de la tabla.

## Change tracking: cómo EF Core sabe qué ha cambiado

Cuando una consulta materializa una entidad, EF Core guarda además una **copia de sus valores originales** (un *snapshot*) en el `ChangeTracker` del contexto. Al llamar a `SaveChangesAsync`, compara el objeto actual con esa copia y deduce qué columnas han cambiado. Cada entidad rastreada tiene un estado:

| Estado | Significa | Qué genera `SaveChangesAsync` |
|---|---|---|
| `Unchanged` | Cargada y sin modificar | Nada |
| `Modified` | Alguna propiedad difiere del *snapshot* | `UPDATE` de las columnas cambiadas |
| `Added` | Nueva, todavía sin fila | `INSERT` |
| `Deleted` | Marcada con `Remove` | `DELETE` |
| `Detached` | El contexto no la conoce | Nada |

De ahí sale el ciclo leer → modificar → guardar, sin escribir un solo `UPDATE`:

```csharp
var pedido = await contexto.Pedidos.FirstAsync(p => p.Id == 4711, ct);
pedido.Estado = "Confirmado";
await contexto.SaveChangesAsync(ct);
```

```sql
SET IMPLICIT_TRANSACTIONS OFF;
SET NOCOUNT ON;
UPDATE [Pedidos] SET [Estado] = @p0
OUTPUT 1
WHERE [Id] = @p1 AND [Version] = @p2;
```

Tres cosas en ese SQL: solo aparece `Estado`, no las demás columnas; el `WHERE` incluye `[Version]`, que es el *token* de concurrencia de más abajo; y el `OUTPUT 1` es cómo EF Core comprueba que la fila realmente se actualizó.

`SaveChangesAsync` es además **atómico**: envuelve todos los cambios pendientes en una transacción implícita. Si insertas un pedido y tres líneas y el tercer `INSERT` falla, no queda nada a medias. No hace falta `BeginTransaction` para eso.

## `AsNoTracking`: el interruptor de las consultas de lectura

El *snapshot* no es gratis: por cada entidad materializada, EF Core copia todos sus valores escalares, crea la entrada en el `ChangeTracker` y la indexa por clave.

```csharp
var catalogo = await contexto.Productos
    .AsNoTracking()                      // ✅ solo lectura: nada que rastrear
    .Where(p => p.Estado == "Activo")
    .ToListAsync(ct);
```

El número concreto: para 1 000 productos con 30 propiedades, el *tracking* implica **30 000 valores copiados** al *snapshot* y 1 000 entradas de diccionario vivas hasta que muere el contexto. Después, `SaveChangesAsync` recorre esas 1 000 entradas comparando 30 000 valores para descubrir que ninguno ha cambiado. Todo eso para devolver un JSON.

Dos efectos secundarios que hay que conocer. El primero: **las entidades devueltas no están vinculadas al contexto**, así que modificarlas y llamar a `SaveChangesAsync` no hace nada. El segundo: sin *tracking*, si dos filas del resultado apuntan al mismo `Cliente`, EF Core crea **dos objetos `Cliente` distintos**, porque la resolución de identidad vive precisamente en el `ChangeTracker`; cuando eso importa existe `AsNoTrackingWithIdentityResolution()`, que mantiene un diccionario de identidades para la consulta sin registrar cambios.

En una API mayoritariamente de lectura tiene sentido invertir el valor por defecto, y que las pocas consultas que modifican pidan `AsTracking()` explícitamente:

```csharp
options.UseSqlServer(cadena).UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
```

## Cargar relaciones: cuatro estrategias

`pedido.Cliente` es `null` hasta que alguien lo carga. Hay cuatro formas:

```csharp
// 1. Eager: una consulta con JOIN, dicho por adelantado
var pedido = await contexto.Pedidos
    .Include(p => p.Cliente)
    .Include(p => p.Lineas).ThenInclude(l => l.Producto)
    .FirstAsync(p => p.Id == 4711, ct);

// 2. Explicit: la entidad ya está cargada y pides la relación cuando la necesitas
await contexto.Entry(pedido).Collection(p => p.Lineas).LoadAsync(ct);
await contexto.Entry(pedido).Reference(p => p.Cliente).LoadAsync(ct);

// 3. Lazy: se carga sola al acceder (requiere proxies y navegaciones virtual)
Console.WriteLine(pedido.Cliente.Nombre);   // aquí se dispara una consulta invisible

// 4. Proyección: no cargas la relación, pides los datos que necesitas de ella
var resumen = await contexto.Pedidos.Where(p => p.Id == 4711)
    .Select(p => new { p.Id, p.Total, Cliente = p.Cliente.Nombre, Lineas = p.Lineas.Count })
    .FirstAsync(ct);
```

| Estrategia | Consultas | Cuándo usarla | Riesgo |
|---|---|---|---|
| *Eager* (`Include`) | 1 | Sabes de antemano qué necesitas y vas a modificar el grafo | Explosión cartesiana con varias colecciones |
| *Explicit* (`Load`) | 1 + 1 por relación | La relación se necesita solo en algunas ramas del código | Fácil de convertir en N+1 dentro de un bucle |
| *Lazy* (proxies) | 1 + 1 por acceso | Casi nunca en un backend | N+1 invisible; excepciones si el contexto ya murió |
| Proyección (`Select`) | 1 | Lectura pura: listados, APIs, informes | Ninguno, pero no sirve para modificar |

Para leer, la proyección es la respuesta correcta casi siempre; para modificar un agregado completo, `Include`. *Lazy loading* es la que más problemas causa: convierte cada acceso a una propiedad en una consulta que no se ve en el código, y basta con serializar el objeto a JSON para que el serializador recorra todas las navegaciones y dispare cientos de consultas.

### La explosión cartesiana y `AsSplitQuery`

`Include` de **una** colección repite los datos del padre en cada hija — molesto pero manejable. `Include` de **dos** colecciones las multiplica entre sí, porque el `JOIN` no tiene forma de mantenerlas separadas:

```csharp
var pedido = await contexto.Pedidos
    .Include(p => p.Lineas)
    .Include(p => p.Pagos)
    .FirstAsync(p => p.Id == 4711, ct);
```

El pedido #4711 tiene 10 líneas y 5 pagos:

```
10 líneas × 5 pagos = 50 filas devueltas para UN solo pedido
y en cada una de las 50, todas las columnas del pedido repetidas
```

EF Core deduplica bien al construir los objetos —acabas con 10 líneas y 5 pagos, no 50— pero las 50 filas ya han viajado por la red. Con 20 líneas y 10 pagos serían 200; el crecimiento es multiplicativo, y EF Core avisa en el log:

```
warn: Microsoft.EntityFrameworkCore.Query[20504]
      Compiling a query which loads related collections for more than one collection
      navigation, either via 'Include' or through projection, but no
      'QuerySplittingBehavior' has been configured.
```

`AsSplitQuery()` emite **una consulta por colección**: tres consultas de 1, 10 y 5 filas en lugar de una de 50. Compensa cuando el producto cartesiano es grande o el padre tiene columnas anchas; no compensa cuando las colecciones son pequeñas, porque pagas dos idas y vueltas de red extra. Regla: **si vuelven muchas más filas que objetos necesitas, parte la consulta.** Dos avisos: las consultas divididas no comparten transacción por defecto, así que una escritura concurrente puede dejarte padre e hijos de momentos distintos, y con paginación exigen un `ORDER BY` totalmente determinista.

## El problema N+1

Es el error de rendimiento más común de cualquier ORM, y merece el cálculo:

```csharp
// ❌ 1 consulta para los pedidos + 1 por cada pedido para su cliente
var pedidos = await contexto.Pedidos.Where(p => p.Fecha >= desde).ToListAsync(ct);
foreach (var pedido in pedidos)
    Console.WriteLine(pedido.Cliente.Nombre);      // consulta oculta aquí

// ✅ una sola consulta con el JOIN
var pedidos = await contexto.Pedidos.Where(p => p.Fecha >= desde)
    .Include(p => p.Cliente).ToListAsync(ct);

// ✅✅ mejor todavía si es solo para leer
var pedidos = await contexto.Pedidos.Where(p => p.Fecha >= desde)
    .Select(p => new { p.Id, p.Total, Cliente = p.Cliente.Nombre }).ToListAsync(ct);
```

Con 500 pedidos, la versión mala hace **501 consultas**. A 2 ms de latencia por consulta —optimista, en la misma red local—:

```
501 consultas × 2 ms  =  1 002 ms
```

**Más de un segundo de pura espera**, con la base de datos casi ociosa y la CPU sin hacer nada. El `Include` hace el mismo trabajo en un viaje de 3 ms. Lo peor del N+1 es que sea invisible, y eso es exactamente lo que aporta el *lazy loading*: `pedido.Cliente.Nombre` no se distingue de leer una propiedad en memoria. Sin *lazy loading*, ese mismo código lanza `NullReferenceException` en la primera prueba — un fallo mucho más barato que un endpoint lento en producción.

## Escrituras

```csharp
// Añadir uno, y su grafo: las líneas se insertan solas con el PedidoId correcto
var pedido = new Pedido { ClienteId = 92, Estado = "Pendiente", Total = 149.90m };
pedido.Lineas.Add(new LineaPedido { ProductoId = 17, Cantidad = 2, PrecioUnitario = 74.95m });
contexto.Pedidos.Add(pedido);

await contexto.Productos.AddRangeAsync(productosNuevos, ct);   // añadir varios

var producto = await contexto.Productos.FindAsync([17], ct);   // modificar
producto!.Precio = 82.50m;

contexto.Pedidos.Remove(otroPedido);                           // borrar

await contexto.SaveChangesAsync(ct);   // una transacción para todo lo anterior
```

`AddAsync`/`AddRangeAsync` son asíncronos solo por un detalle: algunos generadores de valores (como `HiLo`) necesitan ir a la base a reservar claves. Con `int` *identity*, `Add` síncrono es equivalente.

En las **inserciones múltiples** EF Core agrupa las sentencias en lotes, en una sola ida y vuelta. En EF Core 7 y posteriores sobre SQL Server, varias filas con clave *identity* se insertan con un `INSERT` de varios `VALUES` y un `OUTPUT` que devuelve todas las claves; en versiones anteriores hacía falta un `MERGE` con `OUTPUT` para correlacionar cada clave con su fila. El tamaño de lote lo controla `MaxBatchSize`.

Pero **EF Core no es la herramienta para cargar cien mil filas**: aun agrupando, materializa cien mil entidades, las rastrea todas y construye el SQL de cada una.

```
100 000 INSERT vía EF Core      ≈ decenas de segundos y cientos de MB rastreados
100 000 filas con SqlBulkCopy   ≈ 2-4 s y memoria constante
```

Para eso está `SqlBulkCopy`, que se trata en [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md). Y para **modificaciones y borrados masivos**, EF Core 7 trajo la respuesta que muy poca gente conoce: `ExecuteUpdateAsync` y `ExecuteDeleteAsync` traducen la consulta a **una sola sentencia**, sin cargar ni rastrear nada.

```csharp
// ❌ carga 40 000 productos, los rastrea y emite 40 000 UPDATE
var viejos = await contexto.Productos.Where(p => p.FechaAlta < corte).ToListAsync(ct);
foreach (var p in viejos) p.Estado = "Descatalogado";
await contexto.SaveChangesAsync(ct);

// ✅ una sentencia, cero entidades en memoria
await contexto.Productos.Where(p => p.FechaAlta < corte)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Estado, "Descatalogado"), ct);
```

```sql
UPDATE [p] SET [p].[Estado] = N'Descatalogado'
FROM [Productos] AS [p] WHERE [p].[FechaAlta] < @__corte_0
```

## Concurrencia: el pedido #4711 modificado por dos personas

Sin protección, la última escritura gana en silencio. Ana y Luis abren el pedido #4711 en estado `Pendiente`; Ana lo pasa a `Confirmado` y Luis a `Cancelado` dos segundos después. El `UPDATE` de Luis pisa el de Ana y nadie se enteró de que hubo un conflicto.

La solución es un ***token* de concurrencia**: una columna que cambia en cada escritura y que EF Core incluye en el `WHERE`. En SQL Server, el tipo `rowversion` lo hace solo:

```csharp
public byte[] Version { get; set; } = [];                              // en Pedido
modelo.Entity<Pedido>().Property(p => p.Version).IsRowVersion();       // o [Timestamp]
```

El `UPDATE` de Luis lleva la versión que él leyó, la fila ya tiene otra, y por tanto **afecta a 0 filas**:

```sql
UPDATE [Pedidos] SET [Estado] = @p0
OUTPUT 1
WHERE [Id] = 4711 AND [Version] = 0x00000000000007D1;
```

```
Microsoft.EntityFrameworkCore.DbUpdateConcurrencyException: The database operation was
expected to affect 1 row(s), but actually affected 0 row(s); data may have been modified
or deleted since entities were loaded.
```

Resolverlo es elegir una política: **avisar al usuario** (la más honesta en una UI), **que gane la base** (recargar y descartar el cambio) o **que gane el cliente** (sobrescribir):

```csharp
try { await contexto.SaveChangesAsync(ct); }
catch (DbUpdateConcurrencyException ex)
{
    var entrada = ex.Entries.Single();
    var enBase = await entrada.GetDatabaseValuesAsync(ct)
                 ?? throw new InvalidOperationException("El pedido #4711 ya no existe.");

    entrada.OriginalValues.SetValues(enBase);   // que gane el cliente, con la versión actual
    await contexto.SaveChangesAsync(ct);
}
```

`ex.Entries` da las entidades en conflicto y `GetDatabaseValuesAsync` los valores actuales de la fila, que es justo lo que hace falta para mostrar un "esto ha cambiado mientras editabas" en lugar de un 500.

## Transacciones

`SaveChangesAsync` ya es transaccional para sus cambios pendientes. Una transacción explícita hace falta cuando necesitas **más de un `SaveChanges`** en la misma unidad de trabajo, o cuando quieres meter en ella algo que no pasa por EF Core:

```csharp
await using var tx = await contexto.Database.BeginTransactionAsync(ct);
try
{
    contexto.Pedidos.Add(pedido);
    await contexto.SaveChangesAsync(ct);        // el pedido ya tiene Id
    contexto.Auditoria.Add(new Auditoria { PedidoId = pedido.Id });
    await contexto.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
}
catch { await tx.RollbackAsync(ct); throw; }    // el Dispose sin Commit también revierte
```

Un detalle que sorprende: si has activado `EnableRetryOnFailure()` para reintentar errores transitorios, EF Core **prohíbe** `BeginTransactionAsync` a secas, porque la estrategia de reintento necesita poder reejecutar el bloque completo. Hay que envolverlo en `contexto.Database.CreateExecutionStrategy().ExecuteAsync(async () => { ... })`.

Y lo más útil en la práctica: **la conexión y la transacción de EF Core se comparten con [Dapper](Dapper.md)** para esa consulta que LINQ no expresa bien.

```csharp
await using var tx = await contexto.Database.BeginTransactionAsync(ct);

var resumen = await contexto.Database.GetDbConnection().QueryAsync<ResumenVentas>(
    """
    SELECT c.Nombre, SUM(p.Total) AS Total,
           RANK() OVER (ORDER BY SUM(p.Total) DESC) AS Puesto
    FROM Pedidos p JOIN Clientes c ON c.Id = p.ClienteId
    WHERE p.Fecha >= @desde GROUP BY c.Nombre
    """,
    new { desde }, transaction: tx.GetDbTransaction());
```

`GetDbConnection()` devuelve la `SqlConnection` que EF Core ya tiene abierta y `GetDbTransaction()` la transacción viva. Sin pasar `transaction:`, SQL Server rechaza el comando porque la conexión tiene una transacción local pendiente.

## SQL en crudo desde EF Core

Cuando LINQ no llega, no hace falta salir de EF Core. `FromSql` devuelve entidades a partir de tu SQL, y acepta una **cadena interpolada** — ahí está el detalle importante: recibe un `FormattableString`, no un `string`, así que EF Core inspecciona los huecos de la interpolación y **los convierte en parámetros**.

```csharp
var pedidos = await contexto.Pedidos
    .FromSql($"SELECT * FROM dbo.PedidosConRetraso({dias})")   // → ...ConRetraso(@p0)
    .Where(p => p.Estado == "Pendiente")                        // se puede seguir componiendo
    .AsNoTracking()
    .ToListAsync(ct);
```

```csharp
// ❌ FromSqlRaw con concatenación: inyección de SQL
.FromSqlRaw("SELECT * FROM Pedidos WHERE Estado = '" + estado + "'")
// ✅ interpolada: el valor viaja como parámetro
.FromSql($"SELECT * FROM Pedidos WHERE Estado = {estado}")
```

`FromSqlRaw` existe para cuando construyes el SQL dinámicamente, y entonces los parámetros van aparte: `FromSqlRaw("... WHERE Estado = {0}", estado)`. Sus límites: el `SELECT` debe devolver **todas** las columnas de la entidad con los nombres del modelo, y no admite varios *result sets*.

Para escalares hay `SqlQuery<T>`, que no necesita entidad — la columna debe llamarse `Value`. Y para sentencias que no devuelven filas, `ExecuteSqlAsync`:

```csharp
var totales = await contexto.Database
    .SqlQuery<decimal>($"SELECT SUM(Total) AS Value FROM Pedidos WHERE ClienteId = {92}")
    .ToListAsync(ct);
```

## Ver qué está pasando

Si puedes ver el SQL, puedes diagnosticar por tu cuenta. Esta es la sección que convierte la guía en algo utilizable.

```csharp
builder.Services.AddDbContext<TiendaContext>(options => options
    .UseSqlServer(cadena)
    .LogTo(Console.WriteLine, LogLevel.Information)
    .EnableSensitiveDataLogging(builder.Environment.IsDevelopment())
    .EnableDetailedErrors(builder.Environment.IsDevelopment()));
```

- **`LogTo`** escribe cada comando ejecutado con su tiempo. Es el interruptor que más rápido revela un N+1: si una petición imprime 500 `SELECT` casi idénticos, ya sabes qué buscar. Con `LogTo(..., [DbLoggerCategory.Database.Command.Name], LogLevel.Information)` se limita a los comandos y se quita el ruido.
- **`EnableSensitiveDataLogging`** sustituye `@__id_0` por el valor real, que es lo que permite copiar la consulta y pegarla en un cliente SQL. Y es exactamente por eso que **no va a producción**: esos valores incluyen emails, DNI y cualquier dato personal que use la consulta, y acaban en texto plano en unos logs que suelen estar menos protegidos que la propia base de datos.
- **`EnableDetailedErrors`** añade, cuando falla la lectura de un valor, de qué propiedad se trataba. Cuesta rendimiento; solo en desarrollo.

Y para inspeccionar una consulta **sin ejecutarla**, `ToQueryString()`:

```csharp
var consulta = contexto.Pedidos.Where(p => p.Fecha >= desde).Include(p => p.Lineas);
Console.WriteLine(consulta.ToQueryString());
```

Devuelve el SQL exacto, con la declaración de los parámetros lista para pegar en un cliente y pedir el plan de ejecución. Es la forma de comprobar si tu `Where` acabó en el `WHERE` y de ver cuántas columnas trae de verdad esa consulta que creías ligera.

## El ciclo de vida del `DbContext`

`AddDbContext` registra el contexto como ***scoped***: uno nuevo por petición HTTP, destruido al terminarla. No es un detalle de configuración, son dos limitaciones reales.

**No es *thread-safe*.** El `ChangeTracker` es un diccionario mutable sin sincronización, así que dos operaciones a la vez sobre el mismo contexto dan:

```
System.InvalidOperationException: A second operation was started on this context instance
before a previous operation completed. This is usually caused by different threads
concurrently using the same instance of DbContext.
```

```csharp
// ❌ dos consultas en paralelo sobre el mismo contexto
var tareaPedidos  = contexto.Pedidos.ToListAsync(ct);
var tareaClientes = contexto.Clientes.ToListAsync(ct);
await Task.WhenAll(tareaPedidos, tareaClientes);
```

La otra causa habitual es un `await` olvidado, que deja la primera operación en vuelo. La solución no es sincronizar: es **secuenciar** las consultas (que además comparten conexión y no compiten por el pool) o usar un contexto por operación paralela.

**Acumula todo lo que rastrea.** Un contexto *singleton* nunca suelta las entidades cargadas: la memoria crece hasta el reinicio y las entidades quedan obsoletas frente a la base sin que nada las refresque. Un contexto por petición muere pequeño.

Para lo que no es una petición HTTP —un `IHostedService`, un trabajo en segundo plano, un componente de Blazor Server que vive mientras el usuario mire la página— está `IDbContextFactory<T>`:

```csharp
builder.Services.AddDbContextFactory<TiendaContext>(o => o.UseSqlServer(cadena));

await using var contexto = await factory.CreateDbContextAsync(ct);
var pendientes = await contexto.Pedidos.Where(p => p.Estado == "Pendiente").ToListAsync(ct);
```

Regla: **un contexto por unidad de trabajo**, y las unidades de trabajo cortas.

## Cuándo NO usar EF Core

- **Informes con agregaciones, funciones de ventana, `PIVOT` o CTE recursivas.** LINQ traduce algunas, mal y con esfuerzo. Ahí escribes el SQL y lo lees con [Dapper](Dapper.md).
- **Cargas masivas.** Cien mil filas no son cien mil entidades rastreadas: son `SqlBulkCopy` (ver [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md)) o `ExecuteUpdateAsync`/`ExecuteDeleteAsync` si es una modificación en bloque.
- **Cuando el esquema lo controla otro equipo.** Si no puedes generar migraciones ni negociar un cambio de columna, la mitad del valor de EF Core desaparece y el modelo se convierte en un espejo frágil de algo que se mueve sin avisarte.
- **Cuando el equipo necesita ver el SQL exacto de cada operación.** En un sistema donde cada plan de ejecución está revisado, delegar la generación de SQL a un traductor que puede cambiar entre versiones menores es un riesgo que no compensa.

El patrón sensato no es elegir: es **usar los dos**. EF Core manda en las escrituras y en el modelo de dominio, donde el *change tracking*, la transacción implícita y las [migraciones](../migraciones-de-esquema/EF-Core-Migrations.md) son un regalo; Dapper manda en las lecturas que se resisten a LINQ. Comparten driver, cadena de conexión, pool y —si quieres— conexión y transacción, así que mezclarlos no cuesta nada.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `The LINQ expression '...' could not be translated` | Un método tuyo dentro del `Where`/`Select`. Reescríbelo en operadores traducibles, o filtra en SQL lo que puedas y usa `AsEnumerable()` a sabiendas |
| `A second operation was started on this context instance before a previous operation completed` | Dos consultas en paralelo sobre el mismo contexto (`Task.WhenAll`), o un `await` olvidado. Secuencia, o un contexto por operación |
| `The instance of entity type 'Producto' cannot be tracked because another instance with the same key value for {'Id'} is already being tracked` | Un `Attach`/`Update` de una entidad cuya clave ya está en el `ChangeTracker`. Carga la existente y modifícala, o usa `contexto.Entry(x).CurrentValues.SetValues(...)` |
| `DbUpdateConcurrencyException: ... expected to affect 1 row(s), but actually affected 0 row(s)` | Otro lo cambió antes y el *token* de concurrencia no coincide, la fila se borró, o el `UPDATE` apunta a una clave que no existe |
| Una petición tarda segundos y el log muestra cientos de `SELECT` casi iguales | N+1: acceso a una navegación dentro de un bucle. `Include` o, mejor, proyección |
| Una consulta devuelve muchas más filas de las que hay objetos | `Include` de dos colecciones: producto cartesiano. `AsSplitQuery()` |
| Modificas una entidad y `SaveChangesAsync` no hace nada | Venía de una consulta con `AsNoTracking()` (o del `NoTracking` global): no está rastreada |
| `SaveChangesAsync` devuelve 0 y no genera SQL | No hay cambios detectados: modificaste una copia, un DTO, o una propiedad sin `set` accesible |
| `DbUpdateException: The INSERT statement conflicted with the FOREIGN KEY constraint` | Insertas una hija con una clave ajena que no existe, o padre e hija en llamadas separadas y falló el orden |
| `Cannot insert explicit value for identity column` | Asignaste `Id` a mano en una entidad con clave *identity* |
| `NullReferenceException` al leer `pedido.Cliente.Nombre` | La navegación no se cargó y no hay *lazy loading*. Es un buen error: te avisa en desarrollo de lo que sería un N+1 |
| Los `decimal` llegan truncados a dos decimales sin aviso | La columna se creó como `decimal(18,2)` por convención. Configura `HasPrecision` |

## Buenas prácticas avanzadas

- **La frontera del `IQueryable` es donde se decide si el trabajo lo hace el motor o tu proceso.** Sobre un `IQueryable`, cada `Where`/`OrderBy`/`Skip`/`Take` se acumula en el árbol de expresiones y acaba en el SQL; en cuanto llamas a `ToList()` o lo declaras como `IEnumerable<T>`, todo lo que encadenes después se ejecuta en memoria sobre lo ya descargado. Y ese cambio de tipo es invisible: un método de repositorio que devuelve `IEnumerable<Producto>` en lugar de `IQueryable<Producto>` convierte silenciosamente en filtros en memoria todos los filtros de sus llamantes. Devuelve `IQueryable` solo si quieres que se compongan, y asume entonces que la consulta final la escribe quien llama.
- **`Contains` sobre una lista genera un SQL distinto por cada tamaño de lista, y eso te llena la caché de planes.** `Where(p => ids.Contains(p.Id))` se traducía expandiendo un parámetro por elemento (`IN (@p0, @p1, @p2)`), así que una búsqueda con tres ids y otra con cuatro son **dos sentencias distintas** para SQL Server: dos compilaciones y dos planes de un solo uso. En un endpoint que recibe listas de longitud variable, eso ensucia la caché hasta desalojar los planes que sí se reutilizan. EF Core 8 cambió la traducción a `OPENJSON` sobre un único parámetro JSON, lo que estabiliza el texto y el plan; si estás en una versión anterior, o si el `OPENJSON` te da un plan peor, `EF.Parameter` y `EF.Constant` permiten forzar cada estrategia. Es de esas cosas que solo se ven mirando `sys.dm_exec_cached_plans`.
- **`ExecuteUpdateAsync` y `ExecuteDeleteAsync` no pasan por el `ChangeTracker`, con todo lo que eso implica.** Se ejecutan al instante —no esperan a `SaveChangesAsync`, así que no entran en su transacción implícita—, no comprueban el *token* de concurrencia, no disparan interceptores ni un `SaveChanges` sobrescrito (adiós a tu auditoría automática y a tus campos `ModificadoEn`) y dejan obsoleta cualquier entidad ya cargada con esas claves. Son la herramienta correcta para modificar cuarenta mil filas y la incorrecta para modificar una dentro de una unidad de trabajo.
- **`OnModelCreating` se ejecuta una sola vez por combinación de tipo de contexto y opciones, y el modelo queda en caché para todo el proceso.** De ahí sale un fallo que parece imposible: si metes en `OnModelCreating` algo que depende del estado de la petición —el identificador de inquilino para un filtro global, la cultura de quien consulta— el primer valor que llegue queda grabado en el modelo y todas las peticiones siguientes usarán ese. No falla: devuelve datos de otro. Los filtros dinámicos se hacen con `HasQueryFilter` leyendo de un campo del contexto, que sí se evalúa por consulta; y si de verdad necesitas modelos distintos, con un `IModelCacheKeyFactory` propio que incluya la variable en la clave de caché.
- **Configura la precisión de cada `decimal` o aceptarás truncamientos silenciosos.** Sin `HasPrecision`, EF Core mapea `decimal` a `decimal(18,2)` en SQL Server y avisa una sola vez en el log con `No store type was specified for the decimal property`. Con dos decimales, un precio unitario de `74.9950` se guarda como `74.99`: nadie lanza una excepción, el importe simplemente no cuadra. En cualquier modelo con dinero, porcentajes o cantidades fraccionarias, la precisión es parte del contrato y va escrita.
- **Mide con el log de comandos encendido antes de optimizar nada.** La intuición sobre qué consulta es cara falla casi siempre: el `Include` que parecía inocente traía 200 filas, el listado "ligero" arrastraba un `nvarchar(max)`, y el endpoint lento resultó ser un N+1 dentro de un mapeador automático. `LogTo` con la categoría de comandos —o mejor, un interceptor que registre las consultas que pasan de 100 ms junto a su `ToQueryString()`— convierte el rendimiento en datos. Optimizar sin ese log es cambiar código al azar.

## Documentación oficial

- [Documentación de EF Core](https://learn.microsoft.com/ef/core/) — la referencia, y sorprendentemente buena como material de aprendizaje. Empieza por *Querying data* y *Change tracking*: desarrollan con más ejemplos los conceptos de esta guía.
- [EF Core: rendimiento y diagnóstico](https://learn.microsoft.com/ef/core/performance/) — la parte que casi nadie lee y la que más cambia el resultado. Ve ahí cuando un endpoint sea lento y el log de comandos no te baste: proyecciones, *tracking*, consultas divididas, consultas compiladas.
- [Breaking changes de cada versión](https://learn.microsoft.com/ef/core/what-is-new/) — lectura obligada **antes** de subir de versión mayor. El SQL generado cambia entre versiones (la traducción de `Contains`, la forma de los `INSERT` en lote), y esos cambios se notan en producción antes que en las pruebas.
- [Repositorio dotnet/efcore](https://github.com/dotnet/efcore) — para cuando la documentación no responde. Buscar ahí un mensaje de error literal en las *issues* funciona mejor que una búsqueda genérica, porque suele aparecer la discusión de por qué se comporta así.

## Recursos didácticos

- [LINQPad](https://www.linqpad.net/) — permite escribir consultas LINQ contra tu propio `DbContext` y ver **el SQL generado y el resultado al lado**, sin compilar un proyecto. Es la forma más rápida de interiorizar la traducción de LINQ a SQL: escribes, miras el SQL, cambias un operador y comparas.
- [Use The Index, Luke!](https://use-the-index-luke.com/) — no habla de EF Core, y por eso mismo completa esta guía: explica cómo el motor decide usar o no un índice. Sabiendo eso, el SQL que EF Core genera deja de ser una caja negra y se convierte en algo que puedes juzgar.

---

*En resumen: EF Core te deja tratar la base de datos como objetos C# y consultas LINQ, y a cambio te pide una sola disciplina — saber qué SQL está generando: proyecta en lugar de traer entidades enteras, `AsNoTracking` en toda lectura, `Include` en lugar de un bucle, y el log de comandos encendido mientras desarrollas.*
