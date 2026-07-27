# Dead Letter Queues

## ¿Qué es?

Una *dead-letter queue* (cola de mensajes muertos, **DLQ**) es una cola aparte donde el broker aparta automáticamente los mensajes que no se han podido procesar, en lugar de perderlos o reintentarlos para siempre.

## ¿Por qué existe?

Sin DLQ, un mensaje "veneno" —con un dato corrupto, un formato inesperado, un caso que el consumidor no contempla— entra en un bucle: el consumidor falla, el broker reintenta, vuelve a fallar. Indefinidamente.

Ese bucle no solo desperdicia recursos. Hace dos cosas peores:

- **Bloquea lo que viene detrás** si el consumo respeta el orden. Un mensaje imposible de procesar detiene toda la cola.
- **Esconde el problema.** Los logs se llenan del mismo error miles de veces y las alertas por tasa de error se vuelven ruido que nadie mira.

La DLQ corta el bucle: tras un número máximo de intentos, el broker mueve el mensaje a una cola separada donde espera, sin bloquear nada, hasta que alguien lo revise.

> Piensa en la bandeja de "correo no entregado" de una oficina postal: si una carta no se puede entregar tras varios intentos, no se destruye ni se sigue intentando eternamente; se aparta a un sitio donde alguien la revisa a mano.

## ¿Cuándo y para qué se usa?

En cualquier cola de producción, sin excepciones. Para que un pedido con un dato inesperado no bloquee el procesamiento del resto, y para poder investigar después, con calma, por qué falló un envío concreto sin haberlo perdido.

La DLQ es transversal: aplica igual a [RabbitMQ](RabbitMQ.md) y a [Azure Service Bus](Azure-Service-Bus.md), aunque cada uno la configure a su manera.

---

## Por qué acaba un mensaje en la DLQ

No todo llega ahí por el mismo motivo, y saber cuál fue cambia por completo qué hacer después.

| Motivo | Qué pasó | ¿Reprocesar? |
|---|---|---|
| **Máximo de entregas** | Falló N veces seguidas | Sí, si arreglas la causa |
| **Caducidad (TTL)** | Nadie lo consumió a tiempo | Normalmente no: ya no aplica |
| **Rechazo explícito** | El consumidor decidió apartarlo | Depende del motivo que registró |
| **Error de cabeceras o tamaño** | El broker no pudo procesarlo | No: hay que corregir al productor |

La distinción entre las dos primeras es la que más se pasa por alto. Un mensaje que agotó reintentos indica **un problema en el consumidor**: reprocesarlo tras arreglar el bug es lo correcto. Un mensaje caducado indica que **nadie estaba consumiendo**, y reenviarlo puede ser peor que descartarlo: un recordatorio de carrito abandonado de hace tres semanas no debería enviarse hoy.

## Configurarla en Azure Service Bus

Aquí es casi automática: **toda cola y toda suscripción nacen con su DLQ**, accesible como una subcola. Lo que se configura es cuándo se usa.

```bash
az servicebus queue create \
  --resource-group tienda-rg --namespace-name tienda-bus \
  --name facturacion-pedidos \
  --max-delivery-count 5 \
  --default-message-time-to-live PT24H \
  --enable-dead-lettering-on-message-expiration true
```

- **`--max-delivery-count 5`** — tras cinco entregas fallidas, a la DLQ. El valor por defecto es 10, que suele ser demasiado: si algo falla cinco veces seguidas, la sexta no va a cambiar nada y solo retrasa el diagnóstico.
- **`--default-message-time-to-live PT24H`** — el mensaje caduca a las 24 horas (formato ISO 8601).
- **`--enable-dead-lettering-on-message-expiration`** — sin esto, los mensajes caducados **se borran en silencio**. Actívalo siempre: enterarte de que algo caducó es información valiosa.

Desde el código, el consumidor también puede apartar un mensaje sin agotar reintentos, cuando sabe que reintentarlo no servirá:

```csharp
catch (JsonException ex)
{
    await args.DeadLetterMessageAsync(
        args.Message,
        deadLetterReason: "PayloadInvalido",
        deadLetterErrorDescription: ex.Message);
}
```

Rellenar `deadLetterReason` con un código estable y corto es lo que después permite agrupar y contar los mensajes de la DLQ por causa. Sin él, la investigación consiste en abrir mensajes uno a uno.

Para leer la DLQ, el nombre es el de la cola más el sufijo `/$deadletterqueue`:

```csharp
var dlqReceiver = client.CreateReceiver(
    "facturacion-pedidos",
    new ServiceBusReceiverOptions { SubQueue = SubQueue.DeadLetter });

var mensajes = await dlqReceiver.PeekMessagesAsync(maxMessages: 20);

foreach (var m in mensajes)
{
    logger.LogWarning("DLQ {Id}: {Razon} — {Detalle}",
        m.MessageId,
        m.DeadLetterReason,
        m.DeadLetterErrorDescription);
}
```

