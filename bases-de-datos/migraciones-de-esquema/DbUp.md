# DbUp

## ¿Qué es?

DbUp es una librería .NET que aplica migraciones escritas en SQL plano. No es un ejecutable ni un plugin: es un paquete NuGet que llamas desde tu propio código C#, normalmente desde una pequeña aplicación de consola que se ejecuta como paso del despliegue.

## ¿Por qué existe?

Herramientas como [Flyway](Flyway.md) funcionan muy bien como CLI externa, pero eso obliga a que el binario exista y esté configurado allí donde se despliega: en el agente del pipeline, en la imagen de Docker, en el servidor. Algunos equipos .NET prefieren que aplicar las migraciones sea **código C# normal**: una consola que llama a una librería, se compila con el resto de la solución y se distribuye como cualquier otro artefacto. DbUp cubre ese hueco: es un paquete NuGet más, y tú decides desde dónde y cuándo se ejecuta.

> Si Flyway es una herramienta a la que das órdenes desde fuera, DbUp es un motor que atornillas dentro de tu propio programa. Nadie instala nada en el destino: el ensamblado lleva el motor y los scripts consigo.

Lo que DbUp **no** intenta ser es un generador. No sabe nada de tus clases C# ni compara tu modelo de objetos con el esquema: cada script lo escribes a mano, igual que en Flyway. Si buscas que alguien redacte el `ALTER TABLE` por ti a partir de un modelo, eso es [EF Core Migrations](EF-Core-Migrations.md).

## ¿Cuándo y para qué se usa?

En proyectos .NET que quieren migraciones en SQL plano —porque así el SQL que se ejecutará contra producción se revisa en el *pull request* como cualquier otro código— pero prefieren no depender de una herramienta externa en el destino. Si el concepto de migración, la tabla de control o la diferencia entre imperativo y declarativo te suenan a chino, empieza por [Migraciones de esquema](Migraciones-de-Esquema.md): aquí se da por sabido.

El ejemplo conductor es una tienda online con la base de datos `TiendaDB` y las tablas `Productos`, `Pedidos` y `Clientes`. El proyecto de despliegue, `Tienda.Migraciones`, es una consola. Iremos siguiendo un pedido concreto, el **#4711**.

## Puesta en marcha: la consola de despliegue

Hay un paquete por motor de base de datos. Para SQL Server se instala con `dotnet add package dbup-sqlserver`; los equivalentes son `dbup-postgresql`, `dbup-mysql`, `dbup-sqlite` y `dbup-oracle`. Cambia el paquete y cambia la primera llamada (`SqlDatabase`, `PostgresqlDatabase`, `MySqlDatabase`); el resto de la configuración es idéntico. La consola entera cabe en una pantalla y es la pieza central de DbUp:

```csharp
using System.Reflection;
using DbUp;

var connectionString = Environment.GetEnvironmentVariable("TIENDA_CONNECTION_STRING")!;
EnsureDatabase.For.SqlDatabase(connectionString);

var resultado = DeployChanges.To
    .SqlDatabase(connectionString)
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly())
    .WithTransactionPerScript()
    .LogToConsole()
    .Build()
    .PerformUpgrade();

if (!resultado.Successful)
{
    Console.WriteLine(resultado.Error);   // la excepción completa, no solo .Message
    return 1;
}

Console.WriteLine($"Aplicados {resultado.Scripts.Count()} scripts.");
return 0;
```

`EnsureDatabase.For.SqlDatabase` crea `TiendaDB` si no existe; se conecta a `master`, así que necesita permisos para ello y se borra allí donde la base de datos la cree el equipo de sistemas. `PerformUpgrade()` devuelve un `DatabaseUpgradeResult` con tres cosas: `Successful`, `Error` (la excepción, `null` si todo fue bien) y `Scripts`, que son los aplicados **en esta ejecución**, no todo el historial. Y lo más importante: **el `return` es lo único que mira el pipeline**, porque el valor que devuelve `Main` es el código de salida del proceso y cualquier orquestador interpreta 0 como éxito y cualquier otro valor como fallo. Sin ese `return 1`, un despliegue roto sale en verde.

