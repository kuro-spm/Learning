# Mensajería Asíncrona

## ¿Qué es?

La mensajería asíncrona es un estilo de comunicación entre sistemas en el que quien envía un mensaje no espera a que quien lo recibe lo procese: el mensaje se deposita en un intermediario (una **cola** o un **topic**) y el emisor continúa sin bloquearse, mientras el mensaje espera a ser recogido.

## ¿Por qué existe?

En la comunicación síncrona habitual —una llamada HTTP directa de un servicio a otro— quien llama se queda esperando la respuesta. Si el servicio llamado está caído, saturado o simplemente tarda, quien llama se bloquea o falla con él. Ambos quedan acoplados **en el tiempo**: tienen que estar disponibles a la vez para que la operación funcione.

La mensajería asíncrona rompe esa dependencia. El productor deposita el mensaje y sigue con lo suyo; el consumidor lo recoge cuando puede, aunque sea un segundo después o cinco minutos después porque estaba caído.

> Piensa en la diferencia entre una llamada de teléfono y un correo. En la llamada, ambas personas tienen que estar disponibles a la vez, y si una no contesta la comunicación falla. En el correo, lo envías y sigues con tu día; la otra persona lo lee cuando puede y, aunque tarde, el mensaje no se pierde: espera en su bandeja.

## ¿Cuándo y para qué se usa?

Cuando una operación no necesita respuesta inmediata, o cuando quieres que un pico de trabajo no tumbe al sistema que lo recibe. Los tres casos que aparecen una y otra vez:

- **Quitar trabajo del camino crítico.** Al confirmar un pedido en una tienda online, encolar el email de confirmación en lugar de enviarlo dentro de la misma petición HTTP. Si el servidor de correo va lento, el cliente no tiene por qué esperar.
- **Absorber picos.** Miles de imágenes subidas a la vez se reparten entre varios trabajadores que las van cogiendo a su ritmo, en lugar de intentar procesarlas todas de golpe.
- **Avisar a varios sistemas de un mismo hecho.** Que se ha confirmado un pedido interesa a facturación, a inventario y a estadísticas. Con mensajería, el servicio de pedidos publica una vez y no necesita conocer a ninguno de los tres.

Ese último caso es la infraestructura sobre la que se apoyan las [APIs dirigidas por eventos](../../arquitectura-de-software/tipos-de-apis/APIs-Dirigidas-por-Eventos.md): la mensajería es el transporte que hace posible ese estilo de comunicación.

---

## El vocabulario

Cuatro términos que aparecen en toda la colección y conviene fijar antes de seguir:

| Término | Qué es |
|---|---|
| **Productor** (*producer*, *publisher*) | Quien crea y envía el mensaje |
| **Consumidor** (*consumer*, *subscriber*) | Quien lo recibe y lo procesa |
| **Broker** | El intermediario que almacena y entrega (RabbitMQ, Azure Service Bus, Kafka...) |
| **Mensaje** | El dato que viaja: unas cabeceras y un cuerpo, normalmente JSON |

El productor y el consumidor **no se conocen**. Ninguno tiene la dirección del otro, ni sabe cuántos hay al otro lado. Los dos conocen únicamente al broker y el nombre de la cola o el topic. Ese es todo el desacoplamiento, y de ahí sale el resto de propiedades del modelo.

## Cola frente a topic: los dos patrones de reparto

Esta es la primera decisión de diseño de cualquier sistema de mensajería, y equivocarse aquí se paga caro después.

### Cola: cada mensaje lo procesa uno

Un mensaje depositado en una cola lo recoge **un único** consumidor, aunque haya varios escuchando. El broker reparte entre ellos, así que añadir consumidores multiplica la capacidad de proceso.

```csharp
// El servicio de pedidos encola el envío del email y sigue de inmediato
await sender.SendMessageAsync(new ServiceBusMessage(
    JsonSerializer.SerializeToUtf8Bytes(new EnviarEmailConfirmacion { PedidoId = pedido.Id })));
```

