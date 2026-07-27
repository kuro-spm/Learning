# RabbitMQ

## ¿Qué es?

RabbitMQ es un *message broker* de código abierto: recibe mensajes de los productores, los enruta a colas según reglas configurables y los entrega a los consumidores suscritos.

## ¿Por qué existe?

Implementar mensajería a mano —una tabla de mensajes pendientes y consumidores que la sondean— parece sencillo hasta que aparecen las preguntas difíciles: ¿qué pasa si dos consumidores cogen el mismo mensaje a la vez? ¿Cómo se reintenta lo que falló sin reintentar lo que funcionó? ¿Cómo se reparte el trabajo entre cinco procesos sin que uno se lleve todo?

RabbitMQ es software especializado en resolver exactamente eso, con las garantías de bajo nivel ya construidas y probadas. Su rasgo distintivo frente a otros brokers es el **enrutado**: el productor no elige la cola de destino, solo describe el mensaje, y unas reglas declarativas deciden a qué colas llega.

> Piensa en RabbitMQ como el servicio de correos de un edificio de oficinas: hay un cartero que recibe la correspondencia, la clasifica según a qué buzón va y la reparte. Ni el remitente ni el destinatario se preocupan de cómo llega físicamente la carta.

## ¿Cuándo y para qué se usa?

Cuando quieres mensajería en tu propia infraestructura —un contenedor Docker, un servidor propio, tu nube— con control fino sobre el enrutado y sin atarte a un proveedor. Es la opción por defecto para reparto de trabajo entre procesos y para difundir eventos a varios servicios.

Frente a [Azure Service Bus](Azure-Service-Bus.md), la diferencia no está en las capacidades sino en quién lo opera: aquí lo instalas, actualizas y vigilas tú. Frente a Kafka, RabbitMQ está pensado para mensajes que se consumen y desaparecen, no para un registro de eventos que se relee.

---

## Levantarlo en local

Antes de escribir código, ten un broker delante. La imagen con el sufijo `-management` incluye una interfaz web que hace visible todo lo que explica esta ficha:

```bash
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:4-management
```

Los dos puertos tienen cometidos distintos: **5672** es por donde hablan las aplicaciones (protocolo AMQP) y **15672** es la interfaz web. Abre `http://localhost:15672` y entra con `guest` / `guest`.

Esa interfaz es la mejor herramienta de aprendizaje que tiene RabbitMQ. En la pestaña *Exchanges* puedes publicar un mensaje a mano y ver en *Queues* dónde ha caído, sin escribir una línea de código. Úsala mientras lees el resto.

> El usuario `guest` solo funciona conectando desde la propia máquina. Si intentas usarlo desde otro host, la conexión se rechaza sin explicación clara. En cualquier despliegue real hay que crear un usuario propio.

Y en el proyecto .NET:

```bash
dotnet add package RabbitMQ.Client
```

Todo el código de esta ficha usa la **versión 7.x**, cuya API es asíncrona. Si estás en la 6.x los métodos son los mismos pero síncronos y sin el sufijo `Async` (`channel.BasicPublish` en lugar de `channel.BasicPublishAsync`), y el consumidor es `EventingBasicConsumer` en vez de `AsyncEventingBasicConsumer`. Es el motivo de la mitad de los ejemplos de internet que no compilan.

## El modelo: por qué no publicas en una cola

Esta es la idea central de RabbitMQ y la que lo diferencia del resto. **El productor nunca publica en una cola.** Publica en un **exchange**, y el exchange decide a qué colas va el mensaje según los *bindings* configurados.

```
productor → exchange → (binding) → cola → consumidor
                     ↘ (binding) → cola → consumidor
```

Las cuatro piezas:

| Pieza | Qué es |
|---|---|
| **Exchange** | El punto de entrada. Recibe y decide a dónde reparte. |
| **Routing key** | Una etiqueta de texto que el productor pone al mensaje. |
| **Binding** | La regla que une un exchange con una cola, normalmente con un patrón. |
| **Cola** | Donde el mensaje espera a que alguien lo consuma. |

La ventaja de esta indirección: para que un servicio nuevo empiece a recibir eventos de pedidos, se crea su cola y su binding. **El productor no se toca.** Es el mismo desacoplamiento que aporta un topic, pero con reglas de reparto mucho más expresivas.

## Los tipos de exchange

Cómo decide un exchange a dónde va el mensaje depende de su tipo. Son cuatro, y en la práctica se usan dos.

**Direct** — entrega a las colas cuyo binding coincide **exactamente** con la routing key.