`PeekMessagesAsync` **mira sin consumir**: no bloquea los mensajes ni gasta entregas, así que es seguro ejecutarlo para investigar. `ReceiveMessagesAsync`, en cambio, los bloquea y empieza a consumirlos.

## Configurarla en RabbitMQ

RabbitMQ no trae DLQ de serie: hay que construirla con las piezas normales del broker. El "dead letter exchange" es un exchange corriente al que se redirigen los mensajes descartados.

```csharp
// 1. El exchange y la cola donde acabarán los mensajes muertos
await channel.ExchangeDeclareAsync("pedidos.dlx", ExchangeType.Fanout, durable: true);
await channel.QueueDeclareAsync("facturacion.dlq", durable: true, exclusive: false, autoDelete: false);
await channel.QueueBindAsync("facturacion.dlq", "pedidos.dlx", routingKey: "");

// 2. La cola principal, apuntando a ese exchange
var args = new Dictionary<string, object?>
{
    ["x-dead-letter-exchange"] = "pedidos.dlx",
    ["x-message-ttl"] = 86_400_000,      // 24 h en milisegundos
    ["x-max-length"] = 100_000           // tope de seguridad
};

await channel.QueueDeclareAsync(
    queue: "facturacion.pedidos-confirmados",
    durable: true, exclusive: false, autoDelete: false,
    arguments: args);
```

Un mensaje llega a `pedidos.dlx` en tres situaciones: cuando se rechaza con `requeue: false`, cuando caduca por el TTL, o cuando la cola supera `x-max-length`.

Y aquí está la diferencia importante con Service Bus: **RabbitMQ no cuenta los intentos por sí solo**. No existe un `max-delivery-count`. Si el consumidor hace `BasicNackAsync(requeue: true)` una y otra vez, el mensaje vuelve a la cola indefinidamente sin llegar nunca a la DLQ.

El contador hay que llevarlo en el consumidor, leyendo la cabecera `x-death` que RabbitMQ añade cada vez que un mensaje pasa por el dead-lettering:

```csharp
var intentos = ObtenerConteoMuertes(ea.BasicProperties.Headers);

if (intentos >= 5)
{
    // Agotados: descartar definitivamente (va al DLX)
    await channel.BasicRejectAsync(ea.DeliveryTag, requeue: false);
    return;
}
```

Recuerda además que estos argumentos hay que ponerlos **al crear la cola**: añadirlos después choca con el `PRECONDITION_FAILED` que se explica en la ficha de [RabbitMQ](RabbitMQ.md) y obliga a borrar y recrear la cola con lo que tenga dentro.

## Reintentos con espera antes de la DLQ

Reintentar cinco veces seguidas en dos segundos no ayuda cuando la causa es una base de datos que tarda un minuto en volver: agotas los intentos durante el peor momento y mandas a la DLQ mensajes que habrían funcionado perfectamente medio minuto después.

Lo que quieres es esperar cada vez más entre intentos (*backoff exponencial*): 5 s, 30 s, 2 min, 10 min. En Service Bus, la forma directa es reprogramar el mensaje en lugar de abandonarlo:

```csharp
catch (HttpRequestException)   // fallo transitorio de un servicio externo
{
    var intentos = args.Message.DeliveryCount;

    if (intentos >= 5)
    {
        await args.DeadLetterMessageAsync(args.Message, "ReintentosAgotados");
        return;
    }

    var espera = TimeSpan.FromSeconds(Math.Pow(5, intentos));   // 5 s, 25 s, 125 s...
    await sender.ScheduleMessageAsync(
        new ServiceBusMessage(args.Message), DateTimeOffset.UtcNow.Add(espera));

    await args.CompleteMessageAsync(args.Message);
}
```

En RabbitMQ el equivalente es una **cola de espera**: una cola sin consumidores, con un TTL, cuyo dead-letter exchange apunta de vuelta a la cola original. El mensaje "descansa" ahí hasta caducar y entonces vuelve solo. Es un truco muy usado y conviene reconocerlo cuando lo veas en una topología ajena.

Y algo que no debe perderse de vista: **el backoff es para errores transitorios**. Un JSON malformado no mejora esperando diez minutos, así que ese va directo a la DLQ en el primer intento. Distinguir los dos tipos de error es lo que hace útil todo este mecanismo.

## Investigar y reprocesar

Un mensaje en la DLQ necesita tres pasos: entender por qué llegó, arreglar la causa y devolverlo.

**1. Entender.** Las propiedades de dead-lettering dicen casi todo:

```
MessageId:                  pedido-confirmado-4711
DeadLetterReason:           MaxDeliveryCountExceeded
DeadLetterErrorDescription: Object reference not set to an instance of an object
EnqueuedTime:               2026-07-20T03:14:22Z
DeliveryCount:              5
```

**2. Arreglar.** Si es un bug del consumidor, corregirlo y desplegar. Si es un dato corrupto, corregirlo en origen.

