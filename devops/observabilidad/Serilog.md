# Serilog

## ¿Qué es?

Serilog es una librería de logging para .NET diseñada desde el principio para el logging estructurado: en vez de generar líneas de texto, captura cada log como un **evento con propiedades con nombre** y lo reparte a los destinos que configures.

## ¿Por qué existe?

El proveedor de consola que trae [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md) es deliberadamente básico: escribe texto formateado a la salida estándar y poco más. Para producción hace falta más de lo que ofrece: mandar el mismo log a varios sitios a la vez, enviarlo a un sistema que lo indexe y lo haga consultable, añadir contexto automáticamente y hacerlo sin que escribir en disco frene las peticiones.

Serilog nació resolviendo eso con una decisión de diseño que lo distingue: **el evento de log es un objeto con propiedades, no una cadena**. El texto se genera al final, en el destino que lo necesite; el que sepa manejar estructura recibe los campos.

> Es el mismo espíritu que un repositorio en acceso a datos: separa "cómo genero el evento" de "dónde acaba guardado". Cambiar de fichero a Elasticsearch es cambiar una línea de configuración, no el código que loguea.

## ¿Cuándo y para qué se usa?

En prácticamente cualquier aplicación .NET que vaya a producción. Es la opción más extendida del ecosistema y la que asumen por defecto la mayoría de guías y plantillas.

Se elige frente a [NLog](NLog.md) sobre todo por su ecosistema de destinos y por lo natural que resulta configurarlo en código; NLog gana cuando el equipo de operaciones necesita reconfigurar el logging editando un fichero XML sin desplegar.

---

## Montarlo en una aplicación ASP.NET Core

Los paquetes:

```bash
dotnet add package Serilog.AspNetCore
```

`Serilog.AspNetCore` arrastra el núcleo y los destinos de consola y fichero, que cubren el arranque.

La configuración mínima:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSerilog((services, config) => config
    .ReadFrom.Configuration(builder.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .WriteTo.Console());

var app = builder.Build();
```

Qué hace cada línea:

- **`ReadFrom.Configuration`** lee la sección `Serilog` de `appsettings.json`, lo que permite cambiar niveles y destinos por entorno sin recompilar.
- **`ReadFrom.Services`** deja que Serilog resuelva *enrichers* y *sinks* registrados en el contenedor de dependencias.
- **`Enrich.FromLogContext()`** es **imprescindible**: sin ella, `LogContext.PushProperty` no hace absolutamente nada. Es la causa número uno de "mis propiedades de contexto no aparecen".
- **`WriteTo.Console()`** el destino.

Hecho esto, el resto del código no cambia: sigue inyectando `ILogger<T>` y llamando a `LogInformation`. Serilog se ha registrado como proveedor por debajo.

### El logger de arranque

Hay un hueco: si la aplicación revienta configurando el contenedor —una cadena de conexión mal puesta, un servicio que no resuelve— eso pasa **antes** de que Serilog esté montado, y el error se pierde. Justo el error que más cuesta diagnosticar.

La solución es un logger provisional que se crea en la primera línea del programa:

```csharp
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Arrancando la aplicación");

    var builder = WebApplication.CreateBuilder(args);
    builder.Services.AddSerilog((services, config) => config
        .ReadFrom.Configuration(builder.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext());

    var app = builder.Build();
    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "La aplicación no ha podido arrancar");
}
finally
{
    Log.CloseAndFlush();
}
```

`CreateBootstrapLogger()` crea un logger que después se **sustituye** por el definitivo sin perder lo que ya se escribió. Y `Log.CloseAndFlush()` en el `finally` vacía los búferes: sin él, un fallo de arranque puede terminar el proceso con el log del error todavía en cola y sin escribir.

## Los sinks

Un *sink* es un destino. Se pueden combinar todos los que quieras y cada uno recibe una copia del mismo evento.

```csharp
config
    .WriteTo.Console()
    .WriteTo.File("logs/tienda-.log",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 14)
    .WriteTo.Seq("http://localhost:5341");
