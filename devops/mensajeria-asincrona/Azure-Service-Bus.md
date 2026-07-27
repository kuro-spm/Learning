# Azure Service Bus

## ¿Qué es?

Azure Service Bus es el servicio de mensajería **gestionado** de Microsoft Azure: colas y topics listos para usar, sin instalar, actualizar ni escalar tú ningún broker.

## ¿Por qué existe?

Operar un broker propio implica instalarlo, mantenerlo parcheado, dimensionar su capacidad, montar alta disponibilidad y vigilarlo de madrugada. Para muchos equipos —sobre todo si ya despliegan en Azure— ese trabajo no aporta valor de negocio.

Service Bus ofrece las mismas ideas de fondo (colas, publicación/suscripción, entrega fiable) como servicio operado por Microsoft: se crea desde el portal o con un comando, se paga por uso, y la disponibilidad es responsabilidad del proveedor. Y como es un servicio propietario, incorpora funciones que en [RabbitMQ](RabbitMQ.md) tendrías que construir: sesiones para garantizar orden, envíos programados, dead-lettering automático y transacciones entre entidades.

> Es a RabbitMQ lo que una base de datos gestionada es a instalar el motor en tu propia máquina virtual: las mismas ideas, sin que tú administres el software de base.

## ¿Cuándo y para qué se usa?

Cuando la aplicación ya vive en Azure y no quieres asumir la carga operativa de un broker. Casos típicos: una API que encola el envío de emails de confirmación; varios servicios suscritos a un mismo topic de eventos de pedidos, cada uno con su suscripción independiente; procesos que requieren orden estricto por entidad, donde las sesiones ahorran bastante trabajo.

Deja de encajar si necesitas independencia de proveedor, si el volumen es tan alto que la facturación por operaciones se dispara, o si lo que buscas es un registro de eventos releíble (para eso, Event Hubs o Kafka).

---

## Crear la infraestructura

Todo cuelga de un **espacio de nombres** (*namespace*), que es el contenedor donde viven las colas y los topics y el que define el nivel de servicio y el precio.

```bash
az servicebus namespace create \
  --resource-group tienda-rg \
  --name tienda-bus \
  --location westeurope \
  --sku Standard

az servicebus queue create \
  --resource-group tienda-rg \
  --namespace-name tienda-bus \
  --name envio-emails
```

El `--sku` es la decisión con más consecuencias y se explica más abajo. Elegir **Standard** es correcto para empezar: es el único nivel que incluye topics y se puede subir a Premium después.

Y en el proyecto .NET:

```bash
dotnet add package Azure.Messaging.ServiceBus
```

## Conectarse

Hay dos formas, y la diferencia importa.

**Cadena de conexión** — cómoda para empezar y para desarrollo local:

```csharp
await using var client = new ServiceBusClient(connectionString);
```

Esa cadena contiene una clave compartida en texto plano. Es una credencial de larga duración que hay que rotar, custodiar y mantener fuera del repositorio (ver [Por qué los secretos no van a Git](../../seguridad/gestion-de-secretos-en-desarrollo/Por-Que-Los-Secretos-No-Van-A-Git.md)).

**Identidad gestionada** — lo correcto en producción dentro de Azure:

```csharp
await using var client = new ServiceBusClient(
    "tienda-bus.servicebus.windows.net",
    new DefaultAzureCredential());
```

Aquí no hay ninguna credencial en la configuración. `DefaultAzureCredential` obtiene un token de la identidad asignada al recurso (App Service, Container App, VM) y lo renueva solo. En tu portátil, el mismo código usa la sesión de `az login`, así que funciona en ambos sitios sin cambios.

El `ServiceBusClient` es caro de crear y seguro para uso concurrente: **créalo una vez y compártelo**. En una aplicación con inyección de dependencias, registrado como *singleton*:

```csharp
builder.Services.AddSingleton(_ => new ServiceBusClient(
    "tienda-bus.servicebus.windows.net", new DefaultAzureCredential()));
```

Crear uno por mensaje abre una conexión AMQP cada vez y es una causa habitual de agotamiento de puertos bajo carga.

## Enviar mensajes

```csharp
var sender = client.CreateSender("envio-emails");

var mensaje = new ServiceBusMessage(
    JsonSerializer.SerializeToUtf8Bytes(new EnviarEmailConfirmacion { PedidoId = 4711 }))
{
    MessageId = $"email-confirmacion-4711",
    ContentType = "application/json",
    Subject = nameof(EnviarEmailConfirmacion)
};

await sender.SendMessageAsync(mensaje);
```