Con dos scripts pendientes y la base de datos recién creada, `LogToConsole()` imprime:

```
Beginning database upgrade
Checking whether journal table exists..
Journal table does not exist
Executing Database Server script 'Tienda.Migraciones.Scripts.Script0001_CrearTablaProductos.sql'
Checking whether journal table exists..
Creating the [dbo].[SchemaVersions] table
The [dbo].[SchemaVersions] table has been created
Executing Database Server script 'Tienda.Migraciones.Scripts.Script0002_CrearTablaPedidos.sql'
Upgrade successful
Aplicados 2 scripts.
```

Fíjate en el orden real: DbUp **ejecuta el primer script antes de crear la tabla de historial**, y la crea justo antes de registrar ese primer resultado. En la primerísima ejecución, el script 1 y la creación del journal no son la misma operación atómica.

## El *journal*: la tabla `SchemaVersions`

DbUp lleva la cuenta de lo aplicado en `dbo.SchemaVersions`, que crea él solo la primera vez. Tres columnas y ninguna sorpresa: `Id` (`int IDENTITY`, sin más significado), `ScriptName` (`nvarchar(255)`, el nombre completo tal como DbUp lo ve) y `Applied` (`datetime`, en hora del servidor). Consultarla con `SELECT ScriptName, Applied FROM dbo.SchemaVersions ORDER BY Id DESC;` es la forma más rápida de saber en qué punto está un entorno:

```
ScriptName                                                       Applied
---------------------------------------------------------------- -----------------------
Tienda.Migraciones.Scripts.Script0002_CrearTablaPedidos.sql      2026-07-28 09:14:33.117
Tienda.Migraciones.Scripts.Script0001_CrearTablaProductos.sql    2026-07-28 09:14:33.043
```

`ScriptName` incluye el *namespace* porque, con recursos embebidos, el nombre del recurso es `<ensamblado>.<carpetas>.<fichero>`. Consecuencia práctica: **si renombras el proyecto o mueves la carpeta `Scripts`, cambian todos los nombres** y DbUp cree que ninguno se ha aplicado.

El nombre y el esquema de la tabla se cambian con `.JournalToSqlTable("migraciones", "HistorialEsquema")`, útil si `dbo` está reservado o si conviven varias aplicaciones en la misma base de datos. Hazlo **antes** de la primera ejecución: cambiarlo con 47 scripts ya aplicados deja el historial anterior huérfano, así que DbUp crea la tabla nueva vacía, concluye que hay 47 pendientes, arranca por `Script0001_CrearTablaProductos.sql` y revienta con `There is already an object named 'Productos' in the database.` Con suerte falla en el primero; si tus scripts usan comprobaciones defensivas de existencia, no falla, y entonces reaplica los 47 contra una base de datos que ya los tenía, con los `UPDATE` de datos ejecutándose dos veces.

## De dónde salen los scripts

Hay dos formas de indicarle a DbUp dónde están los `.sql`: `WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly())`, que los busca dentro del propio `.dll`, y `WithScriptsFromFileSystem("Scripts")`, que los lee del disco al ejecutar.

| | `WithScriptsEmbeddedInAssembly` | `WithScriptsFromFileSystem` |
|---|---|---|
| ¿El ensamblado viaja solo? | ✅ Sí, los scripts van dentro del `.dll` | ❌ No, hay que copiar la carpeta también |
| ¿Se pueden editar en el servidor? | ❌ No, hay que recompilar | ⚠️ Sí — y eso es el problema |
| Si falta un fichero | Imposible: o está compilado o no existe | Se ignora en silencio, sin error |
| Si falta la carpeta entera | No aplica | `DirectoryNotFoundException` al construir |
| Nombre en el journal | `Tienda.Migraciones.Scripts.Script0001_….sql` | `Script0001_….sql` |