```

En el destino de fichero, los dos parámetros importan más de lo que parece: `rollingInterval: Day` crea un fichero por día (`tienda-20260727.log`) y `retainedFileCountLimit: 14` borra los más antiguos. **Sin ese límite, los logs llenan el disco**, y un disco lleno tumba todo lo que corre en la máquina, no solo el logging.

Cada sink es un paquete NuGet aparte (`Serilog.Sinks.Console`, `Serilog.Sinks.File`, `Serilog.Sinks.Seq`, `Serilog.Sinks.Elasticsearch`...). Hay decenas, y ese catálogo es buena parte de por qué Serilog ganó.

Un detalle que se agradece saber: cada sink puede tener **su propio nivel mínimo**. Es la palanca para no pagar por lo que no vas a consultar:

```csharp
config
    .WriteTo.Console()                                              // todo
    .WriteTo.Seq("http://seq:5341",
        restrictedToMinimumLevel: LogEventLevel.Warning);           // solo lo grave
```

## Configurar desde `appsettings.json`

Todo lo anterior se puede escribir en configuración, que es lo habitual en producción porque permite cambiarlo por entorno:

```json
{
  "Serilog": {
    "Using": [ "Serilog.Sinks.Console", "Serilog.Sinks.Seq" ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning"
      }
    },
    "WriteTo": [
      { "Name": "Console" },
      { "Name": "Seq", "Args": { "serverUrl": "http://seq:5341" } }
    ],
    "Enrich": [ "FromLogContext", "WithMachineName" ],
    "Properties": { "Service": "pedidos" }
  }
}
```

Dos cosas de este fichero:

- **`MinimumLevel.Override`** es el equivalente al filtrado por categoría de M.E.L, y hace la misma falta: sin él, el ruido de EF Core y ASP.NET Core sepulta los logs propios.
- **Serilog ignora la sección `Logging` de `appsettings.json`.** Cuando Serilog está al mando, `"Logging": { "LogLevel": ... }` no hace nada. Es un despiste que cuesta un buen rato: hay que tocar `Serilog:MinimumLevel`.

## Enrichers: contexto automático

Un *enricher* añade propiedades a **todos** los eventos, sin tocar las llamadas:

```csharp
config
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .Enrich.WithProperty("Service", "pedidos");
```

Con esto, cada log lleva de qué máquina, entorno y servicio salió. En un sistema con varios servicios y varias instancias, eso es lo que permite responder "¿esto pasa en todas las instancias o solo en una?".

Y se pueden escribir propios. El caso típico es añadir el usuario autenticado a todos los logs de la petición:

```csharp
public class EnricherUsuario(IHttpContextAccessor accessor) : ILogEventEnricher
{
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory factory)
    {
        var usuarioId = accessor.HttpContext?.User?.FindFirst("sub")?.Value;
        if (usuarioId is not null)
            logEvent.AddPropertyIfAbsent(factory.CreateProperty("UsuarioId", usuarioId));
    }
}
```

Nota que se añade el **identificador**, no el email ni el nombre. El motivo está en [Logging Estructurado](Logging-Estructurado.md).

## `LogContext`: contexto de una operación

Los *enrichers* añaden lo mismo a todo. Para lo que cambia en cada operación está `LogContext`:

```csharp
using (LogContext.PushProperty("PedidoId", pedido.Id))
{
    logger.LogInformation("Stock reservado");
    logger.LogInformation("Pago cobrado");
    // las dos llevan PedidoId, y desaparece al salir del bloque
}
```

Recuerda: **requiere `Enrich.FromLogContext()`** en la configuración. Sin esa línea, el `using` se ejecuta y no añade nada.

Sigue el flujo asíncrono, así que los métodos `async` llamados desde dentro también lo llevan. Es el mecanismo con el que se propaga el `traceId` a todos los logs de una petición.

## El *middleware* de peticiones

ASP.NET Core registra por defecto varias líneas por petición, bastante ruidosas. Serilog ofrece cambiarlas por una sola con lo relevante:

```csharp
app.UseSerilogRequestLogging();
```

```
[10:15:32 INF] HTTP POST /api/pedidos/4711/confirmar responded 200 in 243.1181 ms
```

Y se le puede añadir contexto propio, que es donde se vuelve útil de verdad:

```csharp
app.UseSerilogRequestLogging(options =>
{
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("UsuarioId", httpContext.User?.FindFirst("sub")?.Value);
        diagnosticContext.Set("IP", httpContext.Connection.RemoteIpAddress?.ToString());
    };
});
```

Para que sustituya al log por defecto en lugar de sumarse, hay que silenciar el de ASP.NET Core con `"Microsoft.AspNetCore": "Warning"` en los *overrides*.

## Rendimiento: sinks asíncronos

Escribir a disco o a la red de forma síncrona añade esa latencia **a la petición que generó el log**. Con volumen, se nota.

```bash
dotnet add package Serilog.Sinks.Async
```

```csharp
config.WriteTo.Async(a => a.File("logs/tienda-.log", rollingInterval: RollingInterval.Day));
```

El evento se encola y un hilo en segundo plano lo escribe. La petición no espera.

El coste: si el proceso muere de golpe, lo que quedaba en la cola se pierde. Por eso `Log.CloseAndFlush()` en el apagado no es opcional cuando usas sinks asíncronos.

La consola **no** conviene envolverla en asíncrono: en un contenedor, la salida estándar ya está bufferizada por el runtime y añadir otra capa complica el orden de las líneas sin ganar nada.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `LogContext.PushProperty` no añade nada | Falta `Enrich.FromLogContext()` |
| Cambiar `Logging:LogLevel` no surte efecto | Serilog usa `Serilog:MinimumLevel` |
| No se escriben los últimos logs al terminar | Falta `Log.CloseAndFlush()` |
| Un sink configurado en JSON no funciona | Falta su paquete, o su nombre en `"Using"` |
| El disco se llena de ficheros de log | Falta `retainedFileCountLimit` |
| Se ven dos logs por cada petición HTTP | `UseSerilogRequestLogging` sin silenciar `Microsoft.AspNetCore` |

Cuando Serilog no escribe y no dice por qué, tiene un canal propio de diagnóstico:

```csharp
Serilog.Debugging.SelfLog.Enable(Console.Error);
```

Con eso, los fallos internos —un sink que no conecta, una plantilla inválida— salen por la salida de error en lugar de tragarse en silencio, que es el comportamiento por defecto y lo que hace estos problemas tan opacos.

## Buenas prácticas avanzadas

- **Trata `Log.CloseAndFlush()` como obligatorio, no como buena costumbre.** Con sinks asíncronos o con búfer, los eventos en cola se pierden si el proceso termina sin vaciarlos — y eso incluye el caso en que la aplicación se cae, que es cuando esos logs son lo único que tienes. Ponlo en un `finally` que envuelva todo el programa.
- **Configura un límite de retención y de tamaño en cada sink de fichero.** `retainedFileCountLimit` y `fileSizeLimitBytes` con `rollOnFileSizeLimit: true` acotan el consumo pase lo que pase. Un bucle de errores puede generar gigabytes en minutos, y un disco lleno no solo detiene el logging: tumba la base de datos, el proxy y todo lo demás que comparta la máquina.
- **Filtra por sink, no solo globalmente.** Serilog permite sub-loggers con reglas propias, lo que da el reparto que se acaba necesitando: todo a consola (gratis), solo `Warning` al agregador de pago, y los eventos de auditoría a un fichero aparte que se conserva un año. Un único nivel global obliga a elegir entre pagar de más o quedarse corto en local.
- **Escribe *enrichers* en vez de repetir propiedades.** Cada vez que veas la misma propiedad pasada a mano en varias llamadas, es candidata a *enricher* o a `LogContext`. Además de reducir ruido, garantiza que esté siempre: las propiedades que se escriben a mano faltan precisamente en el log que alguien añadió con prisa durante una incidencia.
- **Cuidado con destructurar objetos grandes en rutas frecuentes.** El operador `@` serializa el objeto completo, y con una entidad que arrastre colecciones eso son eventos de megabytes. Serilog aplica límites por defecto (profundidad y longitud de colección) que puedes ajustar con `Destructure.ToMaximumDepth`, pero la solución sana es destructurar DTOs pequeños diseñados para el log, no entidades de dominio.

## Recursos didácticos

- [Seq](https://datalust.co/seq) — del mismo autor que Serilog y encaja con él sin fricción: un contenedor, `WriteTo.Seq(...)` y ya puedes consultar tus logs por campo. Es la forma más rápida de ver para qué servía todo lo estructurado.
- [Wiki de Serilog](https://github.com/serilog/serilog/wiki) — la documentación oficial; las páginas de *Configuration Basics*, *Structured Data* y *Enrichment* cubren el 90 % de lo que necesitarás.
- [Catálogo de sinks](https://github.com/serilog/serilog/wiki/Provided-Sinks) — la lista completa de destinos disponibles, útil antes de escribir uno propio: casi siempre existe ya.

---

*En resumen: Serilog captura cada log como un evento con propiedades y lo reparte a los destinos que quieras — recuerda `Enrich.FromLogContext()` para que el contexto funcione y `CloseAndFlush()` para no perder los últimos.*