Las propiedades que conviene rellenar siempre:

- **`MessageId`** — el identificador que usará el consumidor para detectar duplicados. Si lo derivas del dato de negocio (`email-confirmacion-4711`) en lugar de un GUID aleatorio, dos publicaciones del mismo hecho comparten identificador y el consumidor puede descartarlas.
- **`Subject`** — el tipo de mensaje. Permite al consumidor decidir cómo deserializar sin abrir el cuerpo, y es lo que usarás en los filtros de las suscripciones.

Para enviar muchos mensajes, hacerlo de uno en uno paga un viaje de red por mensaje. Un lote los agrupa y respeta solo el límite de tamaño:

```csharp
using var lote = await sender.CreateMessageBatchAsync();

foreach (var pedido in pedidos)
{
    var m = new ServiceBusMessage(JsonSerializer.SerializeToUtf8Bytes(pedido));
    if (!lote.TryAddMessage(m))
        break;   // el lote está lleno: enviar este y empezar otro
}

await sender.SendMessagesAsync(lote);
```

`TryAddMessage` devolviendo `false` es la forma de saber que se alcanzó el límite. Ignorar ese valor y asumir que todos entraron hace que se pierdan mensajes en silencio cuando el lote se llena.

## Recibir: el bloqueo y sus consecuencias

El modelo por defecto se llama **PeekLock** y funciona así: al entregarte un mensaje, Service Bus no lo borra, lo **bloquea** durante un tiempo (30 segundos por defecto). Durante ese rato es invisible para los demás consumidores. Si confirmas, se borra; si no, el bloqueo caduca y el mensaje vuelve a estar disponible.

```csharp
var processor = client.CreateProcessor("envio-emails", new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 5,
    AutoCompleteMessages = false
});

processor.ProcessMessageAsync += async args =>
{
    var evento = args.Message.Body.ToObjectFromJson<EnviarEmailConfirmacion>();
    await EnviarEmailAsync(evento.PedidoId);

    await args.CompleteMessageAsync(args.Message);
};

processor.ProcessErrorAsync += args =>
{
    logger.LogError(args.Exception, "Error en {Origen}", args.ErrorSource);
    return Task.CompletedTask;
};

await processor.StartProcessingAsync();
```

Las opciones que hay que entender:

- **`MaxConcurrentCalls`** — cuántos mensajes se procesan a la vez. Sube el rendimiento y también el consumo de recursos; el valor por defecto es 1.
- **`AutoCompleteMessages = false`** — desactiva la confirmación automática. Con el valor por defecto (`true`), el mensaje se confirma al terminar el manejador **sin excepción**, lo que incluye casos en los que tu código decidió no hacer nada. Controlarlo a mano hace explícito qué se considera "procesado".

Y las cuatro formas de terminar con un mensaje:

| Llamada | Efecto |
|---|---|
| `CompleteMessageAsync` | Procesado. Se borra. |
| `AbandonMessageAsync` | Suelta el bloqueo **ya**: vuelve a la cola de inmediato y suma un intento. |
| `DeadLetterMessageAsync` | A la [cola de mensajes muertos](Dead-Letter-Queues.md), sin más reintentos. |
| `DeferMessageAsync` | Se aparta; solo recuperable por número de secuencia. |

La diferencia entre `AbandonMessageAsync` y no hacer nada es sutil y práctica: abandonar libera el mensaje al instante para reintentarlo; dejar que caduque el bloqueo consigue lo mismo pero 30 segundos después. Si sabes que ha fallado, abandónalo explícitamente y no hagas esperar al siguiente intento.

### Cuando el proceso tarda más que el bloqueo

Este es el fallo que más desconcierta. Si procesar tarda más que la duración del bloqueo, este caduca **mientras sigues trabajando**: el mensaje se reentrega a otro consumidor y, al terminar, tu confirmación falla:

```
ServiceBusException: The lock supplied is invalid. Either the lock expired,
or the message has already been removed from the queue. (MessageLockLost)
```

El resultado es trabajo duplicado justo bajo carga, que es cuando peor viene. Hay dos salidas:

```csharp
// Renovar el bloqueo mientras se trabaja
processor.ProcessMessageAsync += async args =>
{
    using var cts = new CancellationTokenSource();
    var renovacion = RenovarPeriodicamenteAsync(args, cts.Token);

    await ProcesoLargoAsync(args.Message);

    cts.Cancel();
    await args.CompleteMessageAsync(args.Message);
};
```

