# ILogger&lt;T&gt;

## ¿Qué es?

`ILogger<T>` es la interfaz de logging que se inyecta en las clases de una aplicación .NET. El parámetro genérico `T` no se usa para nada en tiempo de ejecución: solo sirve para que cada log quede etiquetado automáticamente con el nombre de la clase que lo generó.

## ¿Por qué existe?

Antes de esta abstracción había dos caminos, y los dos daban problemas. O el código escribía directamente a consola —sin forma de cambiar el destino ni de filtrar—, o cada equipo construía su propia clase `LoggerService` estática con la configuración dentro.

Esa segunda opción es la que más aparece todavía en código heredado, y falla en tres cosas: una clase estática no se puede sustituir en un test, la configuración queda repartida por el código en vez de en un sitio, y todos los logs acaban con el mismo origen, así que a gran escala no se sabe quién generó cada línea.

`ILogger<T>` resuelve las tres. Se inyecta como cualquier otra dependencia —sustituible en tests, sin estado global—, la configuración vive fuera, y el `T` incrusta la categoría sin que nadie la escriba a mano.

> Es el mismo espíritu que inyectar `IDbConnection` en lugar de construir la conexión en cada clase: el framework te entrega la instancia ya lista y ajustada a dónde se va a usar.

## ¿Cuándo y para qué se usa?

En cualquier clase que necesite registrar algo: un controlador, un caso de uso, un repositorio, un `BackgroundService`. Es el punto de entrada del sistema completo que se describe en [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md); esta ficha se ocupa de cómo se usa en el día a día.

---

## Inyectarlo

No hay que registrar nada en el contenedor: .NET sabe resolver `ILogger<T>` para cualquier `T`.

```csharp
public class ConfirmarPedido(
    ILogger<ConfirmarPedido> logger,
    IRepositorioPedidos repositorio)
{
    public async Task EjecutarAsync(int pedidoId)
    {
        logger.LogInformation("Confirmando pedido {PedidoId}", pedidoId);

        var pedido = await repositorio.ObtenerAsync(pedidoId);
        // ...
    }
}
```

El `T` debe ser **la clase que loguea**, no una interfaz ni una clase base. Es lo que hace que la categoría sea `MiTienda.Aplicacion.ConfirmarPedido` y no algo genérico:

```csharp
// ✅ Categoría: MiTienda.Aplicacion.ConfirmarPedido
ILogger<ConfirmarPedido> logger

// ❌ Categoría: MiTienda.Aplicacion.IConfirmarPedido — no distingue implementaciones
ILogger<IConfirmarPedido> logger
```

Esa categoría es lo que permite filtrar por origen sin tocar código:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "MiTienda.Infraestructura": "Warning"
    }
  }
}
```

## Las variantes: `ILogger`, `ILogger<T>` e `ILoggerFactory`

Los tres aparecen en la documentación y conviene saber cuándo toca cada uno:

| Tipo | Cuándo usarlo |
|---|---|
| `ILogger<T>` | Lo normal. Categoría automática. |
| `ILogger` | En un método `[LoggerMessage]` o en una clase estática auxiliar. |
| `ILoggerFactory` | Cuando la categoría se decide en tiempo de ejecución. |

El caso de `ILoggerFactory` es raro pero real: un procesador genérico que quiere que sus logs salgan con el nombre del tipo que está procesando en cada momento.

```csharp
public class ProcesadorDeMensajes(ILoggerFactory loggerFactory)
{
    public void Procesar(object mensaje)
    {
        var logger = loggerFactory.CreateLogger(mensaje.GetType());
        logger.LogInformation("Procesando mensaje");
    }
}
```

Fuera de ese tipo de escenario, `ILogger<T>` es la respuesta.

## Los métodos y sus sobrecargas

Hay un método por nivel, y todos tienen la misma forma:

```csharp
logger.LogTrace("...");
logger.LogDebug("...");
logger.LogInformation("...");
logger.LogWarning("...");
logger.LogError("...");
logger.LogCritical("...");
```

Lo que importa son las **sobrecargas con excepción**, porque usar la equivocada destruye información que después hace falta:

```csharp
// ✅ Guarda tipo, mensaje, stack trace e InnerException
logger.LogError(ex, "Fallo al cobrar el pedido {PedidoId}", pedido.Id);

// ❌ Guarda solo el texto: se pierde dónde saltó
logger.LogError("Fallo al cobrar el pedido {PedidoId}: {Error}", pedido.Id, ex.Message);
```

La diferencia en el resultado es enorme. La primera forma produce:

```
fail: MiTienda.Aplicacion.ConfirmarPedido[0]
      Fallo al cobrar el pedido 4711
      System.Net.Http.HttpRequestException: Connection refused
         at MiTienda.Infraestructura.PasarelaPago.CobrarAsync(...) in PasarelaPago.cs:line 42
         at MiTienda.Aplicacion.ConfirmarPedido.EjecutarAsync(...) in ConfirmarPedido.cs:line 28
```

La segunda produce una línea con `Connection refused` y ninguna pista de dónde. La excepción va **siempre como primer argumento**, nunca dentro de la plantilla.

## Plantillas con nombre, nunca interpolación

Esta es la regla que más se incumple, y la razón está en que las dos formas imprimen igual por consola:

```csharp
// ✅ PedidoId queda como campo consultable
logger.LogInformation("Pedido {PedidoId} confirmado", pedido.Id);