La primera es la recomendable para cualquier entorno que no sea tu portátil, y el argumento decisivo no es la comodidad: **un artefacto autocontenido es reproducible**. Con la segunda, el mismo `.dll` desplegado dos veces puede aplicar cosas distintas según lo que hubiera en la carpeta del servidor. Y que el nombre registrado sea distinto entre ambas importa mucho: cambiar de una a otra a mitad de la vida del proyecto reproduce exactamente el problema del `JournalToSqlTable`.

## El orden es alfabético, no numérico

La trampa número uno de DbUp, y no avisa de ninguna forma. DbUp ordena los scripts comparando sus nombres **como cadenas de texto**, carácter a carácter: no interpreta el número del prefijo. Con esta nomenclatura, que parece perfectamente razonable, el orden de ejecución real es el de la derecha:

```
❌ Script1_CrearTablaProductos.sql            1º  Script10_AnadirIndiceEstadoPedido.sql
   Script2_CrearTablaPedidos.sql              2º  Script1_CrearTablaProductos.sql
   Script10_AnadirIndiceEstadoPedido.sql      3º  Script2_CrearTablaPedidos.sql
```

```
Executing Database Server script '...Script10_AnadirIndiceEstadoPedido.sql'
Invalid object name 'Pedidos'.
```

`Script10` va primero porque, comparando cadenas, `"1"` seguido de `"0"` es menor que `"1"` seguido de `"_"`. El índice sobre `Pedidos` se intenta crear antes de que la tabla exista. La solución es rellenar con ceros a la izquierda, con margen para toda la vida del proyecto:

```
✅ Script0001_CrearTablaProductos.sql
   Script0002_CrearTablaPedidos.sql
   Script0010_AnadirIndiceEstadoPedido.sql
```

Ahora la comparación alfabética coincide con la numérica. Cuatro dígitos dan margen para 9 999 migraciones; con tres te quedas corto antes de lo que parece. Otra convención igual de válida es la fecha ordenable, `20260728_1430_AnadirIndiceEstadoPedido.sql`, que además evita colisiones cuando dos ramas añaden migraciones el mismo día. Y no, el orden no lo salva el journal: DbUp no guarda dependencias entre scripts, solo qué nombres ya ha visto.

## El recurso embebido que nadie marcó

El fallo silencioso perfecto. `WithScriptsEmbeddedInAssembly` no lee ficheros del disco: lee **recursos embebidos** del ensamblado. Si los `.sql` no están marcados como *Embedded Resource*, no están en el `.dll`. DbUp no encuentra nada, no falla, y **devuelve éxito**:

```
Beginning database upgrade
Checking whether journal table exists..
Fetching list of already executed scripts.
No new scripts need to be executed - completing.
Upgrade successful
```

Código de salida 0, pipeline en verde, base de datos intacta y la aplicación nueva arrancando contra un esquema viejo. Desde fuera es indistinguible de "no había nada pendiente". Se arregla en el `.csproj`:

```xml
<ItemGroup>
  <None Remove="Scripts\**\*.sql" />
  <EmbeddedResource Include="Scripts\**\*.sql" />
</ItemGroup>
```

El `None Remove` es necesario porque los proyectos SDK ya incluyen esos ficheros como `None`, y sin quitarlos primero la compilación se queja de items duplicados. El comodín `**` recoge las subcarpetas, así que no hay que tocar el `.csproj` cada vez que se añade un script — que es justo cómo se cuela el problema cuando cada fichero se lista a mano. La comprobación que cierra el asunto, y que vale la pena dejar puesta detrás de un argumento `--listar`:

```csharp
foreach (var n in Assembly.GetExecutingAssembly().GetManifestResourceNames())
    Console.WriteLine(n);   // Tienda.Migraciones.Scripts.Script0001_CrearTablaProductos.sql
```

Si esa lista sale vacía, no hay nada más que investigar.

## Transacciones: el valor por defecto es el menos seguro

