# ILogger&lt;T&gt;

**¿Qué es?** `ILogger<T>` es la abstracción de logging incluida de serie en .NET (`Microsoft.Extensions.Logging`): una interfaz genérica que, inyectada por constructor, etiqueta automáticamente cada log con el nombre de la clase que lo genera.

---

## ¿Por qué existe?

Antes de esta abstracción, el código o bien escribía logs directamente a consola, o construía clases propias tipo `LoggerService`/`AppLogger` para no repetir configuración en cada sitio. Ambos caminos complican los tests (una clase estática o global es difícil de sustituir por un doble de prueba) y no dejan claro, a gran escala, quién generó cada línea. `ILogger<T>` resuelve las dos cosas: se inyecta como cualquier otra dependencia, y el parámetro genérico `T` incrusta el nombre de la clase como "categoría" del log de forma automática, sin pasarlo a mano.

> Si conoces Dapper: es el mismo espíritu que inyectar `IDbConnection` en vez de crear la conexión a mano en cada clase — es el framework quien te entrega la instancia ya lista, ajustada a dónde se usa.

---

## ¿Cuándo y para qué se usa?

En cualquier clase de una aplicación .NET que necesite generar logs: un controlador, un caso de uso, un repositorio. Se inyecta como una dependencia más, sin configuración adicional por clase.

---

## Lo mínimo que necesitas saber

**1. Se inyecta con la propia clase como parámetro genérico**

```csharp
public class ConfirmarPedido(ILogger<ConfirmarPedido> logger, IRepositorioPedidos repositorio)
{
    public async Task Ejecutar(int pedidoId)
    {
        logger.LogInformation("Confirmando pedido {PedidoId}", pedidoId);
        // ...
    }
}
```

El tipo `T` no se usa en tiempo de ejecución dentro del método: solo sirve para que el framework rellene la categoría del log (algo como `MiApp.ConfirmarPedido`) sin que haya que escribirla a mano.

**2. La categoría permite ajustar el nivel de log por origen**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "MiApp.Infraestructura": "Warning"
    }
  }
}
```

**3. Las plantillas con nombre son obligatorias, nunca interpolación**

```csharp
// ✅ el motor de logging extrae PedidoId como campo propio
logger.LogInformation("Creado {PedidoId}", pedidoId);

// ❌ destruye la estructura: ya es un texto cerrado, no un dato con campos
logger.LogInformation($"Creado {pedidoId}");
```

**4. No hace falta registrar nada manualmente**

El contenedor de dependencias de .NET sabe resolver `ILogger<T>` para cualquier `T` sin que haya que añadir una línea de registro por cada clase.

---

## Lo que NO hace

- **No decide dónde acaban los logs** — eso lo define el proveedor configurado por debajo (consola, fichero, [Serilog](Serilog.md)...); `ILogger<T>` es solo el punto de entrada común.
- **No filtra por sí solo datos sensibles** — que una plantilla use placeholders con nombre no impide que alguien pase un email, una tarjeta o una contraseña como valor de un campo; sigue siendo responsabilidad de quien escribe el log decidir qué datos van dentro (ver [Logging Estructurado](Logging-Estructurado.md)).

---

## Buenas prácticas avanzadas

- **No lo envuelvas en una clase propia "por comodidad"** — un wrapper custom (`LoggerService`, `AppLogger`) suele perder la categoría automática (todos los logs acaban con el mismo origen genérico), complica sustituirlo en tests y reinventa algo que la plataforma ya resuelve bien. Si hace falta comportamiento común (enriquecer con un campo fijo), un *enricher* o un decorador de `ILoggerProvider` es la extensión correcta, no una capa intermedia por delante de `ILogger<T>`.
- **Nunca metas información de identificación personal (PII) en los campos del log** — un email, un DNI o una dirección quedan indexados y buscables por cualquiera con acceso al sistema de logs, que suele ser más gente de la que tiene acceso directo a la base de datos; usa un identificador interno (`UserId`) en vez del dato personal cuando el log solo necesita poder correlacionarse con ese usuario.
- **Usa `LoggerMessage.Define` o el atributo `[LoggerMessage]` en rutas calientes** — la llamada habitual a `LogInformation` con una plantilla tiene un coste de *parsing* y *boxing* en cada llamada; en código que se ejecuta miles de veces por segundo, el generador de código asociado a `[LoggerMessage]` construye ese logging en tiempo de compilación y elimina ese coste.

---

*En resumen: `ILogger<T>` es la forma estándar de loguear en .NET — se inyecta como cualquier dependencia y el propio `T` etiqueta cada log con su origen, sin necesidad de envolturas propias por delante.*
