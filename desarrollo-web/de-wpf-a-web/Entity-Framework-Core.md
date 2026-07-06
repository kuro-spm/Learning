# Entity Framework Core

## ¿Qué es?

Entity Framework Core (EF Core) es el ORM oficial de .NET: una herramienta que traduce
entre tus clases C# y las tablas de una base de datos. Tú trabajas con objetos
(`Producto`, `Pedido`) y EF Core se encarga de generar el SQL para leerlos y guardarlos.

> Un **ORM** (Object-Relational Mapper) es una librería que hace de puente entre el mundo
> de los objetos (clases, propiedades) y el mundo relacional (tablas, columnas, filas),
> para que no tengas que escribir SQL a mano para cada operación.

## ¿Por qué existe?

En la web, el estado importante ya no vive en memoria como en WPF (donde una colección en
el `ViewModel` bastaba mientras la ventana estuviera abierta). El servidor se reinicia,
atiende a miles de clientes y puede tener varias copias funcionando a la vez: los datos
que deben perdurar tienen que ir a una base de datos. EF Core existe para que guardar y
recuperar esos datos sea tan natural como trabajar con objetos C#, sin escribir SQL para
cada consulta ni mapear columnas a mano.

> Si en WPF llegaste a usar EF para enlazar la UI con una base de datos, EF Core es su
> evolución multiplataforma. Si solo guardabas cosas en memoria o en ficheros, esta es la
> pieza que te faltaba para persistir datos de verdad en el servidor.

## ¿Cuándo y para qué se usa?

Siempre que tu aplicación necesite recordar datos entre peticiones y entre reinicios: los
productos y pedidos de una tienda online, las entradas de un blog, las tareas de una app
de tareas, las usuarias registradas. Es el camino estándar de acceso a datos en ASP.NET
Core cuando quieres productividad y no necesitas exprimir cada milisegundo de cada
consulta.

## Lo mínimo que necesitas saber

**1. Tus clases son las tablas, y un `DbContext` es la puerta de entrada**

Defines tus entidades como clases normales y un `DbContext` que las agrupa. Ese contexto
es tu sesión con la base de datos:

```csharp
public class Producto
{
    public int Id { get; set; }
    public string Nombre { get; set; } = "";
    public decimal Precio { get; set; }
}

public class TiendaContext : DbContext
{
    public DbSet<Producto> Productos { get; set; } // ≈ la tabla Productos
    public TiendaContext(DbContextOptions<TiendaContext> options) : base(options) { }
}
```

**2. Se registra como una dependencia más**

En `Program.cs` lo registras igual que cualquier otro servicio (ver [Inyección de
dependencias](Inyeccion-de-Dependencias.md)) y luego lo recibes por el constructor donde
lo necesites:

```csharp
builder.Services.AddDbContext<TiendaContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("BaseDatos")));
```

**3. Consultas con LINQ, no con SQL**

Haces las consultas con LINQ sobre los `DbSet`, y EF Core las traduce a SQL por ti:

```csharp
var baratos = await _context.Productos
    .Where(p => p.Precio < 20)
    .OrderBy(p => p.Nombre)
    .ToListAsync();
```

**4. Guardar es modificar objetos y llamar a `SaveChangesAsync`**

EF Core lleva un *change tracking*: observa qué objetos has añadido, cambiado o borrado, y
al guardar genera el SQL necesario en un solo paso:

```csharp
var producto = new Producto { Nombre = "Camiseta", Precio = 19.99m };
_context.Productos.Add(producto);
await _context.SaveChangesAsync(); // aquí se ejecuta el INSERT

producto.Precio = 17.99m;
await _context.SaveChangesAsync(); // ahora un UPDATE; EF detecta el cambio solo
```

**5. Las migraciones mantienen la base de datos al día**

Cuando cambias tus clases (añades una propiedad, una entidad nueva), generas una
*migración* que actualiza el esquema de la base de datos para que coincida, sin escribir
el `ALTER TABLE` a mano:

```bash
dotnet ef migrations add AnadirStockAProducto
dotnet ef database update
```

## Lo que NO hace

- **No es siempre la opción más rápida** — para consultas muy complejas o críticas en rendimiento, a veces se combina con un micro-ORM como [Dapper](../../bases-de-datos/acceso-a-datos-dotnet/Dapper.md).
- **No esconde la base de datos del todo** — conviene entender qué SQL genera para evitar consultas accidentalmente pesadas.
- **No valida tu negocio** — comprobar reglas y permisos sigue siendo responsabilidad de tu código.

## Buenas prácticas avanzadas

- **`AsNoTracking()` en toda consulta de solo lectura** — el *change tracking* que hace tan cómodo guardar tiene un coste: EF toma una "foto" de cada objeto que carga por si lo modificas. En una consulta que solo va a mostrarse (un listado, un informe), ese trabajo es puro desperdicio de memoria y CPU. `_context.Productos.AsNoTracking().Where(...)` es de las optimizaciones con mejor relación esfuerzo/beneficio que existen.
- **Proyecta con `Select` en vez de cargar entidades enteras** — si la pantalla solo necesita nombre y precio, pedir la entidad completa (y peor, con `Include` de sus relaciones) trae columnas y tablas de más. `Select(p => new ProductoResumenDto(p.Nombre, p.Precio))` genera un SQL que trae exactamente eso. Además mata de raíz el problema **N+1**: recorrer una lista accediendo a `pedido.Cliente.Nombre` sin haberlo incluido dispara una consulta extra *por cada elemento*, invisible en desarrollo y letal en producción.
- **Filtra y pagina en la base de datos, no en memoria** — la línea es sutil: sobre `IQueryable`, los `Where`/`Skip`/`Take` se traducen a SQL; pero en cuanto llamas a `ToList()` (o conviertes a `IEnumerable`), todo lo que encadenes después se ejecuta en memoria sobre lo ya descargado. Un `ToList()` colocado antes de tiempo trae la tabla entera para quedarse con 10 filas. Regla: materializa (`ToListAsync`) lo más tarde posible.
- **El `DbContext` es de una petición y de un solo hilo** — está registrado como `Scoped` por una razón: no es *thread-safe* y acumula todo lo que trackea. Guardarlo en un singleton, compartirlo entre hilos o lanzarle dos consultas en paralelo (`Task.WhenAll` sobre el mismo contexto) produce errores intermitentes del tipo "A second operation was started on this context". Un contexto por petición, y en trabajos en segundo plano, uno por operación.
- **Lee el SQL de tus migraciones antes de aplicarlas en producción** — `dotnet ef migrations script` genera el SQL que se va a ejecutar. Revisarlo te salva de sorpresas que la migración no ve venir: renombrar una propiedad puede traducirse en "borrar columna + crear columna nueva" (adiós datos), y un cambio de tipo puede bloquear una tabla grande varios minutos. La migración automática es comodísima; aplicarla a ciegas sobre datos reales, no.

---

*En resumen: EF Core es el ORM de .NET que te deja trabajar con la base de datos como si fueran objetos C# y consultas LINQ; es la pieza que persiste el estado que en WPF te bastaba con tener en memoria.*