O subir `LockDuration` de la cola (máximo cinco minutos). La regla: dimensiona el bloqueo para el **peor** caso de tu consumidor, no para el medio.

### El otro modo: ReceiveAndDelete

`ReceiveMode.ReceiveAndDelete` borra el mensaje al entregarlo. Es más rápido y elimina toda la gestión de bloqueos, a cambio de que si el proceso muere el mensaje **se pierde**. Solo tiene sentido para datos descartables, como telemetría.

## Topics y suscripciones

Un topic es el punto de publicación; cada suscripción es una cola independiente que recibe **su propia copia** de cada mensaje.

```bash
az servicebus topic create -g tienda-rg --namespace-name tienda-bus -n pedidos-eventos

az servicebus topic subscription create -g tienda-rg --namespace-name tienda-bus \
  --topic-name pedidos-eventos -n facturacion

az servicebus topic subscription create -g tienda-rg --namespace-name tienda-bus \
  --topic-name pedidos-eventos -n inventario
```

El productor no cambia en nada respecto a una cola: solo apunta al topic.

```csharp
var sender = client.CreateSender("pedidos-eventos");
await sender.SendMessageAsync(new ServiceBusMessage(
    JsonSerializer.SerializeToUtf8Bytes(new PedidoConfirmado { PedidoId = 4711 }))
{
    Subject = nameof(PedidoConfirmado),
    ApplicationProperties = { ["Region"] = "ES", ["Total"] = 149.90 }
});
```

Y el consumidor indica topic **y** suscripción:

```csharp
var processor = client.CreateProcessor("pedidos-eventos", "facturacion");
```

### Filtros: que cada suscripción reciba solo lo suyo

Aquí está la ventaja real de los topics de Service Bus. Cada suscripción puede filtrar por las propiedades del mensaje, de modo que el productor no necesita saber quién recibe qué.

```bash
# La suscripción de facturación solo quiere pedidos españoles de más de 100 €
az servicebus topic subscription rule create \
  -g tienda-rg --namespace-name tienda-bus \
  --topic-name pedidos-eventos --subscription-name facturacion \
  --name solo-es-grandes \
  --filter-sql-expression "Region = 'ES' AND Total > 100"
```

La expresión se evalúa sobre `ApplicationProperties` y las propiedades del sistema (`Subject`, `MessageId`...), **nunca sobre el cuerpo**. Por eso los datos que necesites filtrar tienen que ir como propiedades, no solo dentro del JSON.

Un detalle que causa duplicados inesperados: toda suscripción nace con una regla por defecto llamada `$Default` que acepta todo. Si añades la tuya sin borrar aquella, el mensaje coincide con las dos y **se entrega dos veces**:

```bash
az servicebus topic subscription rule delete \
  -g tienda-rg --namespace-name tienda-bus \
  --topic-name pedidos-eventos --subscription-name facturacion --name '$Default'
```

## Sesiones: cuando el orden importa

Por defecto no hay garantía de orden con varios consumidores. Las **sesiones** la dan para un grupo de mensajes: agrupando por `SessionId`, Service Bus garantiza que se procesan en orden y **de uno en uno** dentro de esa sesión, bloqueando la sesión entera para un solo consumidor.

La cola debe crearse con sesiones activadas (`--enable-session`), y el productor etiqueta cada mensaje:

```csharp
var mensaje = new ServiceBusMessage(body)
{
    SessionId = pedido.Id.ToString()    // todos los eventos de este pedido, en orden
};
```

El consumidor usa un procesador de sesiones:

```csharp
var processor = client.CreateSessionProcessor("pedidos-eventos", "inventario",
    new ServiceBusSessionProcessorOptions { MaxConcurrentSessions = 10 });
```

La elección del `SessionId` lo es todo. Con el identificador del pedido, los eventos de pedidos distintos siguen procesándose en paralelo y solo se serializan los del mismo pedido: es lo que quieres. Con un valor fijo para todos, has convertido la cola en un único hilo y el rendimiento se desploma.

## Envío programado y caducidad

Dos funciones que ahorran construir un planificador propio:

```csharp
// Recordatorio de carrito abandonado dentro de 2 horas
long numeroSecuencia = await sender.ScheduleMessageAsync(
    mensaje, DateTimeOffset.UtcNow.AddHours(2));

// Y cancelarlo si el cliente termina la compra antes
await sender.CancelScheduledMessageAsync(numeroSecuencia);
```

