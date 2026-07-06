# EF Core Migrations

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

EF Core Migrations es el sistema de migraciones integrado en [Entity Framework Core](../../desarrollo-web/de-wpf-a-web/Entity-Framework-Core.md): genera automáticamente los cambios de esquema SQL necesarios a partir de las diferencias entre tus clases C# y el estado actual de la base de datos.

Esta ficha se centra en cómo EF Core versiona y aplica esos cambios de esquema; si buscas cómo se usa EF Core como ORM en general (consultas, `DbContext`, *change tracking*), consulta su [ficha dedicada](../../desarrollo-web/de-wpf-a-web/Entity-Framework-Core.md).

## ¿Por qué existe?

Cuando usas EF Core, el esquema de tu base de datos debería reflejar siempre la forma de tus clases C#: si añades una propiedad `StockMinimo` a `Producto`, la tabla `Productos` necesita esa misma columna. Mantener eso sincronizado a mano (con `ALTER TABLE` escritos uno a uno) es tedioso y propenso a errores, y además EF Core ya conoce exactamente qué ha cambiado entre tu modelo actual y el anterior. EF Core Migrations automatiza justo eso: compara el modelo actual contra el último "snapshot" guardado y genera el SQL necesario para llevar la base de datos del estado anterior al nuevo.

## ¿Cuándo y para qué se usa?

Siempre que trabajas con EF Core como ORM y necesitas evolucionar el esquema junto con tus clases: añadir una tabla nueva para las reseñas de un producto en una tienda online, añadir una columna para el número de seguimiento de un pedido, o cambiar la longitud máxima de un campo de texto. Es la opción natural cuando el resto de tu acceso a datos ya pasa por EF Core, porque no introduce una herramienta ni un flujo de trabajo adicional.

## Lo mínimo que necesitas saber

**1. Instalar las herramientas de EF Core**

```bash
dotnet tool install --global dotnet-ef
dotnet add package Microsoft.EntityFrameworkCore.Design
```

**2. Generar una migración a partir de cambios en tus clases**

```csharp
public class Producto
{
    public int Id { get; set; }
    public string Nombre { get; set; } = "";
    public decimal Precio { get; set; }
    public int StockMinimo { get; set; } // propiedad nueva
}
```

```bash
dotnet ef migrations add AnadirStockMinimoAProducto
```

Esto crea un fichero en `Migrations/` con dos métodos, `Up()` (aplicar el cambio) y `Down()` (deshacerlo):

```csharp
public partial class AnadirStockMinimoAProducto : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<int>(
            name: "StockMinimo",
            table: "Productos",
            type: "int",
            nullable: false,
            defaultValue: 0);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(name: "StockMinimo", table: "Productos");
    }
}
```

**3. Aplicar las migraciones pendientes**

```bash
dotnet ef database update
```

**4. Generar el SQL sin aplicarlo, para revisarlo o llevarlo a un pipeline**

```bash
dotnet ef migrations script --idempotent -o migracion.sql
```

`--idempotent` genera un script que comprueba qué migraciones faltan por aplicar, seguro de ejecutar varias veces sin duplicar cambios.

**5. Aplicar migraciones automáticamente al arrancar (con cautela)**

```csharp
using var scope = app.Services.CreateScope();
var context = scope.ServiceProvider.GetRequiredService<TiendaContext>();
await context.Database.MigrateAsync();
```

Cómodo en desarrollo; en producción se prefiere aplicar las migraciones como paso explícito del despliegue, no al arrancar cada instancia.

## Lo que NO hace

- **No detecta cambios hechos a mano en la base de datos** — si alguien modificó una tabla fuera de EF Core, el historial de migraciones y el esquema real divergen sin que la herramienta lo note hasta que algo falla.
- **No siempre genera el SQL más eficiente** — para cambios delicados en tablas grandes, conviene revisar y a veces reescribir el SQL generado.
- **No gestiona el orden de despliegue entre varios microservicios** — si varios servicios comparten base de datos (algo generalmente desaconsejado), coordinar sus migraciones sigue siendo responsabilidad tuya.

## Buenas prácticas avanzadas

- **Genera siempre el script con `migrations script` antes de aplicar en producción** — ver el SQL exacto que se va a ejecutar (no solo confiar en el nombre de la migración) es la única forma de detectar antes de tiempo un `DROP COLUMN` inesperado o un bloqueo largo sobre una tabla grande.
- **Una migración, un propósito** — agrupar varios cambios no relacionados en una sola migración ("cambios varios de la sprint") hace mucho más difícil revertir uno sin arrastrar los demás. Genera una migración por cambio lógico, aunque sean varias en el mismo día.
- **No mezcles cambios de esquema con cambios de datos masivos en la misma migración** — un `Sql("UPDATE Productos SET ...")` sobre millones de filas dentro de una migración bloquea el despliegue tanto como bloquearía ese `UPDATE` ejecutado suelto. Sepáralo en un proceso de backfill aparte.
- **Cuidado con los cambios "destructivos" silenciosos** — renombrar una propiedad en tu clase C# sin más, EF Core lo interpreta por defecto como "borrar la columna vieja y crear una nueva", perdiendo los datos existentes. Para un renombrado real, hay que editar la migración generada para usar `RenameColumn` en vez de `DropColumn` + `AddColumn`.
- **Revisa el snapshot del modelo (`ModelSnapshot.cs`) en las revisiones de código** — es el fichero que EF Core usa para saber "de dónde venimos"; un conflicto de *merge* mal resuelto ahí puede hacer que la siguiente migración generada sea incorrecta sin que nada avise en tiempo de compilación.

---

*En resumen: EF Core Migrations traduce los cambios en tus clases C# a SQL versionado automáticamente — rápido de usar, pero conviene mirar siempre el SQL que genera antes de aplicarlo a datos reales.*
