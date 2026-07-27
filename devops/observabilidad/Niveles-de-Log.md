# Niveles de Log

## ¿Qué es?

Un nivel de log es la etiqueta de severidad que se asigna a cada mensaje (`Debug`, `Information`, `Warning`, `Error`...). Sirve para dos cosas a la vez: decir de un vistazo cuánto importa lo que ha pasado, y decidir por configuración qué se guarda en cada entorno y qué se descarta.

## ¿Por qué existe?

Sin niveles, todos los logs valdrían lo mismo: el detalle de depuración que solo interesa en local acabaría mezclado con el error que despierta a alguien de madrugada. En producción eso es un problema con dos caras: el volumen cuesta dinero, y el ruido esconde las señales que importan.

Los niveles ponen un **dial**. En el código escribes toda la información que puede ser útil, cada línea con su severidad, y luego decides por configuración —sin recompilar ni tocar nada— a partir de qué nivel se guarda en cada entorno. En local quieres verlo todo; en producción, de `Information` para arriba.

> Si conoces las configuraciones `Debug` y `Release` de .NET, la idea es parecida: el mismo código fuente se comporta distinto según un ajuste externo, no según una rama `if` que alguien escribió a mano.

## ¿Cuándo y para qué se usa?

En cualquier aplicación que genere logs, y en dos momentos distintos: al **escribir** cada mensaje (eliges su nivel) y al **configurar** cada entorno (eliges el umbral). Son dos decisiones independientes que suele tomar gente distinta, y esa separación es justo la gracia del mecanismo.

---

## Los seis niveles, y cómo elegir

En .NET (`Microsoft.Extensions.Logging`) la escala va de menos a más grave:

```csharp
logger.LogTrace("Entrando en CalcularDescuento con {PedidoId}", pedido.Id);
logger.LogDebug("Consulta ejecutada: {Sql} en {DuracionMs} ms", sql, ms);
logger.LogInformation("Pedido {PedidoId} confirmado por {Total}", pedido.Id, pedido.Total);
logger.LogWarning("Reintento {Intento} de 3 al enviar el email del pedido {PedidoId}", intento, pedido.Id);
logger.LogError(ex, "Fallo al cobrar el pedido {PedidoId}", pedido.Id);
logger.LogCritical(ex, "Base de datos inaccesible: el servicio no puede atender peticiones");
```

Qué significa cada uno en la práctica:

| Nivel | Cuándo | ¿En producción? |
|---|---|---|
| `Trace` | Detalle extremo: entrada y salida de métodos, valores intermedios | Nunca |
| `Debug` | Diagnóstico: SQL generado, payloads, decisiones internas | Solo al investigar |
| `Information` | Eventos de negocio: se confirmó un pedido, se registró un usuario | Sí |
| `Warning` | Algo raro, pero se ha podido continuar | Sí |
| `Error` | Una operación ha fallado; el servicio sigue en pie | Sí |
| `Critical` | La aplicación no puede seguir funcionando | Sí, con alerta |

Y el criterio que resuelve el 90 % de las dudas, en forma de preguntas encadenadas:

1. **¿Le importa a alguien que no sea quien programó esto?** Si no, es `Debug` o `Trace`.
2. **¿Es un hecho de negocio que querrías poder contar después?** Entonces `Information`.
3. **¿Ha fallado algo?** Si se recuperó solo, `Warning`. Si la operación no se completó, `Error`.
4. **¿Puede el servicio seguir atendiendo peticiones?** Si no, `Critical`.

### `Warning` frente a `Error`

La confusión más habitual. La pregunta que la resuelve: **¿la operación llegó a completarse?**

```csharp
// Warning: falló un intento, pero el reintento funcionó y el email se envió
logger.LogWarning("Reintento {Intento} al enviar el email del pedido {PedidoId}", 2, 4711);

// Error: se agotaron los reintentos, el email no se ha enviado
logger.LogError(ex, "No se pudo enviar el email del pedido {PedidoId} tras 3 intentos", 4711);
```

Que algo sea `Warning` no significa "poco importante", significa "el sistema lo absorbió". Es una distinción práctica: sobre `Error` se suelen montar alertas, sobre `Warning` no.

### `Error` frente a `Critical`

**¿Ha fallado una operación, o el sistema entero?**

Un pago rechazado por el banco es `Error`: ese pedido no se cobró, pero los demás siguen funcionando. Quedarse sin conexión a la base de datos es `Critical`: **ninguna** petición va a funcionar. La diferencia importa porque `Critical` suele estar enganchado a una alerta que llama a alguien por teléfono, y gastar esa alerta en un pago rechazado hace que en tres semanas nadie la atienda.

