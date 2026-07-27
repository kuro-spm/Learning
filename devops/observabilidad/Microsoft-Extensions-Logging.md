# Microsoft.Extensions.Logging

## ¿Qué es?

`Microsoft.Extensions.Logging` (abreviado **M.E.L**) es el sistema de logging que viene de serie con .NET. No es una pieza sino dos: una **abstracción** común (`ILogger`) contra la que escribe tu código, y un modelo de **proveedores** enchufables que deciden a dónde acaban de verdad los logs.

## ¿Por qué existe?

Antes de M.E.L, cada librería traía su propio sistema de logging. Una aplicación que usara tres librerías tenía tres configuraciones incompatibles y ninguna forma unificada de decir "todo a consola en local, solo errores en producción". Y si querías cambiar de destino, había que tocar el código que escribía los logs.

M.E.L pone una capa en medio y separa dos decisiones que no tienen por qué ir juntas:

- **Qué se registra y con qué severidad** — lo decide el código, con `ILogger`.
- **A dónde va y qué se conserva** — se decide una sola vez al arrancar, con proveedores y configuración.

El resultado práctico: cambiar de escribir en consola a mandar los logs a [Serilog](Serilog.md) y de ahí a Elasticsearch no toca ni una línea del código de negocio.

> Es el mismo espíritu que un repositorio en acceso a datos: separa "qué consulta hago" de "contra qué motor se ejecuta". Aquí separa "qué log genero" de "dónde acaba".

## ¿Cuándo y para qué se usa?

En prácticamente cualquier aplicación .NET moderna —API, worker, consola—, porque **ya está montado** cuando creas el proyecto. `WebApplication.CreateBuilder(args)` registra el sistema de logging con un par de proveedores por defecto antes de que escribas nada.

Y aunque acabes usando Serilog o NLog, sigues usando M.E.L: esas librerías no lo reemplazan, se enganchan por debajo como proveedor. El código sigue inyectando `ILogger<T>`.

---

## Las tres piezas

Conviene distinguirlas porque los tres nombres aparecen en la documentación y hacen cosas distintas:

| Pieza | Qué es |
|---|---|
| `ILogger` / `ILogger<T>` | Lo que inyectas y usas para escribir. Ver [ILogger&lt;T&gt;](ILogger-T.md). |
| `ILoggerFactory` | Crea loggers. Rara vez lo usarás directamente. |
| `ILoggerProvider` | Un destino: consola, depurador, fichero, Serilog. |

El flujo completo de un log:

```
logger.LogInformation(...)
        ↓
  ¿supera el nivel mínimo de su categoría?   → si no, se descarta aquí
        ↓
  se entrega a TODOS los proveedores registrados
        ↓
  [Console]  [Debug]  [Serilog → fichero + Seq]
```

Cada proveedor recibe **una copia del mismo evento**. Añadir o quitar destinos no cambia el código que loguea.

## Escribir un log

No hay que registrar nada: el contenedor de dependencias resuelve `ILogger<T>` para cualquier `T`.

```csharp
public class ConfirmarPedido(ILogger<ConfirmarPedido> logger, IRepositorioPedidos repositorio)
{
    public async Task EjecutarAsync(int pedidoId)
    {
        logger.LogInformation("Confirmando pedido {PedidoId}", pedidoId);

        var pedido = await repositorio.ObtenerAsync(pedidoId);
        // ...
    }
}
```

Por consola sale así:

```
info: MiTienda.Aplicacion.ConfirmarPedido[0]
      Confirmando pedido 4711
```

`MiTienda.Aplicacion.ConfirmarPedido` es la **categoría**, y sale del `T`. Es lo que permite filtrar por origen, así que no es un detalle cosmético.

## Los proveedores