```
routing key "pedido.confirmado" → cola con binding "pedido.confirmado"  ✅
                                → cola con binding "pedido.cancelado"   ❌
```

**Fanout** — entrega a **todas** las colas enlazadas, ignorando la routing key. Es el pub/sub puro: el equivalente exacto de un topic de Service Bus.

**Topic** — entrega según un patrón, con dos comodines sobre las palabras separadas por puntos: `*` sustituye una palabra y `#` sustituye cero o más.

```
Binding "pedido.*"        → recibe pedido.confirmado, pedido.cancelado
                          → NO recibe pedido.linea.añadida (dos palabras tras "pedido")
Binding "pedido.#"        → recibe los tres
Binding "*.confirmado"    → recibe pedido.confirmado, pago.confirmado
```

**Headers** — enruta por las cabeceras del mensaje en lugar de por la routing key. Es más flexible y bastante más lento; se usa poco.

En la práctica: **topic** para eventos (cubre los casos de direct y fanout, y deja margen para afinar después) y **direct** cuando cada mensaje va a un único destino conocido.

## Declarar la topología

Exchanges, colas y bindings se declaran desde el código al arrancar. Las declaraciones son idempotentes: si ya existen con la misma configuración, no pasa nada, así que es normal que cada servicio declare lo que necesita cada vez que arranca.

```csharp
var factory = new ConnectionFactory { HostName = "localhost" };
await using var connection = await factory.CreateConnectionAsync();
await using var channel = await connection.CreateChannelAsync();

// El exchange donde el servicio de pedidos publica sus eventos
await channel.ExchangeDeclareAsync(
    exchange: "pedidos.eventos",
    type: ExchangeType.Topic,
    durable: true);

// La cola de facturación y su regla de enrutado
await channel.QueueDeclareAsync(
    queue: "facturacion.pedidos-confirmados",
    durable: true,
    exclusive: false,
    autoDelete: false);

await channel.QueueBindAsync(
    queue: "facturacion.pedidos-confirmados",
    exchange: "pedidos.eventos",
    routingKey: "pedido.confirmado");
```

Qué hace cada parámetro que importa:

- **`durable: true`** en el exchange y en la cola significa que sobreviven a un reinicio del broker. Con `false` desaparecen, y con ellas todo lo que hubiera dentro. Es el valor por defecto en muchos ejemplos rápidos y una fuente clásica de pérdida de mensajes.
- **`exclusive: false`** permite que varias conexiones usen la cola. Con `true`, la cola pertenece a la conexión que la creó y se borra al cerrarla.
- **`autoDelete: false`** evita que la cola se borre cuando el último consumidor se desconecta. Con `true`, un despliegue que reinicia todos los consumidores a la vez se lleva la cola por delante.

Ojo con un detalle que cuesta encontrar: si declaras una cola que ya existe **con parámetros distintos**, RabbitMQ no la modifica, sino que cierra el canal con un error:

```
PRECONDITION_FAILED - inequivalent arg 'durable' for queue 'facturacion.pedidos-confirmados'
```

Cambiar la configuración de una cola existente obliga a borrarla y volver a crearla, con lo que haya dentro.

## Publicar

```csharp
var evento = new PedidoConfirmado { PedidoId = 4711, Total = 149.90m };
var body = JsonSerializer.SerializeToUtf8Bytes(evento);

var props = new BasicProperties
{
    Persistent = true,                              // sobrevive al reinicio del broker
    MessageId = Guid.NewGuid().ToString(),          // para que el consumidor detecte duplicados
    ContentType = "application/json"
};

await channel.BasicPublishAsync(
    exchange: "pedidos.eventos",
    routingKey: "pedido.confirmado",
    mandatory: false,
    basicProperties: props,
    body: body);
```

Tres cosas que casi nunca están en los ejemplos y sí deben estar en producción:

- **`Persistent = true`** escribe el mensaje en disco. Sin esto, una cola duradera con mensajes no persistentes pierde su contenido al reiniciar el broker: **hacen falta las dos cosas**, cola duradera y mensaje persistente.
- **`MessageId`** es lo que permite al consumidor saber que ya procesó ese mensaje. Sin un identificador estable, la idempotencia del consumidor no tiene de dónde agarrarse.
- **`mandatory: true`** hace que el broker devuelva el mensaje si ninguna cola coincide con la routing key. Con `false`, un mensaje que no encaja en ningún binding **se descarta en silencio**. Es el fallo más desconcertante de RabbitMQ: publicas sin error y el mensaje no aparece en ninguna parte.

### Confirmar que el broker lo recibió