```csharp
// Un mensaje que deja de tener sentido pasada una hora
var mensaje = new ServiceBusMessage(body) { TimeToLive = TimeSpan.FromHours(1) };
```

Al caducar, el mensaje va a la dead-letter queue con el motivo `TTLExpiredException`, no se borra en silencio.

## Los niveles de servicio

La elección del `--sku` condiciona precio, límites y funciones disponibles:

| | Basic | Standard | Premium |
|---|---|---|---|
| Colas | Sí | Sí | Sí |
| **Topics** | **No** | Sí | Sí |
| Sesiones, transacciones | No | Sí | Sí |
| Tamaño de mensaje | 256 KB | 256 KB | 100 MB |
| Recursos | Compartidos | Compartidos | Aislados |
| Facturación | Por operación | Por operación | Tarifa fija por unidad |

Basic se descarta casi siempre: sin topics, no puedes difundir eventos. Standard cubre la mayoría de escenarios. Premium aporta recursos aislados —latencia predecible, sin vecinos ruidosos— y cuesta bastante más; su tarifa fija solo compensa con volumen alto y sostenido. Subir a Premium "por si acaso" es el gasto innecesario más común en Service Bus.

## Buenas prácticas avanzadas

- **Deriva el `MessageId` del dato de negocio, no de un GUID aleatorio.** Con `$"pedido-confirmado-{pedidoId}"`, dos publicaciones del mismo hecho —por un reintento del productor o un [outbox](Outbox-Pattern.md) que republica— comparten identificador y el consumidor puede descartar la segunda. Con un GUID nuevo cada vez, son dos mensajes distintos indistinguibles y la idempotencia se queda sin punto de apoyo.
- **Activa la detección de duplicados solo si entiendes su ventana.** `--enable-duplicate-detection` hace que Service Bus descarte mensajes con un `MessageId` ya visto dentro de una ventana temporal (10 minutos por defecto, ampliable). Es útil, y engaña: fuera de esa ventana el duplicado pasa. No sustituye a un consumidor idempotente, solo reduce la frecuencia con la que hace falta.
- **Vigila `ActiveMessageCount` y la antigüedad, no solo los errores.** Un consumidor puede estar sano según los logs y aun así ir más lento de lo que se publica. La métrica que detecta eso es la profundidad de cola creciendo de forma sostenida, o mejor, la antigüedad del mensaje más viejo. Añade una alerta sobre `DeadletteredMessageCount`: una DLQ que crece sin que nadie mire es indistinguible de perder mensajes.
- **Cierra el procesador de forma ordenada al parar la aplicación.** Un `StopProcessingAsync()` en el apagado deja terminar los mensajes en curso antes de soltar la conexión. Sin él, cada despliegue abandona los mensajes a medio procesar y los reentrega, con lo que el trabajo duplicado pasa a ser rutina en cada actualización.
- **No pongas datos grandes en el mensaje.** El límite de 256 KB de Standard llega antes de lo que parece con un PDF o una imagen en base64, y el error aparece en producción con el pedido grande de un cliente concreto. El patrón correcto (*claim check*) es subir el fichero a Blob Storage y enviar solo la URL: el mensaje queda pequeño y el coste por operación baja.

## Recursos didácticos

- [Service Bus Explorer](https://github.com/paolosalvatori/ServiceBusExplorer) — herramienta de escritorio para inspeccionar colas y topics, mirar mensajes sin consumirlos, reenviar desde la DLQ y probar filtros. Es prácticamente imprescindible para operar Service Bus y hace visible todo lo de esta ficha.
- [Azure Service Bus emulator](https://learn.microsoft.com/azure/service-bus-messaging/overview-emulator) — un emulador local en contenedor para desarrollar y hacer pruebas sin gastar en un espacio de nombres real.
- [Documentación de filtros SQL](https://learn.microsoft.com/azure/service-bus-messaging/topic-filters) — la sintaxis completa de las reglas de suscripción, incluidos los filtros de correlación, que son más rápidos que los SQL cuando basta con comparar por igualdad.

---

*En resumen: Azure Service Bus da las mismas garantías que un broker propio —colas, topics, entrega fiable— más sesiones, filtros y dead-lettering de serie, a cambio de pagar por operación y atarte a Azure.*