Hay tres modos —`WithoutTransaction()` (el que se usa si no dices nada), `WithTransactionPerScript()` y `WithTransaction()`— y el valor por defecto es el que menos protege. Supongamos cinco scripts pendientes y que el tercero falla a mitad, tras haber ejecutado ya su primera sentencia:

| Modo | Scripts 1 y 2 | Script 3 (el que falla) | Scripts 4 y 5 | En `SchemaVersions` |
|---|---|---|---|---|
| `WithoutTransaction` | Aplicados | ⚠️ **A medias**: lo ejecutado antes del error persiste | Sin ejecutar | 1 y 2 |
| `WithTransactionPerScript` | Aplicados | Revertido por completo | Sin ejecutar | 1 y 2 |
| `WithTransaction` | Revertidos | Revertido | Sin ejecutar | Nada: la tabla queda como estaba |

La fila del medio es la interesante. Con `WithoutTransaction`, un script que crea una tabla y luego migra datos a ella puede dejar la tabla creada y vacía; el journal no lo registra, así que la siguiente ejecución lo reintenta desde el principio y falla con `There is already an object named …`. El despliegue queda atascado y hay que arreglarlo a mano en producción.

`WithTransactionPerScript` es el valor razonable: cada script es atómico y, si algo falla, sabes exactamente dónde estás. Solo exige que cada script sea internamente coherente, que es como deberían escribirse de todos modos. `WithTransaction` da la garantía más fuerte —todo o nada— a cambio de dos costes reales: mantiene los bloqueos de todos los scripts abiertos hasta el final, lo que en una migración larga sobre `Pedidos` puede bloquear la aplicación durante minutos, y una reversión deshace también las entradas del journal, con lo que un fallo en el script 5 obliga a reejecutar los cuatro primeros.

Y el aviso imprescindible en SQL Server: **hay sentencias que no admiten una transacción explícita**. `CREATE DATABASE`, `ALTER DATABASE`, `BACKUP`, `CREATE FULLTEXT INDEX` y `RECONFIGURE` fallan con `CREATE DATABASE statement not allowed within multi-statement transaction.` si las envuelves. Si necesitas una de ellas, va en su propio script, y ese script se ejecuta en un `UpgradeEngine` aparte configurado con `WithoutTransaction()`. Mezclarlo todo en un único motor con `WithTransaction()` no tiene arreglo.

## Filtrar, agrupar y reejecutar scripts

Por defecto cada script se aplica una vez y su nombre queda en el journal para siempre. Para vistas, procedimientos y funciones eso es incómodo: cambiar un procedimiento obliga a crear un script nuevo, y el repositorio pierde de vista cuál es la definición vigente. El segundo parámetro de `WithScriptsEmbeddedInAssembly` es un filtro sobre el nombre del recurso, y el tercero permite marcar todo un grupo para reejecutarse en cada despliegue, igual que las migraciones repetibles `R__` de Flyway:

```csharp
using DbUp.Support;
var ensamblado = Assembly.GetExecutingAssembly();
var entorno = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Production";

// Scripts/Versionados/ y Scripts/Entornos/<entorno>/ — una sola vez, en orden
.WithScriptsEmbeddedInAssembly(ensamblado,
    n => n.Contains(".Versionados.") || n.Contains($".Entornos.{entorno}."))
// Scripts/Repetibles/ — en cada despliegue
.WithScriptsEmbeddedInAssembly(ensamblado, n => n.Contains(".Repetibles."),
    new SqlScriptOptions { ScriptType = ScriptType.RunAlways })
```

El orden de las llamadas fija el orden de los grupos, así que los repetibles van después de los versionados: cuando una vista se recrea, la columna que necesita ya existe. Los `RunAlways` sí se registran en `SchemaVersions`, pero DbUp no los excluye por estar ahí, así que la tabla acumula una fila por despliegue. Un `RunAlways` solo es seguro si el objeto se puede recrear sin pérdida:

