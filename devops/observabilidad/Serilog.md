# Serilog

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Serilog es una librería de logging para .NET diseñada desde el principio para el logging estructurado: en lugar de generar líneas de texto, captura cada log como un evento con propiedades con nombre que se pueden enviar a distintos destinos.

## ¿Por qué existe?

El logging "de toda la vida" en .NET (`Console.WriteLine`, o incluso `ILogger` básico sin configurar) trata cada log como una cadena de texto ya formateada. Serilog nace para resolver dos problemas a la vez: capturar el log como datos (no como texto) y poder mandarlo a varios sitios a la vez sin cambiar el código de la aplicación —a la consola en desarrollo, a un fichero rotado y a un sistema centralizado en producción, todo con la misma llamada `logger.LogInformation(...)`.

> Si conoces el patrón *repository* de acceso a datos: Serilog separa "cómo genero el evento de log" de "dónde acaba guardado", igual que un repositorio separa "qué consulta hago" de "en qué base de datos concreta se ejecuta".

## ¿Cuándo y para qué se usa?

En cualquier aplicación .NET (API, worker, aplicación de consola) que necesite logging estructurado en producción. Es la opción más extendida en el ecosistema .NET para sustituir o complementar el proveedor de logging por defecto de `Microsoft.Extensions.Logging`. Ejemplos: una API de una tienda online que necesita mandar sus logs a Elasticsearch además de a consola; un servicio en segundo plano que procesa pedidos y necesita guardar un fichero de logs rotado diariamente en disco.

## Lo mínimo que necesitas saber

**1. Configuración básica con un sink de consola**

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .WriteTo.Console()
    .CreateLogger();
```

**2. Los "sinks" son los destinos de los logs**

Un *sink* es a dónde va cada evento de log: consola, fichero, Seq, Elasticsearch, Application Insights... Se pueden combinar varios a la vez.

```csharp
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day)
    .WriteTo.Seq("http://localhost:5341")
    .CreateLogger();
```

**3. Logs con propiedades estructuradas**

```csharp
logger.LogInformation("Pedido {OrderId} confirmado por {Total:C}", pedido.Id, pedido.Total);
// "OrderId" y "Total" se guardan como campos propios, no solo dentro del texto
```

**4. Enriquecedores (*enrichers*): contexto añadido automáticamente**

```csharp
Log.Logger = new LoggerConfiguration()
    .Enrich.WithMachineName()
    .Enrich.WithProperty("Servicio", "pedidos")
    .WriteTo.Console()
    .CreateLogger();
// cada log incluye MachineName y Servicio sin tener que pasarlos a mano cada vez
```

**5. Integración con `ILogger` de ASP.NET Core**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog((context, config) =>
    config.ReadFrom.Configuration(context.Configuration)
          .WriteTo.Console());
```

Una vez configurado, el resto del código sigue inyectando `ILogger<T>` de forma normal: Serilog sustituye al proveedor por debajo, sin cambiar cómo se escriben los logs en los controladores o servicios.

## Lo que NO hace

- **No sustituye a un sistema de agregación de logs** — Serilog genera y envía los eventos; herramientas como Seq, Elasticsearch o Application Insights son las que los almacenan, indexan y permiten consultar.
- **No decide qué loguear** — sigue siendo responsabilidad de quien escribe el código elegir qué eventos merecen un log y con qué nivel.
- **No es exclusivo de ASP.NET Core** — funciona igual en aplicaciones de consola, servicios de Windows o *workers*, no depende de tener un servidor web.
- **No gestiona métricas ni trazas** — es una librería de logging; para eso conviene combinarlo con [OpenTelemetry](OpenTelemetry.md).

## Buenas prácticas avanzadas

- **Usa `LogContext.PushProperty` para el contexto de una petición, no variables globales** — envolver el procesamiento de una petición o un mensaje de cola en un `LogContext` con el `orderId` o el `traceId` hace que todos los logs generados dentro de ese ámbito lleven esa propiedad sin pasarla a mano en cada llamada, y desaparece automáticamente al salir del bloque.
- **Configura niveles mínimos distintos por namespace** — el ruido de las librerías de terceros (Entity Framework generando SQL, ASP.NET Core registrando cada petición) suele ahogar los logs propios de la aplicación; subir el nivel mínimo de esos namespaces concretos a `Warning` mientras el tuyo se queda en `Information` es una configuración habitual en cualquier proyecto serio.
- **No calcules el mensaje si el nivel está desactivado** — pasar directamente los parámetros a la plantilla (`logger.LogDebug("Payload: {Payload}", SerializarCaro(objeto))`) sigue evaluando `SerializarCaro` aunque `Debug` esté apagado, salvo que uses una sobrecarga o comprobación que lo evite; con objetos costosos de serializar, esto es un coste de rendimiento silencioso.
- **Usa sinks asíncronos o "buffered" en producción** — escribir a disco o a la red de forma síncrona en cada log añade latencia a la petición que lo generó. Los sinks pensados para producción encolan el evento y lo escriben en segundo plano, para que un pico de logs no ralentice las peticiones reales.

---

*En resumen: Serilog es la pieza de .NET que convierte cada log en un evento estructurado y lo reparte, sin cambiar el código de la aplicación, hacia todos los destinos que necesites: consola, fichero o un sistema centralizado.*