**3. Devolver.** Reprocesar es leer de la DLQ y publicar de nuevo en la cola original:

```csharp
var dlq = client.CreateReceiver("facturacion-pedidos",
    new ServiceBusReceiverOptions { SubQueue = SubQueue.DeadLetter });
var sender = client.CreateSender("facturacion-pedidos");

foreach (var m in await dlq.ReceiveMessagesAsync(maxMessages: 100))
{
    await sender.SendMessageAsync(new ServiceBusMessage(m));   // copia limpia
    await dlq.CompleteMessageAsync(m);                         // sacarlo de la DLQ
}
```

El `new ServiceBusMessage(m)` crea una copia con el contador de entregas a cero. Y el orden importa: **primero se reenvía y después se saca de la DLQ**. Al revés, un fallo entre las dos operaciones pierde el mensaje justo cuando estabas intentando recuperarlo.

Antes de lanzar esto contra cien mensajes, prueba con uno. Reprocesar en masa sin haber corregido la causa devuelve los cien a la DLQ y, de paso, satura al consumidor.

## Vigilar la DLQ

Una DLQ que nadie mira es indistinguible de perder mensajes en silencio: peor aún, da la falsa sensación de que hay una red de seguridad.

Lo mínimo son dos alertas:

- **Hay algo en la DLQ.** No "hay más de cien": con uno ya hay algo que investigar. En Azure, una alerta sobre la métrica `DeadletteredMessages` con umbral mayor que cero.
- **La DLQ crece.** Un goteo constante indica un problema sistemático, no un caso raro.

Desde la terminal, la comprobación rápida en cada broker:

```bash
# Azure Service Bus
az servicebus queue show -g tienda-rg --namespace-name tienda-bus \
  -n facturacion-pedidos --query "countDetails.deadLetterMessageCount"

# RabbitMQ
docker exec rabbitmq rabbitmqctl list_queues name messages | grep dlq
```

```
facturacion.dlq    7
```

Un buen hábito de operación es revisar las DLQ como parte de la rutina semanal, igual que se revisan los logs de error. Los mensajes que llevan meses ahí suelen contar la historia de un bug que nadie llegó a diagnosticar.

## Buenas prácticas avanzadas

- **Guarda el motivo en un código estable, no en el texto de la excepción.** `DeadLetterReason = "PayloadInvalido"` permite agrupar y contar; `DeadLetterReason = ex.Message` produce mil variantes distintas del mismo problema y hace imposible saber si son un caso o doscientos. El texto detallado va en `DeadLetterErrorDescription`, que es donde tiene sentido.
- **Baja `MaxDeliveryCount` a 3–5.** El 10 por defecto de Service Bus multiplica por diez el trabajo inútil de un mensaje condenado y retrasa mucho su llegada a la DLQ, que es donde de verdad quieres verlo. Con reintentos escalonados, tres o cinco intentos cubren cualquier fallo transitorio razonable; más allá, el problema no es transitorio.
- **Ten una DLQ por consumidor, no una compartida.** Con una DLQ común para varias colas, un incidente en un servicio contamina las alertas de todos y no se puede reprocesar el trabajo de uno sin tocar el del resto. Separadas, cada equipo ve y arregla lo suyo, y la alerta señala directamente al responsable.
- **Pon TTL en la propia DLQ, con destino a almacenamiento frío.** Suena contradictorio, pero una DLQ sin caducidad acumula mensajes durante años y en algunos brokers cuenta para los límites de la entidad. Un TTL largo (30–90 días) que archive a un blob antes de borrar conserva el rastro para auditar sin que la cola crezca sin fin.
- **Mide cuánto tarda un mensaje en llegar a la DLQ, no solo cuántos hay.** Con reintentos escalonados generosos, un mensaje puede tardar horas en aparecer ahí — y durante todo ese rato el problema existe y nadie lo ve. Si el tiempo entre el primer fallo y la entrada en la DLQ supera tu objetivo de detección, el backoff es demasiado indulgente y hay que recortarlo.

## Recursos didácticos

- [Service Bus Explorer](https://github.com/paolosalvatori/ServiceBusExplorer) — inspeccionar la DLQ, leer los motivos de cada mensaje y reenviarlos a la cola original con un par de clics. Es la herramienta que convierte la operación de una DLQ en algo llevadero.
- [RabbitMQ Simulator](http://tryrabbitmq.com/) — montar un dead-letter exchange visualmente y ver cómo un mensaje rechazado cambia de cola aclara el mecanismo mucho más rápido que la documentación.
- [Documentación de dead-lettering de Azure](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-dead-letter-queues) — la lista completa de motivos de dead-lettering del sistema, útil para interpretar un `DeadLetterReason` que no hayas puesto tú.

---

*En resumen: la dead-letter queue aparta lo que no se puede procesar para que un mensaje problemático no bloquee ni pierda el resto — pero una DLQ que nadie vigila es solo una forma más lenta de perder mensajes.*
