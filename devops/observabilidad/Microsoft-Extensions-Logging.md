# Microsoft.Extensions.Logging

## ¿Qué es?

`Microsoft.Extensions.Logging` (a menudo abreviado "M.E.L") es el sistema de logging que viene de serie con .NET. No es una sola pieza, sino dos: una **abstracción** común (`ILogger`, `ILoggerFactory`) que usa tu código, y un modelo de **proveedores** (*providers*) enchufables que deciden a dónde acaban de verdad los logs (consola, depurador, un fichero, Serilog...).

## ¿Por qué existe?

Antes, cada librería traía su propio sistema de logging y su propia forma de configurarlo. Si tu aplicación usaba tres librerías que logueaban de tres maneras distintas, tenías tres configuraciones incompatibles y ninguna manera unificada de decir "quiero verlo todo en consola en local y solo los errores en producción".

M.E.L pone una capa común en medio: tu código y las librerías escriben contra la misma interfaz `ILogger`, y tú eliges por separado —una sola vez, al arrancar— a dónde van esos logs y con qué nivel. Cambiar de destino (de consola a Serilog, por ejemplo) no toca el código que loguea.

> Si conoces el patrón *repository* de acceso a datos: M.E.L separa "cómo genero el log" de "dónde acaba", igual que un repositorio separa "qué consulta hago" de "en qué motor se ejecuta". El código depende de la abstracción, no del destino concreto.

## ¿Cuándo y para qué se usa?

En prácticamente cualquier aplicación .NET moderna (API, worker, app de consola): es el logging que ya está montado cuando creas el proyecto con `WebApplication.CreateBuilder`. Se usa tanto para *escribir* logs desde el código como para *configurar*, al arrancar, qué proveedores están activos y qué nivel mínimo tiene cada categoría. Ejemplos: una API que escribe a consola en desarrollo y a Application Insights en producción; un worker de procesado de pedidos que quiere silenciar el ruido de Entity Framework pero ver todo lo suyo.

## Lo mínimo que necesitas saber

**1. Ya está registrado: solo pides `ILogger<T>` y logueas**

```csharp
public class ConfirmarPedido(ILogger<ConfirmarPedido> logger)
{
    public void Ejecutar(int orderId) =>
        logger.LogInformation("Confirmando pedido {OrderId}", orderId);
}
```

`ILogger<T>` es el punto de entrada; su ficha propia lo detalla en [ILogger&lt;T&gt;](ILogger-T.md). M.E.L es el sistema completo que hay detrás.

**2. Los *providers* son los destinos, y se añaden al arrancar**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Logging.ClearProviders();
builder.Logging.AddConsole();   // a la salida estándar
builder.Logging.AddDebug();     // a la ventana de depuración del IDE
```

Cada *provider* (Console, Debug, EventLog, ApplicationInsights...) recibe una copia del mismo log. Añadir o quitar destinos no cambia ni una línea del código que escribe los logs.

**3. El nivel mínimo se configura por categoría, no solo global**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  }
}
```

La "categoría" es normalmente el nombre completo de la clase (lo que aporta el `T` de `ILogger<T>`). Aquí se silencia el ruido de ASP.NET Core y EF Core sin afectar a los logs propios de la aplicación. Los niveles se explican en [Niveles de Log](Niveles-de-Log.md).

**4. Los *scopes* añaden contexto a un bloque de logs**

```csharp
using (logger.BeginScope("Pedido {OrderId}", orderId))
{
    logger.LogInformation("Stock reservado");
    logger.LogInformation("Pago cobrado");
    // ambas líneas quedan asociadas a ese OrderId
}
```

Un *scope* adjunta información a todos los logs generados dentro del bloque, sin repetirla en cada llamada. Es la base sobre la que las librerías estructuradas construyen la correlación de una petición completa.

**5. Los frameworks de terceros se enganchan como un proveedor más**

Serilog, NLog o log4net no reemplazan a M.E.L: se conectan por debajo como el proveedor que recibe los logs. Por eso el código sigue inyectando `ILogger<T>` de siempre, aunque quien los escriba a disco sea [Serilog](Serilog.md).

## Lo que NO hace

- **No indexa ni almacena logs** — M.E.L los genera y los reparte a los proveedores; guardarlos, buscarlos y agregarlos es cosa del destino (un fichero, Seq, Application Insights...).
- **No hace logging estructurado potente por sí solo** — el proveedor de consola de serie sabe poco de estructura; para sinks estructurados de verdad, campos consultables y enriquecedores, se combina con [Serilog](Serilog.md) o [NLog](NLog.md).
- **No decide qué loguear** — sigue siendo responsabilidad de quien escribe el código elegir los eventos y su nivel.
- **No es exclusivo de ASP.NET Core** — es un paquete `Microsoft.Extensions.*` que funciona igual en consolas, servicios y *workers*, no depende de tener un servidor web.

## Buenas prácticas avanzadas

- **Configura el filtrado por categoría, no subiendo el umbral global** — el reflejo de "hay demasiado ruido, subo `Default` a `Warning`" apaga también tus propios `Information` útiles; lo correcto es bajar el nivel *de las categorías ruidosas concretas* (`Microsoft.*`, `System.Net.Http.*`) y dejar el tuyo donde estaba.
- **Usa el generador de código `[LoggerMessage]` en rutas calientes** — la llamada normal a `LogInformation` con plantilla tiene coste de *parsing* y *boxing* en cada invocación; el atributo `[LoggerMessage]` genera el método de logging en tiempo de compilación y lo elimina, algo que importa en código que se ejecuta miles de veces por segundo.
- **Comprueba `IsEnabled` antes de construir un argumento caro** — si un log de `Debug` necesita serializar un objeto grande, `if (logger.IsEnabled(LogLevel.Debug))` evita pagar esa serialización cuando `Debug` está apagado en producción; pasar el objeto directo a la plantilla lo evalúa igualmente aunque el nivel esté desactivado.
- **`ClearProviders()` antes de añadir los tuyos en producción** — la plantilla por defecto activa varios proveedores (Console, Debug, EventSource); si vas a mandar los logs a un sink concreto, limpiar primero evita duplicar cada línea y gastar recursos escribiéndola en destinos que no consultas.

## Recursos didácticos

- La [guía de logging en .NET](https://learn.microsoft.com/dotnet/core/extensions/logging) de Microsoft, que recorre proveedores, categorías, niveles y scopes con ejemplos ejecutables.

---

*En resumen: `Microsoft.Extensions.Logging` es la capa común de logging de .NET — tu código escribe contra `ILogger` y tú eliges aparte, al arrancar, a qué proveedores va cada log y con qué nivel por categoría.*