`BasicPublishAsync` termina cuando el mensaje sale por el socket, no cuando el broker lo ha guardado. Si el broker se cae en ese instante, el mensaje se pierde sin que tu código se entere. Para cerrar ese hueco están las *publisher confirms*, que se activan al crear el canal:

```csharp
await using var channel = await connection.CreateChannelAsync(
    new CreateChannelOptions(
        publisherConfirmationsEnabled: true,
        publisherConfirmationTrackingEnabled: true));

await channel.BasicPublishAsync(/* ... */);   // ahora espera el ack del broker
```

Con esto activado, la llamada no termina hasta que el broker confirma. Cuesta latencia y es lo que quieres para eventos que no se pueden perder.

## Consumir

```csharp
await channel.BasicQosAsync(prefetchSize: 0, prefetchCount: 10, global: false);

var consumer = new AsyncEventingBasicConsumer(channel);

consumer.ReceivedAsync += async (sender, ea) =>
{
    try
    {
        var evento = JsonSerializer.Deserialize<PedidoConfirmado>(ea.Body.Span);
        await GenerarFacturaAsync(evento!.PedidoId);

        await channel.BasicAckAsync(ea.DeliveryTag, multiple: false);
    }
    catch (JsonException)
    {
        // El mensaje está corrupto: reintentarlo no lo va a arreglar
        await channel.BasicRejectAsync(ea.DeliveryTag, requeue: false);
    }
    catch (Exception)
    {
        // Fallo transitorio: que vuelva a la cola
        await channel.BasicNackAsync(ea.DeliveryTag, multiple: false, requeue: true);
    }
};

await channel.BasicConsumeAsync(
    queue: "facturacion.pedidos-confirmados",
    autoAck: false,
    consumer: consumer);
```

Las decisiones de este bloque:

- **`autoAck: false`** es obligatorio en cualquier consumidor serio. Con `autoAck: true` el mensaje se borra **en el momento de entregarse**, antes de que tu código lo toque: si el proceso muere procesándolo, se pierde sin rastro.
- **`BasicAckAsync`** confirma el proceso correcto y borra el mensaje.
- **`BasicRejectAsync(requeue: false)`** descarta el mensaje o lo manda a la [dead-letter queue](Dead-Letter-Queues.md) si hay una configurada. Es lo correcto para un error que va a repetirse siempre.
- **`BasicNackAsync(requeue: true)`** lo devuelve a la cola para reintentar. Cuidado: sin límite, un fallo permanente aquí crea un bucle infinito que satura el consumidor. La configuración de dead-lettering es lo que pone el freno.

Distinguir los dos tipos de error —el `catch (JsonException)` frente al `catch (Exception)`— es lo que evita que un mensaje corrupto se reintente eternamente mientras un fallo de red se descarta por error.

### El prefetch

`BasicQosAsync` con `prefetchCount: 10` limita cuántos mensajes sin confirmar puede tener un consumidor a la vez. Es una línea fácil de omitir y con consecuencias muy visibles.

Sin ella, el broker entrega **todo lo que haya en la cola** al primer consumidor que se conecte. Con tres consumidores y 10 000 mensajes encolados, el primero se lleva los 10 000 y los otros dos se quedan sin trabajo:

| `prefetchCount` | Efecto |
|---|---|
| Sin límite | Un consumidor acapara toda la cola; los demás ociosos |
| 1 | Reparto perfectamente equitativo, más viajes de red |
| 10–100 | El equilibrio habitual |

La regla práctica: bajo si cada mensaje tarda mucho (así el reparto es justo), alto si son rápidos (así no se pierde tiempo en la red).

## Dead-lettering

RabbitMQ no aparta solo los mensajes problemáticos: hay que decírselo al declarar la cola, mediante argumentos.

```csharp
await channel.ExchangeDeclareAsync("pedidos.dlx", ExchangeType.Fanout, durable: true);
await channel.QueueDeclareAsync("facturacion.dlq", durable: true, exclusive: false, autoDelete: false);
await channel.QueueBindAsync("facturacion.dlq", "pedidos.dlx", routingKey: "");

var args = new Dictionary<string, object?>
{
    ["x-dead-letter-exchange"] = "pedidos.dlx",
    ["x-message-ttl"] = 86_400_000        // 24 h en milisegundos
};

await channel.QueueDeclareAsync(
    queue: "facturacion.pedidos-confirmados",
    durable: true, exclusive: false, autoDelete: false,
    arguments: args);
```

Con esto, un mensaje rechazado con `requeue: false` —o que caduque— va a `pedidos.dlx` en lugar de desaparecer. El tratamiento completo está en [Dead Letter Queues](Dead-Letter-Queues.md).

