# EF Core Migrations

## ¿Qué es?

EF Core Migrations es el sistema de migraciones que viene dentro de [Entity Framework Core](../acceso-a-datos-dotnet/Entity-Framework-Core.md): compara tus clases C# con el estado que la base de datos tenía la última vez y genera por ti el SQL que hace falta para ponerlas de acuerdo.

Esta ficha trata **solo cómo EF Core versiona y aplica cambios de esquema**. Si buscas EF Core como ORM —consultas, `DbContext`, *change tracking*—, eso está en su [ficha dedicada](../acceso-a-datos-dotnet/Entity-Framework-Core.md). Y el concepto general de migración (la tabla de control, imperativo frente a declarativo, qué motores tienen DDL transaccional) está en [Migraciones de esquema](Migraciones-de-Esquema.md).

## ¿Por qué existe?

Con EF Core, el esquema real debería ser siempre el reflejo de tus clases. Si añades una propiedad `StockMinimo` a `Producto`, la tabla `Productos` necesita esa columna; si no la tiene, la primera consulta falla en ejecución con un `Invalid column name 'StockMinimo'`. Mantener eso a mano significa escribir dos veces el mismo cambio: una en C# y otra en un `ALTER TABLE`. La segunda se olvida.

Lo interesante es que EF Core ya tiene la información para hacerlo solo: guarda en el repositorio una fotografía del modelo —el *snapshot*— y sabe exactamente qué has cambiado desde entonces.

```bash
dotnet ef migrations add AnadirStockMinimoAProducto
```

Piensa en el *snapshot* como en el último `commit` de tu esquema: la migración es el `diff`. Esa es la idea central de la herramienta, y también el origen de casi todos sus problemas, porque ese fichero se puede desincronizar.

## ¿Cuándo y para qué se usa?

Siempre que EF Core ya sea tu vía de acceso a datos y el esquema evolucione con el código: añadir la tabla de reseñas de un producto, el número de seguimiento a un pedido, ampliar la longitud de un campo. Es la opción natural porque no introduce ninguna herramienta nueva.

En esta ficha el ejemplo conductor es una **tienda online**: base de datos `TiendaDB`, un `TiendaContext` con las tablas `Productos`, `Pedidos` y `Clientes`, y el pedido **#4711** como fila de referencia.

## Instalación y el primer ciclo completo

Dos piezas: la herramienta de línea de comandos (global, una vez por máquina) y el paquete de diseño (por proyecto, porque el comando necesita cargar tu modelo).

```bash
dotnet tool install --global dotnet-ef
dotnet add package Microsoft.EntityFrameworkCore.Design
```

**Paso 1: cambia la clase.**

```csharp
public class Producto
{
    public int Id { get; set; }
    public string Nombre { get; set; } = "";
    public decimal Precio { get; set; }
    public int StockMinimo { get; set; }   // propiedad nueva
}
```

**Paso 2: genera la migración.** El nombre se queda para siempre en el historial, así que trátalo como un mensaje de commit.

```bash
> dotnet ef migrations add AnadirStockMinimoAProducto
Build succeeded.
Done. To undo this action, use 'ef migrations remove'
```

**Paso 3: lee el fichero generado.** Este paso no es opcional. Está en `Migrations/20260728091412_AnadirStockMinimoAProducto.cs`:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
    => migrationBuilder.AddColumn<int>(
        name: "StockMinimo", table: "Productos",
        type: "int", nullable: false, defaultValue: 0);

protected override void Down(MigrationBuilder migrationBuilder)
    => migrationBuilder.DropColumn(name: "StockMinimo", table: "Productos");
```

`Up()` aplica el cambio, `Down()` lo deshace. Fíjate en el `defaultValue: 0`: la columna es `NOT NULL` y las filas existentes necesitan un valor, así que EF Core inventa el *default* del tipo. Si el valor correcto para tus productos no es 0, esto es lo que hay que corregir a mano antes de aplicar.

**Paso 4: aplícala.**

```bash
> dotnet ef database update
Applying migration '20260728091412_AnadirStockMinimoAProducto'.
Executing DbCommand [Parameters=[], CommandType='Text', CommandTimeout='30']
      ALTER TABLE [Productos] ADD [StockMinimo] int NOT NULL DEFAULT 0;