Un matiz que se pasa por alto: un pago rechazado por fondos insuficientes probablemente **no** es ni siquiera un `Error` de la aplicación. Es un resultado de negocio esperado, y encaja mejor como `Information` con un campo `Motivo`. Reserva `Error` para lo que no debería haber pasado.

## El umbral: "este nivel y todos los más graves"

No se eligen niveles sueltos: se elige un mínimo, y se guarda ese y todo lo que esté por encima.

```json
// appsettings.Development.json
{ "Logging": { "LogLevel": { "Default": "Debug" } } }

// appsettings.Production.json
{ "Logging": { "LogLevel": { "Default": "Information" } } }
```

Con `Default: "Information"` en producción, las llamadas a `LogTrace` y `LogDebug` no escriben nada.

Una precisión importante: **el código de esas llamadas sí se ejecuta**. La llamada al método ocurre, se comprueba el nivel y se descarta. Lo que significa que cualquier trabajo que hagas para construir sus argumentos se paga igualmente:

```csharp
// ⚠️ SerializarPedido() se ejecuta aunque Debug esté apagado
logger.LogDebug("Estado del pedido: {Estado}", SerializarPedido(pedido));
```

Para eso está `IsEnabled`, que se comprueba antes:

```csharp
if (logger.IsEnabled(LogLevel.Debug))
    logger.LogDebug("Estado del pedido: {Estado}", SerializarPedido(pedido));
```

Solo merece la pena cuando construir el argumento es caro. Para un `pedido.Id` la comprobación cuesta más que lo que ahorra.

## Filtrar por categoría, no solo globalmente

Aquí está el ajuste que más rendimiento da y que casi nadie hace de entrada.

El problema: las librerías de terceros generan muchísimo `Information`. Entity Framework registra cada consulta SQL, ASP.NET Core registra cada petición. Con `Default: Information` en producción, tus logs de negocio quedan sepultados bajo ese ruido.

El reflejo habitual es subir el umbral global a `Warning`, y es la decisión equivocada: silencia también **tus** eventos de negocio, y te quedas sin saber qué está haciendo tu propia aplicación.

Lo correcto es subir el umbral solo de las categorías ruidosas:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning",
      "System.Net.Http.HttpClient": "Warning",
      "MiTienda.Infraestructura": "Warning"
    }
  }
}
```

La **categoría** es normalmente el nombre completo del tipo, que es lo que aporta el `T` de [`ILogger<T>`](ILogger-T.md). El emparejamiento es por prefijo y gana **el más específico**: con la configuración de arriba, un log de `Microsoft.EntityFrameworkCore.Database.Command` se filtra con la regla de `Microsoft.EntityFrameworkCore`, y uno de `MiTienda.Aplicacion.ConfirmarPedido` cae en `Default`.

Comprobar qué categoría emite un log molesto es fácil: sale en la propia línea de la consola.

```
warn: Microsoft.EntityFrameworkCore.Query[10102]
      Compiling query expression...
```

Ese `Microsoft.EntityFrameworkCore.Query` es literalmente lo que hay que poner en la configuración.

## Subir el detalle sin desplegar

Cuando algo falla en producción, lo que quieres es ver los `Debug` **de una parte concreta** durante un rato, sin reiniciar el servicio ni desplegar.

En .NET, si la configuración viene de `appsettings.json`, el proveedor recarga los cambios en caliente: editar el fichero (o cambiar el valor en Azure App Configuration, en un ConfigMap de Kubernetes, etc.) surte efecto sin reiniciar.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "MiTienda.Aplicacion.Pagos": "Debug"
    }
  }
}
```

Serilog tiene además un mecanismo pensado para esto, `LoggingLevelSwitch`, que permite cambiar el nivel en tiempo de ejecución desde el propio código —por ejemplo, desde un endpoint protegido de administración.

Y la parte que se olvida siempre: **volver a bajarlo**. Un `Debug` que se activó para investigar una incidencia y se quedó puesto genera un volumen y una factura que aparecen tres meses después sin que nadie recuerde por qué. Ponerse un recordatorio al activarlo es menos ridículo de lo que suena.

## Los nombres cambian según la librería

La escala de severidad es la misma en todas partes; las etiquetas no. Conviene tener la equivalencia a mano cuando trabajas con código heredado:

| .NET (M.E.L) | Serilog | NLog | log4net | Syslog |
|---|---|---|---|---|
| `Trace` | `Verbose` | `Trace` | — | debug |
| `Debug` | `Debug` | `Debug` | `DEBUG` | debug |
| `Information` | `Information` | `Info` | `INFO` | info |
| `Warning` | `Warning` | `Warn` | `WARN` | warning |
| `Error` | `Error` | `Error` | `ERROR` | err |
| `Critical` | `Fatal` | `Fatal` | `FATAL` | crit |