Configúralo **al crear la cola**. Añadir estos argumentos a una cola existente choca con el `PRECONDITION_FAILED` de antes y obliga a recrearla, justo cuando ya tienes mensajes atascados y menos ganas de borrar nada.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| El mensaje se publica sin error y no llega a ninguna cola | Ninguna binding coincide con la routing key; publica con `mandatory: true` para detectarlo |
| Todo se pierde al reiniciar el broker | Falta `durable: true` en la cola, o `Persistent = true` en el mensaje, o ambos |
| Un consumidor trabaja y los demás están parados | Falta `BasicQosAsync` |
| `PRECONDITION_FAILED - inequivalent arg` | Se declaró una cola existente con parámetros distintos |
| `ACCESS_REFUSED` conectando desde otra máquina | El usuario `guest` solo vale desde localhost |
| El consumidor procesa el mismo mensaje sin parar | `BasicNackAsync(requeue: true)` sobre un error permanente, sin dead-lettering |

Para diagnosticar, la interfaz web contesta casi todo. Y desde la terminal:

```bash
docker exec rabbitmq rabbitmqctl list_queues name messages consumers
```

```
name                                messages  consumers
facturacion.pedidos-confirmados     0         3
inventario.pedidos-confirmados      1842      0        ← nadie consume
facturacion.dlq                     7         0        ← revisar esto
```

Esa salida diagnostica de un vistazo: `consumers` a cero con mensajes acumulándose es un consumidor caído o mal enlazado.

## Buenas prácticas avanzadas

- **Comparte la conexión y abre un canal por hilo.** Una conexión TCP es cara y un canal es barato: el patrón correcto es una conexión por aplicación y un canal por cada hilo o tarea concurrente. Compartir un canal entre hilos es la causa de errores intermitentes e irreproducibles, porque `IChannel` no es seguro para uso concurrente. Y abrir una conexión por mensaje —que se ve— agota los descriptores de fichero del broker bajo carga.
- **Usa colas *quorum* en producción, no las clásicas.** `x-queue-type: quorum` replica la cola entre los nodos del clúster con un algoritmo de consenso, de modo que la caída de un nodo no se lleva los mensajes. Las colas clásicas espejadas quedaron obsoletas por sus problemas de división de red, y las no replicadas simplemente pierden lo que tuvieran. El coste es algo más de latencia y de disco.
- **Pon un límite de longitud a las colas que puedan crecer sin control.** `x-max-length` o `x-max-length-bytes` hacen que la cola descarte lo más antiguo —o lo mande a la DLQ— al llegar al límite. Sin eso, un consumidor caído durante el fin de semana llena la memoria del broker y **tumba todas las demás colas**, incluidas las que funcionaban. Perder los mensajes de una cola es mejor que perder el broker entero.
- **No lo uses como almacén de datos.** Es tentador dejar mensajes sin consumir "por si acaso" o leer la cola para consultar estado. RabbitMQ está optimizado para tránsito: las colas largas degradan su rendimiento y, cuando el contenido no cabe en memoria, la paginación a disco ralentiza todo lo que pase por el broker. Si necesitas releer eventos pasados, la herramienta es un registro tipo Kafka, no una cola.
- **Nombra las colas por el consumidor, no por el evento.** `facturacion.pedidos-confirmados` dice quién lee y qué; `pedidos-confirmados` a secas invita a que tres servicios se enganchen a la misma cola y se roben los mensajes entre ellos —cada uno procesaría un tercio— en lugar de tener una cola cada uno. Es un error de diseño que la convención de nombres previene sola.

## Recursos didácticos

- [RabbitMQ Simulator](http://tryrabbitmq.com/) — arrastras exchanges, colas y bindings y ves los mensajes moverse en tiempo real según las routing keys. Es la mejor forma de entender los comodines `*` y `#` de los exchanges topic sin escribir código.
- [La interfaz de gestión en `localhost:15672`](https://www.rabbitmq.com/docs/management) — publicar un mensaje a mano desde la pestaña *Exchanges* y ver en qué colas cae enseña más sobre enrutado que cualquier explicación.
- [Tutoriales oficiales de RabbitMQ](https://www.rabbitmq.com/tutorials) — seis tutoriales incrementales (de "hola mundo" a RPC) con el código en .NET, Java, Python y más. Están bien hechos y son cortos.

---

*En resumen: en RabbitMQ el productor no elige la cola, describe el mensaje y el exchange decide — a cambio de declarar la topología a mano, ganas que añadir un consumidor nuevo no toque una línea del productor.*