- ✅ Vistas, procedimientos, funciones, sinónimos, *triggers*. `CREATE OR ALTER VIEW` lo hace idempotente en una sola sentencia.
- ✅ Datos de referencia con un `MERGE` idempotente: los estados posibles de un pedido, las provincias.
- ❌ Tablas. "Recrear" `Pedidos` significa borrar el pedido #4711 y todos sus hermanos.
- ❌ Cualquier script con un `INSERT` sin protección: se ejecuta en cada despliegue y duplica filas.

El filtro por entorno del ejemplo hace que una carpeta `Scripts/Entornos/Development/` con datos de prueba —el pedido #4711 sembrado a mano— nunca llegue a producción, porque allí el filtro no la selecciona. Cuidado con la simetría: un script que solo se aplica en algunos entornos hace que los journals divergan, y eso es aceptable para datos de prueba, nunca para estructura. Cuando hace falta control sobre las **fases**, `SqlScriptOptions` tiene además `RunGroupOrder`: DbUp agrupa por ese número, lo ordena y dentro de cada grupo aplica el orden alfabético habitual (`0` estructura, `100` vistas y procedimientos, `200` permisos). Y para casos exóticos existe `.WithFilter(IScriptFilter)`, que recibe la lista ya ordenada junto con los nombres ya ejecutados y devuelve la lista final: una puerta de escape potente y peligrosa, porque la lógica que pongas ahí decide qué se ejecuta contra producción. Que sea trivial de leer.

## Scripts en C# con `IScript`

A veces el SQL plano no llega: una transformación que necesita lógica de verdad, o un script que debe leer un fichero o consultar la propia base de datos para decidir qué generar. DbUp permite que un "script" sea una clase C# que **devuelve** el SQL a ejecutar.

```csharp
public class Script0011_NormalizarTelefonosClientes : IScript
{
    public string ProvideScript(Func<IDbCommand> crearComando)
    {
        using var comando = crearComando();
        comando.CommandText = "SELECT Id, Telefono FROM Clientes WHERE Telefono IS NOT NULL";
        var sentencias = new List<string>();
        using var lector = comando.ExecuteReader();
        while (lector.Read())
        {
            var digitos = new string(lector.GetString(1).Where(char.IsDigit).ToArray());
            sentencias.Add($"UPDATE Clientes SET Telefono='{digitos}' WHERE Id={lector.GetInt32(0)};");
        }
        return string.Join(Environment.NewLine, sentencias);
    }
}
```

El comando que recibe ya está conectado y, si hay transacción, alistado en ella. Para que DbUp descubra estas clases hay que usar el otro método de registro, `WithScriptsAndCodeEmbeddedInAssembly(ensamblado)`, que recoge los `.sql` embebidos **y** las implementaciones de `IScript`. En el journal aparece el nombre completo del tipo, `Tienda.Migraciones.Scripts.Script0011_NormalizarTelefonosClientes`, y participa en la misma ordenación alfabética que los `.sql`, así que el prefijo con ceros también aplica aquí.

El aviso importante: **aquí desaparece la ventaja principal de DbUp**, que es poder leer en el *pull request* el SQL exacto que se va a ejecutar. Ahora se construye en tiempo de ejecución y el revisor tiene que razonar sobre un programa. Úsalo cuando de verdad no haya alternativa en SQL, y casi siempre la hay.

## Variables: el mismo SQL para varios entornos

`WithVariables` sustituye tokens con la forma `$nombre$` dentro del SQL antes de ejecutarlo. Sirve para lo que cambia entre entornos sin cambiar la estructura: un nombre de esquema, un usuario, una ruta de *filegroup*.

```csharp
.WithVariables(new Dictionary<string, string>
{
    ["nombreEsquema"] = Environment.GetEnvironmentVariable("TIENDA_ESQUEMA") ?? "ventas",
    ["usuarioLectura"] = "tienda_informes"
})
```

```sql
-- Script0012_PermisosLecturaPedidos.sql
GRANT SELECT ON $nombreEsquema$.Pedidos TO [$usuarioLectura$];
```

