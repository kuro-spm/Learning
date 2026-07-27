# Mensajería Asíncrona — Guía de tecnologías

Cómo desacoplar servicios con colas y eventos en lugar de llamadas directas: por qué hacerlo, qué se gana y qué se pierde, cómo funcionan los dos brokers más habituales y qué piezas hacen falta para que el sistema sea fiable de verdad.

Está escrita para perfiles backend junior-medio con experiencia en APIs REST pero sin experiencia previa en sistemas de mensajería. No presupone nada sobre brokers: cada concepto se explica desde el problema que resuelve, con código ejecutable y la salida que devuelve.

Todas las fichas usan el mismo ejemplo —una tienda online donde confirmar un pedido dispara la facturación, el ajuste de inventario y un email de confirmación— para que las piezas encajen entre documentos.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Cada bloque se apoya en el anterior.

### 1. El concepto

Entiende la idea antes de mirar herramientas. Esta ficha define el vocabulario que usan todas las demás.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Mensajería Asíncrona](Mensajeria-Asincrona.md) | Cola frente a topic, el ciclo de vida de un mensaje, las garantías de entrega y por qué la idempotencia te toca construirla a ti. Empieza siempre por aquí. |

### 2. Los brokers concretos

Las plataformas que implementan esas ideas. Lee al menos una de las dos, según lo que uses.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [RabbitMQ](RabbitMQ.md) | El broker autogestionado más extendido. Su rasgo propio es el enrutado: exchanges, bindings y routing keys. |
| 3 | [Azure Service Bus](Azure-Service-Bus.md) | El equivalente gestionado en Azure. Mismas ideas, más sesiones, filtros y dead-lettering de serie. |

### 3. Piezas para que sea fiable

Conceptos transversales, sea cual sea el broker. Sin ellos, un sistema de mensajería pierde o duplica trabajo en producción.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 4 | [Outbox Pattern](Outbox-Pattern.md) | Cómo garantizar que el evento se publica siempre, sin quedar a medias con el dato que lo origina. |
| 5 | [Dead Letter Queues](Dead-Letter-Queues.md) | Qué hacer con los mensajes que no se pueden procesar, para que uno solo no bloquee la cola entera. |

---

## Índice completo por bloque

<details>
<summary>Ver todos los archivos</summary>

**El concepto**
- [Mensajería Asíncrona](Mensajeria-Asincrona.md)

**Brokers**
- [RabbitMQ](RabbitMQ.md)
- [Azure Service Bus](Azure-Service-Bus.md)

**Fiabilidad**
- [Outbox Pattern](Outbox-Pattern.md)
- [Dead Letter Queues](Dead-Letter-Queues.md)

</details>

---

## Las cuatro decisiones que aparecen siempre

Un resumen de las disyuntivas que se repiten en toda la colección, con dónde se desarrolla cada una.

| Decisión | La pregunta que la resuelve | Dónde |
|---|---|---|
| ¿Cola o topic? | ¿Pido que se haga algo, o cuento que ha pasado algo? | [Mensajería Asíncrona](Mensajeria-Asincrona.md) |
| ¿Cómo evito procesar dos veces? | ¿Es la operación repetible, o necesito registrar lo ya hecho? | [Mensajería Asíncrona](Mensajeria-Asincrona.md) |
| ¿Cómo evito perder el evento? | ¿Puede el proceso morir entre guardar y publicar? | [Outbox Pattern](Outbox-Pattern.md) |
| ¿Qué hago con lo que falla? | ¿Es un error transitorio o va a fallar siempre? | [Dead Letter Queues](Dead-Letter-Queues.md) |

## Comprobaciones rápidas

Los comandos que responden "¿qué está pasando en mis colas?" en cada broker.

| Qué quieres saber | RabbitMQ | Azure Service Bus |
|---|---|---|
| Cuántos mensajes esperan | `rabbitmqctl list_queues name messages` | `az servicebus queue show ... --query "countDetails.activeMessageCount"` |
| Si hay consumidores conectados | `rabbitmqctl list_queues name consumers` | Portal → *Metrics* → `ActiveConnections` |
| Cuántos hay en la DLQ | `rabbitmqctl list_queues name messages \| grep dlq` | `az servicebus queue show ... --query "countDetails.deadLetterMessageCount"` |
| Inspeccionar un mensaje sin consumirlo | Interfaz web → *Queues* → *Get messages* | `PeekMessagesAsync`, o Service Bus Explorer |

---

> Piezas relacionadas en otras colecciones: [Observabilidad](../observabilidad/README.md) para reconstruir el recorrido de un mensaje entre servicios (sin trazas, depurar mensajería es adivinar), [APIs dirigidas por eventos](../../arquitectura-de-software/tipos-de-apis/APIs-Dirigidas-por-Eventos.md) para el estilo de arquitectura que se apoya en esto, [Domain Events](../../arquitectura-de-software/patrones-de-diseno/Domain-Events.md) para el caso en que el evento no sale del proceso y no hace falta broker, y [Despliegue en un VPS](../despliegue-en-vps/README.md) si el broker va a correr en un servidor propio.