// ❌ El 4711 solo existe dentro de una cadena
logger.LogInformation($"Pedido {pedido.Id} confirmado");
```

Con la segunda forma se pierde la capacidad de filtrar por `PedidoId`, de agrupar eventos del mismo tipo y de construir cualquier panel. El porqué completo está en [Logging Estructurado](Logging-Estructurado.md).

Los huecos se emparejan **por posición, no por nombre**, así que esto compila y guarda los valores cruzados sin avisar:

```csharp
// ⚠️ PedidoId valdrá "tarjeta"
logger.LogInformation("Pedido {PedidoId} pagado con {MetodoPago}", metodoPago, pedido.Id);
```

El analizador que trae .NET avisa de la interpolación si activas las reglas de calidad de código (`CA2254`), y merece la pena tenerlo puesto: es un fallo que no se ve en ejecución.

## Probar código que loguea

Una de las ventajas de que `ILogger<T>` sea una interfaz inyectada es que se puede sustituir. Para la mayoría de los tests, lo que quieres es que no moleste:

```csharp
var servicio = new ConfirmarPedido(NullLogger<ConfirmarPedido>.Instance, repositorio);
```

`NullLogger<T>.Instance` descarta todo y no requiere configuración. Está en `Microsoft.Extensions.Logging.Abstractions`.

Si el test debe **ver** los logs —por ejemplo en un test de integración con xUnit—, lo habitual es un proveedor que escriba al `ITestOutputHelper`:

```csharp
var logger = LoggerFactory
    .Create(b => b.AddProvider(new XUnitLoggerProvider(output)))
    .CreateLogger<ConfirmarPedido>();
```

Lo que **no** conviene es afirmar sobre los logs en los tests unitarios. Verificar con un mock que se llamó a `LogInformation` con un texto concreto ata el test a la redacción del mensaje: cualquiera que mejore el texto rompe un test que no tenía nada que ver con el comportamiento. Los logs son un efecto secundario para personas, no parte del contrato de la clase. Si de verdad necesitas garantizar que algo queda registrado —una auditoría, por ejemplo—, eso no es un log: es una funcionalidad, y merece su propia abstracción.

## No lo envuelvas

Aparece en muchos proyectos y casi nunca compensa:

```csharp
// ❌ Envolver ILogger en una clase propia
public class LoggerService(ILogger<LoggerService> logger)
{
    public void Info(string mensaje) => logger.LogInformation(mensaje);
    public void Error(string mensaje, Exception ex) => logger.LogError(ex, mensaje);
}
```

Qué se pierde:

- **La categoría automática.** Todos los logs de toda la aplicación salen como `LoggerService`. Adiós al filtrado por origen.
- **Las plantillas estructuradas.** La firma `Info(string)` obliga a construir el texto antes de llamar, que es exactamente lo que rompe la estructura.
- **Los ámbitos, `IsEnabled`, `EventId`** y todo lo que no se haya replicado a mano en el envoltorio.

Cuando lo que se busca es añadir un campo común a todos los logs, la extensión correcta no es una capa por delante sino un *enricher* de [Serilog](Serilog.md) o un `ILoggerProvider` decorado, que actúan por debajo y no tocan cómo se escribe.

## Buenas prácticas avanzadas

- **Nunca metas datos personales en los campos, ni siquiera "temporalmente".** Un email o un DNI en un log queda indexado, replicado en las copias de seguridad y visible para todo el que tenga acceso al agregador, que suele ser bastante más gente que la que tiene acceso a la base de datos. Usa el identificador interno (`UsuarioId`): correlaciona igual de bien y no convierte el sistema de logs en una segunda copia de los datos personales, con las obligaciones legales que eso arrastra.
- **Usa `[LoggerMessage]` en las rutas calientes.** La llamada normal interpreta la plantilla y hace *boxing* de los argumentos en cada invocación; en código que se ejecuta miles de veces por segundo eso aparece en el perfil. El generador construye el método en compilación y además valida que los parámetros cuadren con los huecos, que es el error silencioso más común de la forma normal.
- **Registra el resultado, no la intención.** `logger.LogInformation("Cobrando pedido {PedidoId}")` antes de la llamada deja un log que miente si la operación falla justo después. Un solo evento al terminar, con el desenlace y la duración, cuenta más y cuesta la mitad. La excepción es una operación tan larga que necesites saber que empezó.
- **No registres una excepción que vas a relanzar.** Si haces `catch { logger.LogError(ex, ...); throw; }` y arriba hay un manejador global que también registra, la misma excepción aparece dos o tres veces y parece que hubo tres fallos. La convención sana: se registra donde se **maneja**, y quien solo propaga añade contexto a la excepción en lugar de un log nuevo.
- **Que la categoría refleje el diseño, no el árbol de carpetas.** Como la categoría es el *namespace* completo del tipo, la configuración de logging hereda la estructura del proyecto. Namespaces coherentes (`MiTienda.Aplicacion`, `MiTienda.Infraestructura`) permiten silenciar la infraestructura y dejar el dominio hablando con una sola línea de configuración; namespaces desordenados obligan a enumerar clases una a una.

## Recursos didácticos

- [La guía de logging de .NET](https://learn.microsoft.com/dotnet/core/extensions/logging) — la referencia oficial; la sección de categorías explica bien de dónde sale el nombre y cómo se usa en el filtrado.
- [Regla CA2254](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca2254) — el analizador que detecta plantillas construidas dinámicamente. Activarlo es la forma de que la regla de "nunca interpolar" no dependa de la revisión humana.
- [Seq](https://datalust.co/seq) — ver la misma llamada con plantilla y con interpolación llegando al agregador, y comprobar que solo una deja campos consultables, convence más rápido que cualquier explicación.

---

*En resumen: `ILogger<T>` se inyecta como cualquier dependencia y el propio `T` etiqueta cada log con su origen — no lo envuelvas en una clase propia y pasa siempre la excepción como primer argumento.*