Con esos valores, DbUp ejecuta literalmente `GRANT SELECT ON ventas.Pedidos TO [tienda_informes];`. Dos cosas que muerden. Si un script contiene un `$token$` sin valor definido, DbUp aborta con un error del tipo `Variable nombreEsquema has no value defined` — lo cual es bueno: falla en lugar de ejecutar SQL a medio construir. Y la sustitución se aplica a **todo** el texto, comentarios y literales de cadena incluidos, así que un script que inserte `'$99.90 USD$'` puede acabar destrozado; para esos casos se desactiva con `.WithVariablesDisabled()` en un motor aparte. La regla que conviene decir en voz alta: las variables son para nombres y configuración, nunca para credenciales. La cadena de conexión llega por variable de entorno o por el gestor de secretos, jamás dentro de un script versionado.

## Cómo encaja en el despliegue

Hay dos sitios donde llamar a DbUp, y solo uno es buena idea en general. **Como paso del pipeline**, la consola es un artefacto más, se ejecuta antes de desplegar la aplicación y su código de salida decide si el despliegue continúa:

```yaml
- script: dotnet Tienda.Migraciones.dll
  displayName: Aplicar migraciones a TiendaDB
  env: { TIENDA_CONNECTION_STRING: $(CadenaConexionMigraciones) }
```

Un `return 1` corta el pipeline y la versión nueva de la API no llega a arrancar contra un esquema viejo. Es un único proceso, con las credenciales de despliegue —que sí tienen permisos de DDL— separadas de las que usa la aplicación en marcha. Un detalle que se paga en tiempo perdido: en Windows un `return -1` se ve como `-1`, pero en un agente Linux el código de salida se trunca a 8 bits y `-1` se convierte en **255**. Sigue siendo distinto de cero, así que el pipeline falla igual; el problema aparece si alguien compara contra un valor concreto. Devuelve `1`.

**Al arrancar la aplicación** es tentador, porque no hay que orquestar nada. El problema es la concurrencia: si el servicio corre con tres réplicas y un despliegue las arranca a la vez, las tres ejecutan `PerformUpgrade` simultáneamente, y DbUp **no toma ningún bloqueo distribuido**. Las tres consultan `SchemaVersions`, las tres ven el mismo script pendiente y las tres lo ejecutan. Lo mejor que puede pasar es que dos fallen con `There is already an object named 'Pedidos' in the database.` y sus contenedores entren en bucle de reinicio; lo peor es un script de datos aplicado tres veces sin que nada proteste. Si aun así tiene que ser al arranque, el mínimo imprescindible es serializar con un bloqueo de aplicación de SQL Server —`sp_getapplock` con ámbito de sesión antes de `PerformUpgrade` y `sp_releaseapplock` después— y aceptar que la aplicación necesita permisos de DDL en producción de forma permanente, justo lo que el paso de pipeline evita.

## DbUp no valida checksums

Esta es la diferencia funcional más importante frente a Flyway y hay que decirla sin rodeos: **DbUp solo compara nombres.** No guarda ningún *hash* del contenido y no comprueba nada. Si alguien edita `Script0002_CrearTablaPedidos.sql` después de que se haya aplicado en producción, DbUp ve ese nombre en `SchemaVersions`, concluye que ya está hecho y lo salta, sin aviso ni registro. La base de datos contiene una cosa y el repositorio describe otra, y nadie se enterará hasta que alguien monte un entorno nuevo desde cero: ahí sí se aplica la versión editada, y el esquema del entorno nuevo **no coincide** con el de producción. El fallo se manifiesta meses después, en el sitio equivocado, con la trazabilidad perdida.

Flyway lo detecta porque guarda el checksum y lo compara. DbUp no tiene equivalente, así que la compensación es de disciplina: **los scripts son inmutables por convención** —uno aplicado en cualquier entorno compartido no se toca nunca, se corrige con un script nuevo— y **esa regla se vigila en la revisión de código**, porque un *pull request* que **modifica** un `.sql` existente en lugar de **añadir** uno nuevo es una señal visible en el diff y basta con acordar que se rechaza. El aviso automático, además, es fácil de montar: un paso del pipeline que ejecute `git diff --name-status origin/master -- Scripts/` y falle si aparece una `M` en lugar de una `A` son diez líneas que replican el 80 % del valor de los checksums. Y comparar los esquemas de dos entornos (con SQL Server Data Tools, `mssql-scripter` o similar) es la red que detecta la divergencia si todo lo anterior falla.

