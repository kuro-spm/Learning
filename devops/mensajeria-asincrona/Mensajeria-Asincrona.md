# Mensajería Asíncrona

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

La mensajería asíncrona es un estilo de comunicación entre sistemas en el que quien envía un mensaje no espera a que quien lo recibe lo procese: el mensaje se deposita en un intermediario (una **cola** o un **topic**) y el emisor continúa sin bloquearse, mientras el mensaje espera a ser recogido y procesado.

## ¿Por qué existe?

En la comunicación síncrona habitual (una llamada HTTP directa de un servicio a otro), quien llama se queda esperando la respuesta. Si el servicio llamado está caído, saturado o simplemente tarda mucho, quien llama se bloquea o falla con él. Además, ambos servicios quedan acoplados en el tiempo: los dos tienen que estar disponibles a la vez para que la operación funcione.

La mensajería asíncrona rompe esa dependencia temporal: el productor deposita el mensaje y sigue con lo suyo; el consumidor lo recoge cuando puede, aunque eso sea un segundo después o cinco minutos después porque estaba caído.

> Piensa en la diferencia entre una llamada de teléfono y un correo. En la llamada (síncrona), ambas personas tienen que estar disponibles a la vez, y si una no contesta, la comunicación falla. En el correo (asíncrono), lo envías y sigues con tu día; la otra persona lo lee cuando puede, y aunque tarde en responder, el mensaje no se pierde: espera en su bandeja de entrada.

## ¿Cuándo y para qué se usa?

Cuando una operación no necesita una respuesta inmediata, o cuando quieres que un pico de trabajo no tumbe al sistema que lo recibe. Ejemplos habituales: al confirmar un pedido en una tienda online, encolar el envío del email de confirmación en lugar de mandarlo en la misma petición HTTP (si el servicio de email está lento, el cliente no tiene por qué esperar); repartir el procesamiento de miles de imágenes subidas a la vez entre varios trabajadores que las van cogiendo de una cola a su ritmo; notificar a la vez a varios sistemas distintos (facturación, inventario, estadísticas) de que se ha creado un pedido, sin que el servicio de pedidos tenga que conocerlos ni llamarlos uno a uno. Este es, de hecho, el mismo patrón que sustenta las [APIs dirigidas por eventos](../../arquitectura-de-software/tipos-de-apis/APIs-Dirigidas-por-Eventos.md): la mensajería asíncrona es la infraestructura (colas, brokers) que hace posible ese estilo de comunicación entre servicios.

## Lo mínimo que necesitas saber

**1. Cola (*queue*): cada mensaje lo procesa uno**

Un mensaje depositado en una cola lo recoge y procesa **un único** consumidor, aunque haya varios escuchando (se reparten el trabajo entre ellos). Útil para repartir carga de trabajo.

```csharp
// El servicio de pedidos encola el envío del email, sin esperar a que se envíe
await queueClient.SendMessageAsync(new EnviarEmailConfirmacion { OrderId = pedido.Id });
```

**2. Topic + suscripciones: cada mensaje llega a varios**

Un mensaje publicado en un *topic* llega a **todas** las suscripciones activas, cada una con su propia copia. Útil cuando varios sistemas independientes necesitan enterarse del mismo hecho.

```csharp
await topicClient.SendMessageAsync(new PedidoConfirmado { OrderId = pedido.Id });
// Se suscriben por separado: Facturación, Inventario y Estadísticas
```

**3. El productor no espera respuesta**

```csharp
// El servicio de pedidos publica y continúa de inmediato
await bus.PublishAsync(new PedidoConfirmado { OrderId = pedido.Id });
return Ok(); // no espera a que se procesen las notificaciones derivadas
```

**4. El consumidor procesa a su ritmo**

```csharp
// El servicio de facturación recoge el mensaje cuando puede
await processor.ProcessMessageAsync(async args =>
{
    var evento = args.Message.Body.ToObjectFromJson<PedidoConfirmado>();
    await GenerarFacturaAsync(evento.OrderId);
    await args.CompleteMessageAsync(args.Message);
});
```

**5. El mensaje debe confirmarse (*ack*) tras procesarse**

Mientras el consumidor no confirme que ha terminado, el broker considera el mensaje pendiente. Si el consumidor se cae a mitad de proceso sin confirmar, el mensaje vuelve a estar disponible para reintentarlo, en lugar de perderse.

## Lo que NO hace

- **No da una respuesta inmediata** — si necesitas el resultado al instante dentro de la misma petición, esto no es lo que buscas; usa una llamada síncrona (REST, gRPC).
- **No garantiza el orden de entrega sin configuración explícita** — por defecto, los mensajes pueden procesarse en un orden distinto al que se enviaron.
- **No garantiza entrega única sin esfuerzo** — un mensaje puede entregarse más de una vez (p. ej. si el consumidor confirma tarde); los consumidores deben diseñarse para ser *idempotentes*.
- **No es gratis en complejidad operativa** — añade una pieza más (el broker) que hay que desplegar, monitorizar y entender al depurar un problema.

## Buenas prácticas avanzadas

- **Diseña los consumidores para ser idempotentes desde el principio** — dado que un mensaje puede llegar duplicado, procesar dos veces "cobrar 50€" o "generar una factura" sin protección produce cobros o facturas duplicadas. Guardar el identificador del mensaje ya procesado (o hacer que la operación de negocio sea naturalmente repetible, como un `UPSERT`) evita este problema de raíz.
- **No conviertas el envío del mensaje y el cambio de estado en dos operaciones separadas sin red de seguridad** — guardar el pedido en base de datos y publicar el evento son dos acciones distintas; si el proceso falla justo entre medias, tienes un pedido sin evento (o un evento de un pedido que no llegó a guardarse). El patrón *transactional outbox* (guardar el evento en la misma transacción que el dato, y publicarlo después desde esa tabla) es la forma estándar de evitarlo.
- **Decide con cabeza entre cola y topic, no por costumbre** — usar una cola cuando en realidad varios sistemas necesitan enterarse del mismo hecho obliga a construir "colas paralelas" a mano; usar un topic cuando solo un consumidor debería procesar cada mensaje puede acabar en que varios lo procesen sin querer. El patrón importa tanto como la tecnología concreta.
- **Monitoriza cuántos mensajes esperan y desde cuándo, no solo si hay errores** — un consumidor sin excepciones puede seguir estando "enfermo": si procesa más despacio de lo que se publica, la cola crece sin parar y los efectos (facturas, emails, notificaciones) llegan cada vez más tarde sin que salte ninguna alerta de error.

---

*En resumen: la mensajería asíncrona desacopla a quien envía de quien procesa —ni tienen que estar disponibles a la vez, ni uno espera al otro— a cambio de asumir que el orden y la entrega única ya no vienen gratis.*