Con tres instancias del servicio de emails escuchando la misma cola, cada email lo envía una sola de ellas. Si una se cae, las otras dos absorben su parte. Esto es **reparto de carga**, y es lo que quieres cuando la tarea es "haz esto una vez".

### Topic: cada mensaje llega a todos

Un mensaje publicado en un *topic* llega a **todas** las suscripciones activas, cada una con su propia copia independiente.

```csharp
// El servicio de pedidos publica el hecho una sola vez
await sender.SendMessageAsync(new ServiceBusMessage(
    JsonSerializer.SerializeToUtf8Bytes(new PedidoConfirmado { PedidoId = pedido.Id })));
```

Facturación, inventario y estadísticas tienen cada uno su suscripción sobre el mismo topic. Los tres reciben el mismo `PedidoConfirmado` y cada uno hace lo suyo sin enterarse de los demás. Esto es **difusión**, y es lo que quieres cuando la tarea es "entérate de que ha pasado esto".

### Cómo elegir

La pregunta que lo resuelve: **¿estoy pidiendo que se haga algo, o estoy contando que ha pasado algo?**

| | Cola | Topic |
|---|---|---|
| Intención | Un comando: "envía este email" | Un evento: "se confirmó el pedido 4711" |
| Nombre típico | Imperativo: `EnviarEmailConfirmacion` | Pasado: `PedidoConfirmado` |
| Receptores | Uno, el que toque | Todos los interesados |
| Añadir un receptor nuevo | Hay que cambiar al productor | El productor no se entera |

Ese último punto es el que suele decidir. Si mañana hay que avisar también al servicio de fidelización, con un topic basta con crear una suscripción nueva: nadie toca el servicio de pedidos. Con colas, alguien tiene que añadir un envío más en el productor.

El error clásico es usar una cola donde debería haber un topic y acabar construyendo "colas paralelas" a mano: el productor envía el mismo mensaje a tres colas distintas y vuelve a conocer a sus tres consumidores, que era justo lo que se quería evitar.

## El ciclo de vida de un mensaje

Entender estos cuatro pasos explica casi todo el comportamiento raro que te encontrarás después.

**1. El productor publica.** El mensaje llega al broker, que lo guarda.

**2. El broker lo entrega a un consumidor.** Aquí está la parte que sorprende: el mensaje **no se borra al entregarse**. Queda marcado como "en curso" (bloqueado, invisible para los demás) durante un tiempo limitado.

**3. El consumidor procesa y confirma.** Al terminar bien, envía la confirmación (*ack*, *complete*) y **entonces** el broker lo borra.

```csharp
processor.ProcessMessageAsync += async args =>
{
    var evento = args.Message.Body.ToObjectFromJson<PedidoConfirmado>();
    await GenerarFacturaAsync(evento.PedidoId);

    await args.CompleteMessageAsync(args.Message);   // ← aquí se borra, no antes
};
```

**4. Si no confirma, el mensaje vuelve.** Si el consumidor se cae a mitad, o tarda más que el tiempo de bloqueo, el broker considera que nadie lo procesó y **lo vuelve a poner disponible**. Otro consumidor lo recogerá.

Ese cuarto punto es el que garantiza que no se pierde nada cuando un proceso muere. Y es también el origen del efecto secundario más importante de todo el modelo.

## Las garantías de entrega

Hay tres niveles teóricos y, en la práctica, uno que vas a usar.

| Garantía | Qué significa | Realidad |
|---|---|---|
| **Como mucho una vez** | Puede perderse, nunca se duplica | Solo si el dato es descartable (telemetría) |
| **Al menos una vez** | Nunca se pierde, puede duplicarse | **Lo que usan RabbitMQ y Service Bus por defecto** |
| **Exactamente una vez** | Ni se pierde ni se duplica | No existe de extremo a extremo |

Lo tercero merece explicación, porque muchos brokers lo anuncian. Un broker puede evitar duplicar **dentro de su propio ámbito**, pero no puede saber si el efecto de tu consumidor —cobrar una tarjeta, llamar a una API externa— llegó a ocurrir antes de que el proceso muriera. Siempre queda una ventana entre "hice el trabajo" y "confirmé el mensaje". Si el proceso cae ahí, el mensaje vuelve y el trabajo se repite.