Executing DbCommand [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
      VALUES (N'20260728091412_AnadirStockMinimoAProducto', N'9.0.0');
Done.
```

Esas dos sentencias son todo el mecanismo: el `ALTER TABLE` y el registro en `__EFMigrationsHistory`, la tabla que EF Core crea para saber qué ya se aplicó.

## Anatomía de la carpeta `Migrations/`

Un `migrations add` toca tres ficheros, y solo uno es obvio:

| Fichero | Qué es | ¿Se edita a mano? |
|---|---|---|
| `20260728091412_AnadirStockMinimoAProducto.cs` | Tu migración, con `Up()` y `Down()` | **Sí**, es lo normal |
| `..._AnadirStockMinimoAProducto.Designer.cs` | El modelo completo *tal como quedó* tras esta migración | No |
| `TiendaContextModelSnapshot.cs` | El modelo completo *actual*: el «de dónde venimos» | Nunca a mano |

Los tres van al repositorio. El prefijo numérico es la fecha y hora UTC de generación, y es lo que fija el orden de aplicación: no es un contador, es un *timestamp*. El fichero peligroso es el **snapshot**: cuando ejecutas `migrations add`, EF Core no mira la base de datos, compara tu modelo actual contra `TiendaContextModelSnapshot.cs` y escribe la diferencia. De ahí se deduce lo importante: si el snapshot está mal, la migración generada está mal — y no hay ningún aviso, porque el snapshot **compila perfectamente**. Es C# válido describiendo un modelo que nunca existió.

El caso típico es un conflicto de *merge* resuelto a ojo en ese fichero. Si te quedas con la versión de una rama y descartas la columna que añadió la otra, el snapshot deja de mencionarla; la siguiente migración que alguien genere volverá a crear esa columna, y el `ALTER TABLE` fallará contra la base de datos que ya la tiene. Por eso el snapshot merece mirada explícita en las revisiones de código, y por eso sus conflictos no se resuelven a mano (más abajo, la receta).

## Los comandos que se usan de verdad

```bash
dotnet ef migrations add AnadirTelefonoACliente     # generar
dotnet ef migrations remove                         # borrar la última, si no está aplicada
dotnet ef migrations list                           # ver el estado
dotnet ef database update                           # aplicar todo lo pendiente
dotnet ef database update CrearEsquemaInicial       # volver ATRÁS hasta esa migración
dotnet ef database update 0                         # deshacerlo todo
dotnet ef database drop                             # tirar la base de datos entera
```

`migrations list` contesta «¿en qué estado estoy?»:

```
> dotnet ef migrations list
20260701102233_CrearEsquemaInicial (Applied)
20260728091412_AnadirStockMinimoAProducto (Pending)
```

`database update <Migracion>` funciona en los dos sentidos: si la migración indicada es anterior a la última aplicada, EF Core ejecuta los `Down()` en orden inverso. Es la vuelta atrás, y sale bien **siempre que los `Down()` estén escritos de verdad** — cosa que solo sabes si los has leído.

`migrations remove` solo borra la última y se niega si ya está aplicada:

```
The migration '20260728091412_AnadirStockMinimoAProducto' has already been applied to the
database. Revert it and try again. If the migration has been applied to other databases,
consider reverting its changes using a new migration instead.
```

Ese consejo final es clave: si la migración ya salió del repositorio, no se borra, se compensa con una migración nueva. Por último, `migrations script` genera el SQL sin tocar nada:

```bash
dotnet ef migrations script --idempotent -o migracion.sql
dotnet ef migrations script --from CrearEsquemaInicial --to AnadirStockMinimoAProducto
```

`--from`/`--to` delimitan el tramo. Si omites el `--from`, EF Core asume «desde cero», y ese script no sirve para una base de datos que ya tiene datos.

## `migrations bundle`: el ejecutable que no necesita el SDK

Poco conocido, y resuelve un problema real: para ejecutar `dotnet ef database update` en el servidor de destino harían falta el SDK de .NET, la herramienta `dotnet-ef` y el código fuente. En producción no quieres ninguna de las tres cosas.

```bash
> dotnet ef migrations bundle --self-contained -r win-x64 --configuration Release
Building bundle...
Done. Migrations Bundle: C:\repos\Tienda\efbundle.exe
```

Ese ejecutable se publica como artefacto del build y se lanza donde toque, con la cadena de conexión por parámetro:

```bash
./efbundle --connection "Server=sql-prod;Database=TiendaDB;User Id=deploy;Password=***"
# Applying migration '20260728091412_AnadirStockMinimoAProducto'.
# Done.
```

Ventajas concretas: el artefacto es **el mismo** que se probó en preproducción, no un `git pull` esperando que compile igual; no hace falta el código fuente en el servidor; y admite `--verbose` y `--dry-run` para ver qué haría. La pega es que sigue siendo un binario opaco para quien tiene que aprobar el cambio, así que genera además el script si alguien debe revisarlo.

## Cómo aplicar migraciones en producción

| Opción | ¿Se revisa el SQL antes? | Qué necesita el servidor | Permisos DDL de la app | Varias instancias |
|---|---|---|---|---|
| `Database.MigrateAsync()` al arrancar | ❌ Nunca lo ves | Nada extra | **Permanentes** | Todas compiten al arrancar |
| Script SQL revisado y ejecutado a mano | ✅ Es un fichero `.sql` | Un cliente SQL | Ninguno | Irrelevante: se aplica una vez |
| *Bundle* ejecutado por el pipeline | ⚠️ Solo si generas el script aparte | Nada: ejecutable autónomo | Ninguno | Irrelevante: un paso del pipeline |
| `dotnet ef database update` desde CI | ⚠️ Parcial (queda en el log) | SDK + código en el *runner* | Ninguno | Irrelevante: un paso del pipeline |

Las tres últimas comparten lo esencial: el esquema se cambia en un **paso explícito del despliegue**, con credenciales del despliegue, antes de que arranque la aplicación nueva. La primera es la que hay que entender bien, porque es la que sale en todos los tutoriales:

```csharp
using var scope = app.Services.CreateScope();
var context = scope.ServiceProvider.GetRequiredService<TiendaContext>();
await context.Database.MigrateAsync();
```

En desarrollo es cómoda y correcta: clonas el repositorio, pulsas F5 y tienes la base de datos al día sin acordarte de ningún comando. En producción tiene dos problemas que no se arreglan con cuidado.

**Uno: cuatro instancias arrancan a la vez.** Con réplicas, las cuatro ejecutan ese código casi en el mismo milisegundo, las cuatro leen `__EFMigrationsHistory`, las cuatro ven la misma migración pendiente y las cuatro intentan aplicarla. La que llega segunda se encuentra con esto:

```
Microsoft.Data.SqlClient.SqlException (0x80131904): There is already an object named
'Resenas' in the database.
```

Esa instancia muere en el arranque, el orquestador la reinicia, y acabas con un despliegue a medias: una migración registrada por una réplica y tres contenedores reiniciándose en bucle. Que el motor o la versión de EF Core lleguen a serializar el acceso con un bloqueo no arregla el fondo: el resto de instancias se queda esperando en el arranque, fallan los *health checks* y el despliegue se cae igual.

**Dos: la aplicación necesita permisos de DDL para siempre.** Para ejecutar `ALTER TABLE` y `DROP TABLE` al arrancar, el usuario de la cadena de conexión tiene que poder hacerlo **en todo momento**, no solo durante el minuto del despliegue. Una inyección SQL que en condiciones normales llegaría como mucho a leer datos, ahí llega a borrar tablas.

## El script idempotente

`--idempotent` es la diferencia entre un script que solo puedes ejecutar una vez y uno que puedes ejecutar tantas veces como quieras. Cada bloque va envuelto en una comprobación sobre `__EFMigrationsHistory`:

```sql
IF NOT EXISTS(SELECT * FROM [__EFMigrationsHistory] WHERE [MigrationId] = N'20260728091412_AnadirStockMinimoAProducto')
BEGIN
    ALTER TABLE [Productos] ADD [StockMinimo] int NOT NULL DEFAULT 0;
END;
GO
```

Cada migración aporta dos bloques con esa misma condición: uno con el cambio y otro que inserta su fila en `__EFMigrationsHistory`. El script empieza además creando esa tabla si no existe. Ejecutarlo dos veces no duplica nada; ejecutarlo contra una base de datos que va tres migraciones por detrás aplica exactamente las tres que faltan. Es el formato que quieres entregar a un DBA o pegar en un paso de pipeline, porque un reintento no es una catástrofe. Ojo: la guarda comprueba la migración completa, no cada sentencia, así que un `migrationBuilder.Sql()` con un `UPDATE` que reventó a medias en el intento anterior sigue siendo tu problema.

## Editar la migración generada

La migración es código tuyo: cuando EF Core no adivina bien, se corrige antes de aplicarla.

**El caso destructivo clásico: renombrar.** Cambias la propiedad `Nombre` a `NombreProducto` y generas la migración. Esto es lo que sale:

```csharp
// ❌ Lo que genera EF Core: pierde los datos
migrationBuilder.DropColumn(name: "Nombre", table: "Productos");
migrationBuilder.AddColumn<string>(
    name: "NombreProducto", table: "Productos",
    type: "nvarchar(max)", nullable: false, defaultValue: "");
```

EF Core no puede saber que has renombrado: solo ve que una propiedad desapareció y otra apareció. Y esto es lo que queda tras aplicarlo sobre un catálogo de 48 213 productos:

```
> SELECT COUNT(*) AS Vacios FROM Productos WHERE NombreProducto = '';
Vacios
-----------
48213
```

Los 48 213 nombres se han ido, y el `Down()` no los recupera: hace lo simétrico, borrar la columna nueva y crear la vieja vacía. La corrección es una línea:

```csharp
// ✅ Renombrado real: conserva los valores
protected override void Up(MigrationBuilder migrationBuilder)
    => migrationBuilder.RenameColumn(
        name: "Nombre", table: "Productos", newName: "NombreProducto");

protected override void Down(MigrationBuilder migrationBuilder)
    => migrationBuilder.RenameColumn(
        name: "NombreProducto", table: "Productos", newName: "Nombre");
```

Que los datos sobrevivan no significa que el despliegue sea seguro: entre que se renombra la columna y que se despliega el código nuevo, las instancias antiguas siguen pidiendo `Nombre`. Eso se resuelve con la secuencia *expand/contract* que explica [Estrategias zero-downtime](Estrategias-Zero-Downtime.md).

**Lo que la API no expresa: `Sql()`.** Vistas, funciones, índices filtrados, restricciones `CHECK` con expresiones del motor. Se escribe SQL crudo, y el `Down()` también:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
    => migrationBuilder.Sql(@"CREATE VIEW VentasPorProducto AS
        SELECT p.Id, p.Nombre, SUM(l.Cantidad) AS UnidadesVendidas
        FROM Productos p JOIN LineasPedido l ON l.ProductoId = p.Id
        GROUP BY p.Id, p.Nombre;");

protected override void Down(MigrationBuilder migrationBuilder)
    => migrationBuilder.Sql("DROP VIEW VentasPorProducto;");
```

**El orden importa y lo controlas tú.** Las llamadas de `Up()` se ejecutan tal como están escritas, y EF Core no siempre las ordena bien cuando el cambio es compuesto. Para añadir una columna obligatoria con un valor calculado, la secuencia correcta es: `AddColumn` como nullable, `Sql("UPDATE Productos SET ...")` para rellenarla y `AlterColumn` con `nullable: false`. Reordenar líneas a mano es una intervención legítima y frecuente.

## Datos con `HasData`

`HasData` sirve para datos que forman parte del esquema, no del negocio: catálogos cerrados, estados de pedido, países, roles.

```csharp
modelBuilder.Entity<EstadoPedido>().HasData(
    new EstadoPedido { Id = 1, Codigo = "pendiente", Descripcion = "Pendiente de pago" },
    new EstadoPedido { Id = 2, Codigo = "pagado",    Descripcion = "Pagado" },
    new EstadoPedido { Id = 3, Codigo = "enviado",   Descripcion = "Enviado" });
```

La siguiente migración incluye los `InsertData` correspondientes, y EF Core calcula el *diff* de los datos igual que el del esquema: si mañana cambias `"Pagado"` por `"Pago confirmado"`, genera un `UpdateData` de esa fila. Sus tres límites:

- **Necesita claves primarias explícitas y fijas.** Son la identidad con la que EF Core compara. Si las generas con `Guid.NewGuid()`, cada `migrations add` verá filas distintas y producirá un `DELETE` + `INSERT` interminable.
- **Nada calculado en tiempo de ejecución.** Un `DateTime.Now` en `HasData` se congela como literal en el SQL de la migración: la fecha del día en que se generó, para siempre.
- **Quitar una entrada genera `DeleteData`.** Si alguien editó esa fila en producción, se borra sin preguntar.

Y lo que **no** va aquí: un *backfill*. Rellenar `StockMinimo` en 48 213 productos con un valor calculado no son datos de referencia; es un proceso que bloquea la tabla mientras dura y con el que el despliegue se queda esperando. Sepáralo en un job aparte, por lotes, después de la migración.

## Dos ramas, dos migraciones

Situación real: tú generas `AnadirStockMinimoAProducto` y otra persona genera `AnadirTelefonoACliente` el mismo día. Al integrar, Git marca conflicto en `TiendaContextModelSnapshot.cs`. No lo resuelvas a mano: el snapshot es un fichero generado y solo puede describir **un** estado final, así que elegir líneas de los dos lados produce un modelo que no corresponde a ninguna migración. Deja que EF Core lo regenere:

```bash
dotnet ef database update CrearEsquemaInicial   # 1. deshazla en tu BD local si la aplicaste
dotnet ef migrations remove                     # 2. retírala y revierte el snapshot
git merge origin/master                         # 3. integra la rama ajena, ya sin conflicto
dotnet ef migrations add AnadirStockMinimoAProducto   # 4. regenérala sobre el modelo unido
dotnet ef database update                       # 5. aplica las dos
```

El resultado es el que quieres: la migración ajena queda primero, la tuya se genera **encima** de un snapshot que ya incluye el teléfono del cliente, y el orden de los *timestamps* coincide con el orden real de los cambios.

Si el conflicto llega cuando tu migración ya está en `master` y aplicada en un entorno compartido, esto no vale: ahí toca una migración nueva que compense la diferencia. Borrar historia que otros ya aplicaron rompe sus bases de datos.

## Varios `DbContext`, varios proyectos

La mitad de los errores del principio no son de migraciones: son de que `dotnet ef` no encuentra lo que busca. Necesita saber tres cosas y adivina las tres.

```bash
dotnet ef migrations add AnadirStockMinimoAProducto \
  --context TiendaContext --project src/Tienda.Datos --startup-project src/Tienda.Api
```

- `--context` — cuál de los `DbContext`. Sin él, si hay más de uno: `More than one DbContext was found. Specify which one to use.`
- `--project` — dónde vive el `DbContext` y dónde se escriben las migraciones (el *migrations assembly*). Por defecto, el del directorio actual.
- `--startup-project` — qué proyecto se ejecuta para construir el modelo: de ahí salen el `Program.cs`, la inyección de dependencias y la cadena de conexión de `appsettings.json`. Es el que se olvida, y sin él la herramienta no sabe cómo instanciar tu contexto.

Esa combinación es la habitual en una solución en capas: contexto en el proyecto de datos, arranque en el de API.

## Detectar el *drift*

*Drift* es que el esquema real y tus migraciones dejen de coincidir. Ocurre cuando alguien entra con un cliente SQL y añade un índice «rápido, que esto va lento», o cuando se restaura una copia de otro entorno. EF Core **no lo detecta**: no lee el esquema, lee `__EFMigrationsHistory` y el snapshot. Un índice creado a mano es invisible hasta que una migración intenta crear otro con el mismo nombre y falla, o hasta que un `Down()` intenta borrar algo que ya no está.

Lo que sí detecta es la otra mitad del problema: que tu **modelo** haya avanzado sin generar migración. Desde EF Core 9 eso rompe el arranque en vez de pasar desapercibido:

```
The model for context 'TiendaContext' has pending changes. Add a new migration before
updating the database. This exception can be suppressed or logged by passing event ID
'RelationalEventId.PendingModelChangesWarning' to the 'ConfigureWarnings' method.
```

Es una buena noticia disfrazada de error: antes, ese desajuste se descubría en producción con una columna que faltaba. La respuesta correcta es `dotnet ef migrations add`, no silenciar el aviso. En CI conviene adelantarse:

```bash
dotnet ef migrations has-pending-model-changes
# Changes have been made to the model since the last migration. Add a new migration.
```

Devuelve código de salida distinto de cero, así que sirve tal cual como paso que hace fallar el *build*. Para el drift del esquema real la única defensa es de proceso: nadie toca producción con un cliente SQL, y los permisos de DDL los tiene el despliegue, no las personas.

## Errores frecuentes

| Síntoma (mensaje literal) | Causa |
|---|---|
| `Unable to create an object of type 'TiendaContext'. For the different patterns supported at design time...` | La herramienta no consigue instanciar el contexto: falta `--startup-project`, el `Program.cs` no llega a construir el host, o el constructor exige parámetros que no sabe rellenar. Se resuelve con un `IDesignTimeDbContextFactory<TiendaContext>` |
| `The model for context 'TiendaContext' has pending changes` | Cambiaste las clases y no generaste migración: `dotnet ef migrations add` |
| `There is already an object named 'Productos' in the database` | El objeto se creó fuera de las migraciones, o dos instancias aplicaron la misma migración a la vez con `MigrateAsync()` al arrancar |
| `No project was found. Change the current working directory or use the --project option.` | Estás en la carpeta de la solución, no en la de un `.csproj`. Añade `--project` |
| `Your target project 'Tienda.Api' doesn't match your migrations assembly 'Tienda.Datos'` | Las migraciones viven en otro proyecto del que `dotnet ef` cree. Ajusta `--project`, o declara el ensamblado con `MigrationsAssembly("Tienda.Datos")` |
| `The migration '20260715083010_AnadirDescuento' was not found.` al volver atrás | Esa migración está registrada en `__EFMigrationsHistory` pero ya no existe en el código: alguien la borró después de aplicarla. La base de datos tiene un cambio del que nadie guarda el `Down()` |
| Columna vacía después de un renombrado | `DropColumn` + `AddColumn` en vez de `RenameColumn`. Los datos no vuelven; recupéralos de la copia de seguridad |
| `SqlException: Execution Timeout Expired. The timeout period elapsed prior to completion of the operation` | La sentencia tarda más que el `CommandTimeout` (30 s por defecto). Típico al añadir un índice o una columna `NOT NULL` con default sobre una tabla grande |

Ese último merece una nota: el `Timeout expired` a mitad de un despliegue es peor de lo que parece, porque el `ALTER TABLE` **puede seguir corriendo en el servidor** después de que el cliente se rinda. No lo relances a ciegas.

## Cuándo NO usar EF Core Migrations

- **El esquema es propiedad de otro equipo o de un DBA.** Si los cambios se piden por ticket y se revisan como SQL, una herramienta que lo genera sola estorba: escribe los scripts a mano con [Flyway](Flyway.md) o [DbUp](DbUp.md) y deja que EF Core solo lea.
- **La base de datos la comparten varios servicios.** El snapshot de un `TiendaContext` describe lo que *ese* contexto conoce. Si otro servicio tiene sus propias tablas ahí, cada `migrations add` puede proponer borrar lo que no reconoce.
- **Necesitas control fino del SQL.** Índices `CONCURRENTLY`, particionado, `lock_timeout` antes de cada DDL, orden exacto de operaciones. Se puede meter todo en `migrationBuilder.Sql()`, pero entonces estás escribiendo SQL a mano dentro de una herramienta que existe para no escribirlo.
- **El proyecto no usa EF Core como ORM.** Con Dapper o ADO.NET, arrastrar `Microsoft.EntityFrameworkCore.Design` y un `DbContext` fantasma solo para tener migraciones es un peaje que no compensa.

## Buenas prácticas avanzadas

- **Genera el script con `migrations script --idempotent` antes de aplicar en producción, y léelo.** El nombre de la migración no te dice nada; el SQL sí. Es lo único que destapa a tiempo un `DROP COLUMN` que no pediste, un `AddColumn` con un default que reescribe la tabla entera o un índice que va a bloquear `Productos` durante minutos. Guarda ese `.sql` como artefacto del despliegue: es tu registro de qué se ejecutó exactamente.
- **Una migración, un propósito.** Empaquetar la sprint completa en `CambiosVarios` hace imposible revertir un cambio sin arrastrar los otros cuatro, y convierte el `Down()` en una ruleta. Varias migraciones el mismo día no son un problema; una migración con siete cosas dentro sí.
- **Nunca mezcles esquema y datos masivos en la misma migración.** Un `Sql("UPDATE Productos SET ...")` sobre cientos de miles de filas dentro de un `Up()` bloquea la tabla y el despliegue queda esperando, con el *timeout* del pipeline corriendo. El esquema en la migración; el *backfill* como proceso aparte, por lotes y reanudable.
- **Revisa el snapshot en las revisiones de código, y no lo resuelvas a mano en un merge.** `TiendaContextModelSnapshot.cs` es el «de dónde venimos» y compila igual de bien estando mal, así que ningún test lo delata: el síntoma llega días después, en la migración *siguiente*, que sale incorrecta. Ante un conflicto, `migrations remove` y regenerar; nunca elegir líneas.
- **Que la aplicación en producción no tenga permisos de DDL.** Es la consecuencia práctica de renunciar a `MigrateAsync()` al arrancar: el usuario del despliegue puede hacer `ALTER TABLE` durante el minuto que dura, y el de la aplicación solo `SELECT`/`INSERT`/`UPDATE`/`DELETE`, siempre. Además de cerrar la puerta a que una inyección SQL borre tablas, hace imposible el peor incidente de esta ficha: cuatro réplicas migrando a la vez al arrancar.
- **Lee siempre el `Down()`, aunque no pienses usarlo.** Un `Down()` que EF Core generó y nadie miró es una vuelta atrás que falla el día que la necesitas, normalmente a las tres de la mañana. Si un cambio no se puede deshacer —borrar una columna con datos—, es mejor que el `Down()` lance una excepción explicando el motivo que aparente funcionar.

## Recursos didácticos

- [Managing Database Schemas en la documentación de EF Core](https://learn.microsoft.com/ef-core/managing-schemas/migrations/) — la referencia completa, con páginas propias para *bundles* y personalización del historial. La sección «Migrations in Team Environments» es corta y contesta justo las dudas del snapshot.
- [EF Core Power Tools](https://github.com/ErikEJ/EFCorePowerTools) — extensión de Visual Studio que dibuja el modelo y compara el esquema real con el modelo de EF Core, que es la única forma cómoda de ver *drift*.
- [Breaking changes de EF Core 9](https://learn.microsoft.com/ef-core/what-is-new/ef-core-9.0/breaking-changes) — explica el `PendingModelChangesWarning` y por qué se decidió que fallara en vez de avisar. Vale la pena leerlo antes de actualizar un proyecto con migraciones antiguas.

---

*En resumen: EF Core Migrations convierte los cambios de tus clases en SQL versionado sin esfuerzo — pero el snapshot es un fichero delicado, un renombrado mal generado borra datos, y en producción las migraciones se aplican como paso del despliegue, nunca al arrancar la aplicación.*