Los dos detalles que despistan: Serilog llama `Verbose` a lo que .NET llama `Trace`, y lo que en .NET es `Critical` en Serilog, NLog y log4net es `Fatal`. Cuando Serilog actúa como proveedor de `Microsoft.Extensions.Logging` la traducción es automática, pero al leer un `nlog.config` o un `appsettings` heredado conviene saberlo.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| No aparece ningún log en producción | El umbral global está en `Warning` o más |
| Los logs propios se ahogan entre ruido | Falta filtrar por categoría las librerías de terceros |
| Un `LogError` no muestra dónde falló | Se pasó `ex.Message` en vez de la excepción como primer argumento |
| El nivel cambia en `appsettings` y no surte efecto | Serilog está configurado en código y no lee `appsettings` |
| Producción va lenta al activar `Debug` | Argumentos caros construidos sin comprobar `IsEnabled` |

El segundo de la lista tiene una versión concreta que cuesta muchas horas: **`logger.LogError(ex, "...")` guarda el *stack trace* completo**, mientras que `logger.LogError("Falló: {Error}", ex.Message)` guarda solo el texto y **pierde dónde saltó**.

```csharp
// ❌ "Object reference not set to an instance of an object" y nada más
logger.LogError("Error al confirmar: {Mensaje}", ex.Message);

// ✅ Con stack trace, tipo de excepción e InnerException
logger.LogError(ex, "Error al confirmar el pedido {PedidoId}", pedido.Id);
```

La excepción va **siempre como primer argumento**, no dentro de la plantilla.

## Buenas prácticas avanzadas

- **Reserva `Information` para eventos de negocio, no para trazar el flujo del código.** "Pedido confirmado" o "usuario registrado" son `Information`; "entrando en el método X" es `Debug`. Es el error más común y tiene un efecto en cadena: llenar producción de `Information` de diagnóstico obliga a subir el umbral, y al subirlo se pierden también los eventos que sí importaban. Un `Information` debería poder contarse y significar algo para alguien de negocio.
- **Un `Warning` sin acción posible debería ser `Debug`.** La prueba: si nadie va a hacer nada al verlo, no es un aviso, es ruido con una etiqueta llamativa. Los `Warning` que se repiten cien veces al día y todo el mundo ignora entrenan al equipo a ignorar también los que importan; revisarlos periódicamente y degradar los inútiles mantiene el nivel con significado.
- **Ajusta el umbral por categoría también en el destino, no solo en la aplicación.** Es habitual querer `Information` en consola (barato) pero solo `Warning` en el sistema centralizado (que se paga por GB). Serilog y NLog permiten un mínimo distinto por *sink*: `WriteTo.Console()` con `Information` y `WriteTo.Seq(restrictedToMinimumLevel: LogEventLevel.Warning)`. Es la palanca que baja el coste sin perder capacidad de diagnóstico local.
- **No deduzcas métricas contando líneas de log.** Contar cuántos `Warning` hubo para saber "cuántos pagos se reintentaron" es frágil: depende de que nadie cambie el nivel ni el texto, y se rompe en cuanto el muestreo o el filtrado descartan eventos. Si necesitas medir la frecuencia de algo, eso es una [métrica](Metricas.md), que además cuesta una fracción de lo que cuesta guardar los logs equivalentes.
- **Registra siempre el arranque con la configuración efectiva.** Un `Information` al iniciar que diga qué umbral, qué destinos y qué entorno están activos convierte "no veo logs" en un diagnóstico de diez segundos. Sin eso, la duda entre "no está pasando nada" y "está pasando pero no se registra" consume la primera media hora de cada incidencia.

## Recursos didácticos

- [La tabla de niveles de la documentación de .NET](https://learn.microsoft.com/dotnet/core/extensions/logging#log-level) — la referencia oficial de los seis niveles con el criterio de Microsoft para cada uno, útil para zanjar discusiones de equipo sobre qué es `Warning` y qué es `Error`.
- [Seq](https://datalust.co/seq) — filtrar por nivel sobre logs reales y ver cuánto ruido genera cada uno es la forma más rápida de calibrar el criterio; su vista de volumen por nivel enseña de inmediato dónde se va el presupuesto.
- [The Twelve-Factor App, *Logs*](https://12factor.net/logs) — el argumento de por qué la aplicación debería escribir a la salida estándar y desentenderse del destino, que es la base del modelo de niveles + proveedores.

---

*En resumen: el nivel de log es el dial de severidad de cada mensaje — escribes todo en el código y decides por configuración, categoría a categoría, a partir de qué gravedad se guarda.*