Se configuran al arrancar, y ese es el único sitio donde se decide el destino:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Logging.ClearProviders();
builder.Logging.AddConsole();
builder.Logging.AddDebug();
```

`ClearProviders()` merece explicación. La plantilla por defecto activa varios proveedores (Console, Debug, EventSource, y en Windows EventLog). Si vas a mandar los logs a un destino concreto, limpiar primero evita escribir cada línea tres veces en sitios que nadie consulta — coste de CPU y, en algunos casos, de dinero.

Los proveedores habituales:

| Proveedor | Para qué |
|---|---|
| `AddConsole()` | Salida estándar. En contenedores, el estándar de facto. |
| `AddDebug()` | Ventana de depuración del IDE. Solo en desarrollo. |
| `AddJsonConsole()` | Consola en JSON, para que lo consuma un agregador. |
| `AddApplicationInsights()` | Azure Monitor. |
| `AddSerilog()` / `UseNLog()` | Delegar en una librería completa. |

En un contenedor, escribir a la salida estándar es lo correcto: el runtime la captura y la reenvía a donde toque, y la aplicación no necesita saber nada del destino. La rotación de esos logs se configura en el motor de contenedores, como se explica en [Docker en un VPS](../despliegue-en-vps/Docker-en-un-VPS.md).

Para producción, la variante JSON es la que hace útil el [logging estructurado](Logging-Estructurado.md):

```csharp
builder.Logging.AddJsonConsole(options =>
{
    options.IncludeScopes = true;      // ← imprescindible, ver más abajo
    options.TimestampFormat = "O";
});
```

```json
{"Timestamp":"2026-07-27T10:15:32.4180000Z","LogLevel":"Information","Category":"MiTienda.Aplicacion.ConfirmarPedido","Message":"Confirmando pedido 4711","State":{"PedidoId":4711}}
```

## Categorías y filtrado

El filtrado por categoría es la función que más rendimiento da en el día a día. Se configura por `appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    },
    "Console": {
      "LogLevel": { "Default": "Debug" }
    }
  }
}
```

Dos cosas de este fichero:

- La sección `LogLevel` de primer nivel aplica **a todos** los proveedores.
- Una sección con el nombre de un proveedor (`Console`) lo sobrescribe **solo para él**. Así puedes tener `Debug` en consola y `Information` en el destino que se paga por GB.

El emparejamiento de categorías es por prefijo y gana la regla más específica. El detalle de cómo elegir umbrales está en [Niveles de Log](Niveles-de-Log.md).

También se puede filtrar en código, con más control:

```csharp
builder.Logging.AddFilter("Microsoft.EntityFrameworkCore.Database.Command", LogLevel.Information);
builder.Logging.AddFilter<ConsoleLoggerProvider>("MiTienda", LogLevel.Debug);
```

La segunda forma —filtro por proveedor concreto— no tiene equivalente cómodo en JSON y es útil cuando un destino es caro.

## Los ámbitos

Un **ámbito** (*scope*) adjunta propiedades a todos los logs generados dentro de un bloque, sin repetirlas en cada llamada:

```csharp
using (logger.BeginScope(new Dictionary<string, object> { ["PedidoId"] = pedidoId }))
{
    logger.LogInformation("Stock reservado");
    logger.LogInformation("Pago cobrado");
    // las dos líneas llevan PedidoId
}
```

Y aquí está el detalle que hace perder tiempo: **el proveedor de consola no muestra los ámbitos por defecto**. Escribes el `BeginScope`, no ves nada y concluyes que no funciona. Hay que activarlo:

```csharp
builder.Logging.AddConsole(options => options.IncludeScopes = true);
```

```
info: MiTienda.Aplicacion.ConfirmarPedido[0]
      => PedidoId:4711
      Stock reservado
```

Pasar un `Dictionary` en lugar de una cadena con formato no es indiferente: con un diccionario, cada clave se convierte en una propiedad estructurada aparte. Con `BeginScope("Pedido {PedidoId}", pedidoId)` funciona igual en Serilog, pero el formato varía entre proveedores.

Los ámbitos son la base de la correlación: un middleware que abre uno con el `traceId` al principio de cada petición hace que todos los logs de esa petición lo lleven. ASP.NET Core ya lo hace por ti con el identificador de la petición.

## `[LoggerMessage]`: logging sin coste

La llamada normal a `LogInformation` con plantilla tiene un coste en cada invocación: interpretar la plantilla, meter los argumentos en un `object[]` y hacer *boxing* de los tipos por valor. En una ruta que se ejecuta miles de veces por segundo, se nota.

El generador de código `[LoggerMessage]` construye ese logging en tiempo de compilación y elimina el coste:

```csharp
public static partial class LogsDePedidos
{
    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Information,
        Message = "Pedido {pedidoId} confirmado por {total}")]
    public static partial void PedidoConfirmado(ILogger logger, int pedidoId, decimal total);
}
```

```csharp
LogsDePedidos.PedidoConfirmado(logger, pedido.Id, pedido.Total);
```

Requisitos: la clase y el método han de ser `partial`, y el método `partial` sin cuerpo — lo genera el compilador. Además de rendimiento, aporta dos cosas que se agradecen: el `EventId` da un identificador estable sobre el que agrupar y alertar (más robusto que el texto del mensaje), y el compilador valida que los parámetros cuadren con los huecos de la plantilla, un error que en la forma normal solo se descubre mirando los datos.

Merece la pena en rutas calientes y en logs con muchos parámetros. Para el resto, la llamada normal está bien.

## Fuera de ASP.NET Core

M.E.L es un paquete `Microsoft.Extensions.*`, no depende de tener servidor web. En una aplicación de consola o un worker:

```csharp
var builder = Host.CreateApplicationBuilder(args);
builder.Logging.AddConsole();
builder.Services.AddHostedService<ProcesadorDePedidos>();