## DbUp, Flyway y EF Core Migrations

| | DbUp | [Flyway](Flyway.md) | [EF Core Migrations](EF-Core-Migrations.md) |
|---|---|---|---|
| Quién escribe el SQL | Tú, a mano | Tú, a mano | Lo genera desde el modelo C# |
| Validación de checksums | ❌ No | ✅ Sí | ⚠️ Compara el *snapshot* del modelo, no el SQL |
| Dependencias en el destino | Ninguna: va en el `.dll` | El binario de Flyway o un contenedor | El SDK, o un *bundle* autocontenido |
| Portabilidad entre lenguajes | ❌ Solo .NET | ✅ Cualquiera | ❌ Solo .NET |
| Cómo se invoca | Código C# tuyo | CLI, Maven, Gradle, Docker | `dotnet ef` o `Migrate()` |
| Curva de entrada | Muy baja: una consola | Baja, pero hay que instalarlo | Media: hay que entender el modelo |
| Bloqueo entre instancias | ❌ No lo hace | ✅ Sí | ⚠️ Depende del proveedor |

La decisión corta: si el equipo ya usa EF Core y nadie pide SQL revisable, quédate ahí. Si el esquema lo comparten servicios en varios lenguajes, Flyway. Si es .NET, quieres el SQL a mano y te molesta depender de un binario externo, DbUp.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `No new scripts need to be executed` y de verdad no se aplicó nada | Los `.sql` no están marcados como `EmbeddedResource`; compruébalo con `GetManifestResourceNames()` |
| Los scripts se aplican en un orden que no es el del número | Comparación alfabética: `Script10_` va antes que `Script2_`; rellena con ceros |
| `Invalid object name 'Pedidos'` en un script de índice | Consecuencia de lo anterior: el índice se ejecutó antes que la tabla |
| Editaste un script ya aplicado y no se reaplica | DbUp solo compara nombres; hace falta un script nuevo |
| El esquema de un entorno recién creado no coincide con producción | Alguien editó en el pasado un script que ya estaba aplicado |
| El despliegue quedó a medias y ahora falla siempre con `There is already an object named …` | Script parcialmente aplicado sin transacción; el journal no lo registró |
| `CREATE DATABASE statement not allowed within multi-statement transaction.` | Sentencia incompatible con `WithTransaction`; sácala a un motor `WithoutTransaction` |
| `CREATE SCHEMA must be the first statement in a query batch.` | Falta un `GO` antes del `CREATE SCHEMA` dentro del script |
| El pipeline pasa en verde con la base de datos sin tocar | Falta el `return 1` cuando `Successful` es `false`, o no se registra `resultado.Error` |
| DbUp cree que ningún script está aplicado, en un entorno que sí los tenía | Cambió el nombre del ensamblado, la carpeta, el journal, o se pasó de recurso embebido a fichero |
| `Login failed for user 'tienda_deploy'.` | Cadena de conexión del entorno equivocado, o el usuario existe en el servidor pero no en `TiendaDB` |
| Los contenedores reinician en bucle tras un despliegue | Varias réplicas ejecutando `PerformUpgrade` a la vez; muévelo al pipeline |
| `Timeout expired` en una migración grande | El `Command Timeout` por defecto es 30 s; súbelo en la cadena de conexión del despliegue |

## Cuándo NO usar DbUp

