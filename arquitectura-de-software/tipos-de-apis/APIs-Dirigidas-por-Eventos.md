# APIs Dirigidas por Eventos

## ¿Qué es?

Una **API dirigida por eventos** (*event-driven*) es un estilo en el que los sistemas no se llaman directamente, sino que se comunican **publicando eventos** ("ha pasado esto") en un intermediario, y otros sistemas **se suscriben** para reaccionar cuando les interesa. La pieza central es un *message broker* (cola o bus de mensajes) que reparte esos eventos.

## ¿Por qué existe?

En el modelo clásico, el servicio A llama directamente al B y espera su respuesta. Si B está caído o lento, A se bloquea o falla. Y si mañana quieres que también reaccionen C y D, tienes que modificar A para que los llame a todos. Eso acopla los sistemas y los hace frágiles.

Las APIs dirigidas por eventos rompen ese acoplamiento: A solo anuncia "ha pasado X" y se desentiende. Quien quiera enterarse, se suscribe. A no sabe ni le importa quién escucha.

> Piensa en un tablón de anuncios de la comunidad. Quien tiene una novedad la cuelga (publica) y sigue con su vida. Los vecinos interesados la leen cuando pasan (se suscriben). Nadie tiene que ir puerta por puerta avisando a todos.

## ¿Cuándo y para qué se usa?

Cuando un mismo hecho desencadena varias reacciones independientes, o cuando quieres que los sistemas no dependan unos de otros en tiempo real. Ejemplo típico: en una tienda online, al confirmarse un pedido se publica el evento `OrderPlaced`. A partir de ahí, sin que el servicio de pedidos sepa nada, reaccionan por su cuenta el servicio de facturación, el de inventario, el de envío de emails y el de estadísticas. Tecnologías habituales: colas y *brokers* como RabbitMQ, Apache Kafka o los servicios de mensajería de la nube.

## Lo mínimo que necesitas saber

**1. El productor publica un evento y se olvida**

No llama a nadie en concreto; solo anuncia que algo ocurrió.

```csharp
// El servicio de pedidos publica el hecho, sin saber quién lo consumirá
await broker.PublishAsync("orders", new OrderPlaced { OrderId = 87, Total = 59.90m });
```

**2. Uno o varios consumidores se suscriben**

Cada sistema interesado escucha el tipo de evento que le importa y reacciona a su ritmo.

```csharp
// El servicio de facturación reacciona a su aire
broker.Subscribe<OrderPlaced>("orders", async evento =>
    await GenerarFacturaAsync(evento.OrderId));
```

**3. El *broker* desacopla a productor y consumidor**

Entre medias hay una cola o bus que guarda los eventos y los entrega. Si un consumidor está caído, el evento espera en la cola hasta que vuelva.

```
[Pedidos] --OrderPlaced--> ( broker ) --> [Facturación]
                                      --> [Inventario]
                                      --> [Emails]
```

**4. Es comunicación asíncrona**

El productor no espera respuesta. Esto hace el sistema más resistente (un consumidor lento no frena al resto) pero también significa que las cosas pasan "con retardo" y en otro orden.

**5. Dos patrones habituales: colas y pub/sub**

- **Cola (*queue*)**: cada mensaje lo procesa **un** consumidor. Útil para repartir trabajo.
- **Pub/Sub**: cada mensaje llega a **todos** los suscritos. Útil para notificar un hecho a varios.

## Lo que NO hace

- **No da respuesta inmediata** — si necesitas un resultado al instante (petición-respuesta), usa [REST](REST.md), [RPC](RPC.md) o [gRPC](gRPC.md).
- **No garantiza orden ni unicidad sin esfuerzo** — los eventos pueden llegar desordenados o repetidos; diseña consumidores *idempotentes*.
- **No es gratis en complejidad** — añade una pieza nueva (el broker) que hay que operar, monitorizar y entender al depurar.
- **No es lo mismo que un webhook** — el [webhook](Webhooks.md) es una llamada HTTP directa entre dos partes; aquí hay un intermediario que desacopla a muchos.

## Buenas prácticas avanzadas

- **El *dual write* es el bug silencioso número uno: usa el patrón *outbox*** — guardar el pedido en la base de datos y *después* publicar `OrderPlaced` son dos escrituras separadas: si el proceso muere entre ambas, tienes un pedido del que nadie se enteró (o al revés). El patrón **transactional outbox** lo resuelve: en la misma transacción que guarda el pedido, insertas el evento en una tabla `outbox`; un proceso aparte lee esa tabla y publica al broker. O pasan las dos cosas, o ninguna.
- **Un evento publicado es un contrato tan público como una API** — quitar un campo de `OrderPlaced` rompe consumidores que ni sabes que existen (esa es justo la gracia del desacoplamiento... y su peligro). Trata el esquema de cada evento con la misma disciplina que un endpoint: solo cambios compatibles (añadir campos opcionales) y, para lo demás, versiones (`OrderPlaced.v2`) o un *schema registry* que valide la compatibilidad al publicar.
- **Configura una *dead-letter queue* antes de necesitarla** — siempre habrá algún mensaje "veneno" que hace fallar al consumidor una y otra vez (un dato corrupto, un caso no previsto). Sin plan, ese mensaje bloquea la cola o se reintenta para siempre quemando CPU. Tras N intentos fallidos debe apartarse a una cola de mensajes muertos que alguien monitorice: el evento no se pierde y el resto de la cola sigue fluyendo.
- **Decide conscientemente entre eventos "finos" y "gordos"** — `OrderPlaced { orderId: 87 }` (notificación fina) obliga a cada consumidor a llamar de vuelta al servicio de pedidos, reintroduciendo el acoplamiento que querías evitar; incluir todos los datos (*event-carried state transfer*) evita esas llamadas pero convierte cada campo en contrato y puede viajar desactualizado. No hay respuesta universal: elegir sin pensarlo es como se acaba con lo peor de ambos.
- **Monitoriza el *lag* del consumidor, no solo los errores** — un consumidor puede estar "sano" (sin excepciones) y aun así procesar más despacio de lo que se publica: la cola crece y las facturas se generan cada vez más tarde, sin que salte ninguna alarma de error. La métrica clave de un sistema de eventos es cuántos mensajes esperan y cuánto retraso llevan, con alertas sobre ella.

---

*En resumen: las APIs dirigidas por eventos son un tablón de anuncios entre sistemas —uno publica "ha pasado X" y los interesados reaccionan por su cuenta— logrando bajo acoplamiento y resistencia a cambio de inmediatez.*