var host = builder.Build();
await host.RunAsync();
```

Y si necesitas un logger sin contenedor de dependencias —un script, un test— se crea a mano:

```csharp
using var factory = LoggerFactory.Create(b => b.AddConsole().SetMinimumLevel(LogLevel.Debug));
var logger = factory.CreateLogger<ProcesadorDePedidos>();
```

Ese `using` no es opcional: al liberar la fábrica se vacían los búferes de los proveedores. Sin él, un programa corto puede terminar **sin haber escrito sus últimos logs**, que suelen ser justo los del error que investigabas.

## Cómo se enganchan Serilog y NLog

No sustituyen a M.E.L: se registran como proveedor y reciben los eventos.

```csharp
// Serilog
builder.Services.AddSerilog((services, config) => config
    .ReadFrom.Configuration(builder.Configuration)
    .WriteTo.Console());

// NLog
builder.Logging.ClearProviders();
builder.Host.UseNLog();
```

El código de negocio no cambia: sigue inyectando `ILogger<T>` y llamando a `LogInformation`. Por eso las fichas de [Serilog](Serilog.md) y [NLog](NLog.md) hablan de configuración y destinos, no de una forma distinta de loguear.

Con una advertencia: cuando Serilog toma el control, **su propia configuración de niveles manda**. Poner `"Logging": { "LogLevel": ... }` en `appsettings.json` y esperar que Serilog lo respete es una causa clásica de "cambio el nivel y no pasa nada"; Serilog lee su sección `Serilog:MinimumLevel`.

## Buenas prácticas avanzadas

- **Filtra por categoría, nunca subas el umbral global para callar el ruido.** Es el reflejo natural ante un log saturado y apaga también tus propios eventos de negocio. Bajar el nivel de `Microsoft.*` y `System.Net.Http.*` deja limpio el registro sin perder visibilidad de la aplicación. Si acabas necesitando `Warning` global, el problema real es que hay demasiados `Information` mal puestos.
- **Comprueba `IsEnabled` solo cuando el argumento sea caro.** `if (logger.IsEnabled(LogLevel.Debug))` evita serializar un objeto grande cuando `Debug` está apagado, y no aporta nada para un `int`. Ponerlo en todas partes ensucia el código sin beneficio; no ponerlo donde hay una serialización deja un coste invisible en producción que no aparece en ningún perfil obvio.
- **Usa `EventId` en los logs sobre los que vayas a alertar.** El texto del mensaje es lo que la gente reformula sin pensar, y cada reformulación rompe en silencio la alerta que agrupaba por él. Un `EventId` numérico y estable —vía `[LoggerMessage]` o pasándolo explícitamente— convierte el log en un identificador con contrato, igual que el nombre de una métrica.
- **Cierra el host de forma ordenada para no perder los últimos logs.** Los proveedores con búfer escriben en segundo plano; un proceso que muere abruptamente se lleva lo que quedaba en cola, que es exactamente lo que ocurre cuando la aplicación se cae y es cuando más falta hace. `await host.StopAsync()` en el apagado, o el `using` de la fábrica, es lo que garantiza el vaciado.
- **Registra la configuración efectiva al arrancar.** Un `Information` inicial con el entorno, los proveedores activos y el nivel mínimo por categoría convierte "no aparecen los logs" en un diagnóstico inmediato. Sin eso, cada incidencia empieza por averiguar si el problema es que no pasa nada o que no se está registrando.

## Recursos didácticos

- [La guía de logging de .NET](https://learn.microsoft.com/dotnet/core/extensions/logging) — recorre proveedores, categorías, niveles y ámbitos con ejemplos ejecutables; la sección de reglas de filtrado explica el emparejamiento por prefijo mejor que ninguna otra fuente.
- [Documentación de `[LoggerMessage]`](https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator) — el generador de código con sus requisitos y las comparativas de rendimiento frente a la llamada normal.
- [Seq](https://datalust.co/seq) — un contenedor y `AddSeq()` bastan para ver qué llega realmente desde M.E.L, incluidos los ámbitos y las propiedades estructuradas, que en consola quedan escondidos.

---

*En resumen: `Microsoft.Extensions.Logging` separa qué registras de dónde acaba — tu código escribe contra `ILogger` y tú eliges aparte, al arrancar, los proveedores y el umbral de cada categoría.*