- **El proyecto ya usa EF Core y no hay una razón concreta para otra herramienta.** Añadir DbUp significa dos fuentes de verdad del esquema y dos historiales que pueden divergir. Quédate en [EF Core Migrations](EF-Core-Migrations.md) hasta que aparezca un motivo nombrable.
- **Varios servicios en distintos lenguajes comparten el mismo esquema.** DbUp obliga a que el paso de migración sea .NET; [Flyway](Flyway.md) es agnóstico y cualquiera de los servicios puede ejecutarlo.
- **Necesitas validación de checksums de serie.** Si el equipo es grande o rota mucho, la disciplina no basta.
- **Buscas *zero-downtime* como característica.** Ninguna herramienta te lo da: se consigue con el diseño de los propios cambios, y eso está en [Estrategias zero-downtime](Estrategias-Zero-Downtime.md).

## Buenas prácticas avanzadas

- **Pon `WithTransactionPerScript()` explícitamente, siempre.** El valor por defecto es `WithoutTransaction` y es el peor de los tres: un script que falla a mitad deja el esquema en un estado que el journal no describe, y el despliegue se atasca hasta que alguien lo arregla a mano en producción. Escribirlo aunque coincidiera con el defecto también documenta la decisión para quien lea la consola en dos años.
- **Verifica que los scripts están embebidos como parte de la compilación, no confiando en el `.csproj`.** Una prueba de tres líneas que afirme `GetManifestResourceNames().Count(n => n.EndsWith(".sql")) > 0` convierte el fallo más peligroso de DbUp en un test rojo. Es la única forma de distinguir "no había nada pendiente" de "no encontré nada".
- **Usa dos usuarios de base de datos: uno para migrar y otro para la aplicación.** El de despliegue tiene `db_ddladmin` y solo lo conoce el pipeline; el de la aplicación tiene `SELECT`/`INSERT`/`UPDATE` y ningún permiso de DDL. Así una inyección SQL en la API no puede alterar el esquema y, de paso, migrar al arrancar deja de ser técnicamente posible: la restricción impone la buena práctica.
- **Sube el `Command Timeout` en la cadena de conexión del despliegue, no el `Connect Timeout`.** El primero limita cuánto puede tardar cada sentencia y viene en 30 segundos; un `ALTER TABLE` sobre `Pedidos` con millones de filas se lo come. La confusión entre ambos explica la mitad de los `Timeout expired` en migraciones, porque subir el de conexión no cambia nada.
- **Registra `resultado.Error` completo antes de devolver el código de error.** Su `ToString()` incluye el número de error de SQL Server y el nombre del script, que es todo lo que necesitas; imprimir solo `.Message` te deja con un mensaje genérico y un despliegue roto a las tres de la mañana. Y no captures la excepción para seguir adelante: si la migración falló, el despliegue debe fallar.
- **Decide la convención de numeración el primer día.** Cambiarla a mitad no rompe nada retroactivamente —lo aplicado ya está en el journal— pero mezcla dos esquemas de ordenación en la misma carpeta, y a partir de ahí el orden real solo se sabe listando los nombres. Con ramas paralelas, el prefijo de fecha y hora evita además que dos personas elijan el mismo número.

## Recursos didácticos

- [Documentación de DbUp](https://dbup.readthedocs.io/) — corta y completa. Las páginas de *Script Providers* y *Transactions* son las dos que hay que leer enteras antes del primer despliegue serio.
- [DbUp en GitHub](https://github.com/DbUp/DbUp) — el código fuente es lo bastante pequeño para leerlo. `SqlTableJournal` y `UpgradeEngine` ocupan unas doscientas líneas cada uno y aclaran de una vez qué se registra, cuándo y en qué orden.
- [Migraciones de esquema](Migraciones-de-Esquema.md) y [Flyway](Flyway.md) — la teoría común y la alternativa directa. Compararlas es la forma rápida de ver qué te da DbUp y a qué estás renunciando.

---

*En resumen: DbUp es Flyway convertido en código C# y sin dependencias en el destino — a cambio de dos disciplinas que tienes que imponer tú: numerar con ceros a la izquierda, porque el orden es alfabético, y no editar nunca un script aplicado, porque nadie valida los checksums.*