El resultado práctico: **da por supuesto que cualquier mensaje puede llegarte dos veces.** No es un caso raro que ocurre una vez al año; ocurre en cada despliegue, en cada reinicio y en cada pico de latencia.

## Idempotencia: la parte que te toca construir a ti

Idempotente significa que ejecutar la operación dos veces produce el mismo resultado que ejecutarla una. No es una propiedad que dé el broker: la construyes tú en el consumidor.

Mira la diferencia con un ejemplo de facturación. Este consumidor genera dos facturas si el mensaje llega repetido:

```csharp
// ❌ No idempotente: dos entregas → dos facturas
await GenerarFacturaAsync(evento.PedidoId);
```

Y este no:

```csharp
// ✅ Idempotente: registra qué mensajes ya procesó
if (await procesados.ExisteAsync(args.Message.MessageId))
{
    await args.CompleteMessageAsync(args.Message);   // ya hecho, confirmar y salir
    return;
}

await GenerarFacturaAsync(evento.PedidoId);
await procesados.RegistrarAsync(args.Message.MessageId);
await args.CompleteMessageAsync(args.Message);
```

Hay tres formas de conseguirlo, de mejor a peor:

1. **Que la operación sea naturalmente repetible.** Un `UPDATE Pedidos SET Estado = 'Confirmado'` da igual ejecutarlo diez veces. Si puedes plantear el trabajo así, no necesitas nada más.
2. **Una clave única en la base de datos.** Un índice único sobre `PedidoId` en la tabla de facturas hace que el segundo intento falle con violación de clave, que capturas y tratas como "ya estaba hecho". La base de datos hace el trabajo y no hay condición de carrera.
3. **Una tabla de mensajes procesados**, como el ejemplo de arriba. Es la más general y también la que tiene una trampa: entre comprobar y registrar hay una ventana en la que dos entregas simultáneas pueden colarse las dos. Si eso importa, la comprobación y el trabajo tienen que ir en la misma transacción.

Ese registro de mensajes ya procesados tiene nombre propio, *inbox pattern*, y es el reflejo exacto del [Outbox Pattern](Outbox-Pattern.md) en el lado del consumidor.

## Qué pierdes al pasar a asíncrono

No es gratis. Cuatro cosas que tenías y dejas de tener:

**La respuesta inmediata.** El productor recibe un "recibido", no un "hecho". Si la interfaz necesita mostrar el resultado, hay que resolverlo de otra forma: devolver `202 Accepted` con una URL donde consultar el estado, o notificar por WebSocket cuando termine.

**El orden.** Con varios consumidores repartiéndose una cola, el mensaje 2 puede terminar antes que el 1. Los brokers ofrecen mecanismos para garantizar orden dentro de un grupo (las *sesiones* de Service Bus, las claves de partición de Kafka), pero todos funcionan igual: serializando el proceso de ese grupo, lo que reduce el paralelismo. El orden se paga en rendimiento.

**La pila de llamadas.** Cuando algo falla en una llamada HTTP, la excepción sube y ves dónde reventó. Con mensajería, el fallo ocurre en otro proceso, minutos después, sin relación aparente con lo que lo originó. Reconstruir el recorrido exige propagar un identificador de correlación en las cabeceras de cada mensaje y tener [observabilidad](../observabilidad/README.md) montada. No es opcional: sin eso, depurar es adivinar.

**La consistencia inmediata.** Entre que el pedido se guarda y que facturación se entera pasa un tiempo. Durante esa ventana el sistema es incoherente: el pedido existe y su factura no. Casi siempre es aceptable, pero tiene que ser una decisión, no una sorpresa.

## Cuándo no usar mensajería

Merece una sección propia porque el patrón se aplica de más con frecuencia:

- **Cuando necesitas el resultado para continuar.** Validar un pago antes de confirmar el pedido es síncrono por naturaleza. Meter una cola ahí solo añade latencia y complejidad.
- **Cuando solo hay un productor y un consumidor y ambos son tuyos y siempre están vivos.** Una llamada HTTP con reintentos es más simple de escribir, desplegar y depurar.
- **Cuando el evento no sale del proceso.** Si emisor y receptor viven en la misma aplicación, un [domain event](../../arquitectura-de-software/patrones-de-diseno/Domain-Events.md) en memoria hace el trabajo sin desplegar un broker.
- **Cuando el equipo no tiene monitorización.** Un broker mal vigilado convierte un fallo visible (un 500 en el log) en uno invisible (una cola que crece). Si no vas a poner alertas, la llamada síncrona falla de forma más honesta.

---

## Buenas prácticas avanzadas

- **Versiona el contrato del mensaje desde el primer día.** Un mensaje publicado es una API pública con consumidores que no controlas y que se despliegan en otro momento. Añadir un campo opcional es seguro; renombrar o eliminar uno rompe a quien todavía no se ha actualizado. Incluir una propiedad `version` en las cabeceras y no reutilizar nunca el significado de un campo existente cuesta nada al principio y evita una migración coordinada de varios servicios después.
- **Que el mensaje lleve el identificador, no el objeto entero.** Publicar `PedidoConfirmado { PedidoId = 4711 }` en lugar del pedido completo con sus líneas evita dos problemas: el mensaje no queda obsoleto si el pedido cambia entre publicación y consumo, y no acopla el esquema interno del productor a todos sus consumidores. El coste es una consulta extra en el consumidor, que casi siempre compensa. La excepción es cuando el consumidor necesita el estado **en el momento del evento**, no el actual.
- **Mide la antigüedad del mensaje más viejo, no solo cuántos hay.** Una cola con 10 000 mensajes que se vacía en un minuto está sana; una con 50 que llevan dos horas esperando está rota. La profundidad de cola sube y baja con el tráfico normal y genera falsas alarmas; la antigüedad del mensaje más antiguo detecta el consumidor atascado, que es el fallo real y el que no lanza ninguna excepción.
- **Fija el tiempo de bloqueo por encima del peor caso de tu consumidor, no del caso medio.** Si procesar tarda 30 segundos de media pero 3 minutos cuando la API externa va lenta, con un bloqueo de un minuto el broker reentregará mensajes que **sí se estaban procesando**. El resultado es trabajo duplicado bajo carga, justo cuando peor viene. O subes el bloqueo, o el consumidor lo renueva periódicamente mientras trabaja.
- **Separa el reintento del error transitorio del error permanente.** Un timeout de red merece reintentarse; un mensaje con un campo obligatorio vacío fallará igual las diez veces y solo gasta capacidad. Distinguirlos en el consumidor —reintentar unos, mandar los otros directamente a la [dead-letter queue](Dead-Letter-Queues.md)— es lo que evita que un mensaje corrupto consuma el presupuesto de reintentos de toda la cola.

## Recursos didácticos

- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/) — el catálogo de referencia de los patrones de mensajería, con un diagrama por patrón. Es de 2003 y sigue siendo la fuente que usan todos los brokers para nombrar sus piezas; leer las fichas de *Point-to-Point Channel*, *Publish-Subscribe Channel* e *Idempotent Receiver* fija el vocabulario de golpe.
- [RabbitMQ Simulator](http://tryrabbitmq.com/) — un simulador visual donde arrastras productores, exchanges y colas y ves los mensajes moverse. Aunque uses otro broker, es la forma más rápida de que la diferencia entre cola y topic deje de ser abstracta.
- [Microservices.io — Messaging](https://microservices.io/patterns/communication-style/messaging.html) — fichas cortas sobre mensajería, outbox e idempotencia, con las ventajas y los inconvenientes de cada decisión enumerados sin adornos.

---

*En resumen: la mensajería asíncrona desacopla a quien envía de quien procesa —ni tienen que estar disponibles a la vez, ni uno espera al otro— a cambio de que el orden, la entrega única y la trazabilidad dejen de venir gratis.*
