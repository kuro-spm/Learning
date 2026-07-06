# RabbitMQ

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

RabbitMQ es un *message broker* (intermediario de mensajería) de código abierto que recibe mensajes de los productores, los organiza en colas según reglas de enrutado, y los entrega a los consumidores que se suscriben a ellas.

## ¿Por qué existe?

Implementar mensajería asíncrona a mano —guardar mensajes pendientes en una tabla de base de datos, hacer que los consumidores hagan *polling* para ver si hay algo nuevo— es frágil y no escala bien: hay que resolver a mano problemas como qué pasa si dos consumidores cogen el mismo mensaje a la vez, o cómo reintentar si algo falla. RabbitMQ es software especializado en resolver exactamente eso: recibir, guardar temporalmente, enrutar y entregar mensajes de forma fiable, con las garantías de bajo nivel ya resueltas.

> Piensa en RabbitMQ como el servicio de correos de un edificio de oficinas: hay un cartero (RabbitMQ) que recibe la correspondencia, la clasifica según a qué buzón va (las colas) y la reparte. Ni el remitente ni el destinatario tienen que preocuparse de cómo llega físicamente la carta.

## ¿Cuándo y para qué se usa?

Cuando necesitas mensajería asíncrona autogestionada (en tu propia infraestructura, en un contenedor Docker o en un servidor propio), con control fino sobre cómo se enrutan los mensajes. Ejemplos: repartir el procesamiento de pedidos de una tienda online entre varios trabajadores; notificar a la vez a los servicios de facturación, inventario y envío de emails de que se ha confirmado un pedido, cada uno con su propia cola independiente aunque el mensaje se publique una sola vez.

## Lo mínimo que necesitas saber

**1. Los mensajes no van directos a una cola, pasan por un *exchange***

El productor nunca publica directamente en una cola: publica en un **exchange**, y es el exchange quien decide, según sus reglas de enrutado, a qué cola o colas llega el mensaje.

```csharp
channel.BasicPublish(
    exchange: "pedidos.eventos",
    routingKey: "pedido.confirmado",
    body: Encoding.UTF8.GetBytes(json));
```

**2. Tipos de exchange según cómo enrutan**

- **Direct**: entrega el mensaje a la cola cuya *routing key* coincide exactamente.
- **Fanout**: entrega el mensaje a *todas* las colas conectadas, ignorando la routing key. Es el equivalente a un pub/sub puro.
- **Topic**: entrega según un patrón de la routing key (`pedido.*` coincide con `pedido.confirmado` y `pedido.cancelado`).

**3. Las colas se declaran y se enlazan a un exchange**

```csharp
channel.QueueDeclare(queue: "facturacion.pedidos-confirmados", durable: true, exclusive: false, autoDelete: false);
channel.QueueBind(queue: "facturacion.pedidos-confirmados", exchange: "pedidos.eventos", routingKey: "pedido.confirmado");
```

**4. Consumir mensajes**

```csharp
var consumer = new EventingBasicConsumer(channel);
consumer.Received += (model, ea) =>
{
    var evento = JsonSerializer.Deserialize<PedidoConfirmado>(ea.Body.Span);
    GenerarFactura(evento.OrderId);
    channel.BasicAck(ea.DeliveryTag, multiple: false);
};
channel.BasicConsume(queue: "facturacion.pedidos-confirmados", autoAck: false, consumer: consumer);
```

**5. Confirmación manual (*ack*) frente a automática**

Con `autoAck: false`, el mensaje solo se elimina de la cola cuando el consumidor confirma explícitamente con `BasicAck`. Si el consumidor se cae antes de confirmar, RabbitMQ vuelve a entregar el mensaje.

## Lo que NO hace

- **No es un servicio gestionado en la nube** — a diferencia de Azure Service Bus, tú lo instalas, actualizas y escalas (aunque existen versiones gestionadas de terceros).
- **No garantiza el orden global de los mensajes** — dentro de una única cola con un solo consumidor sí se respeta el orden, pero en cuanto hay varios consumidores repartiéndose el trabajo, esa garantía desaparece.
- **No persiste los mensajes para siempre por defecto** — una cola no duradera (`durable: false`) se pierde si RabbitMQ se reinicia; hay que declararla explícitamente como duradera.
- **No procesa ni transforma el contenido del mensaje** — solo lo transporta; la lógica de qué hacer con él es responsabilidad del consumidor.

## Buenas prácticas avanzadas

- **Declara siempre colas y exchanges duraderos y mensajes persistentes en producción** — la combinación por defecto en muchos ejemplos rápidos (`durable: false`, sin marcar el mensaje como persistente) hace que un simple reinicio del broker borre todo lo que estuviera en cola. En producción, esa combinación debe ser una decisión consciente, no un olvido.
- **Configura un *prefetch count* acorde a la capacidad del consumidor** — sin límite, RabbitMQ puede entregarle a un único consumidor cientos de mensajes de golpe, saturando su memoria mientras otros consumidores están ociosos. `BasicQos` con un prefetch bajo reparte mejor la carga entre varios consumidores.
- **Usa un exchange *dead-letter* en cada cola crítica, no lo dejes para cuando aparezca el problema** — un mensaje "veneno" que hace fallar siempre al consumidor puede bloquear la cola en un bucle de reintentos infinito si no hay a dónde apartarlo. Configurar `x-dead-letter-exchange` desde el principio evita que un solo mensaje corrupto pare el resto del flujo.
- **No conviertas RabbitMQ en un sustituto de base de datos** — es tentador dejar mensajes sin consumir "por si acaso" o usarlo como almacén de estado, pero un broker está optimizado para tránsito, no para consultas ni para guardar datos a largo plazo; eso satura la memoria del broker y degrada el rendimiento de todo lo que pasa por él.

---

*En resumen: RabbitMQ es el cartero especializado que recibe mensajes, decide por dónde enrutarlos según reglas configurables (exchanges) y los entrega de forma fiable a quien esté suscrito, sin que productor y consumidor tengan que coordinarse directamente.*
