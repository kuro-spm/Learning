# Azure Service Bus

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Azure Service Bus es el servicio de mensajería **gestionado** de Microsoft Azure: ofrece colas y topics/subscriptions listos para usar, sin que tengas que instalar, actualizar ni escalar tú mismo un broker.

## ¿Por qué existe?

Operar un broker de mensajería propio (como RabbitMQ) implica instalarlo, mantenerlo actualizado, dimensionar su capacidad, configurar alta disponibilidad y vigilarlo. Para muchos equipos, sobre todo si ya despliegan en Azure, ese trabajo de infraestructura no aporta valor de negocio directo. Azure Service Bus ofrece las mismas ideas de fondo (colas, publicación/suscripción, entrega fiable) como un servicio ya operado por Microsoft: se crea desde Azure Portal o con código, se paga por uso, y la disponibilidad y el escalado son responsabilidad del proveedor.

> Es a RabbitMQ lo que una base de datos gestionada (Azure SQL) es a instalar SQL Server en tu propia máquina virtual: mismas ideas de fondo, pero sin que tú administres el software de base.

## ¿Cuándo y para qué se usa?

Cuando la aplicación ya vive en Azure y no quieres asumir la carga operativa de un broker propio. Ejemplos: una API de una tienda online alojada en Azure App Service que encola el envío de emails de confirmación en una cola de Service Bus; varios microservicios en Azure que se suscriben a un mismo *topic* de eventos de pedidos, cada uno con su propia *subscription* e independiente de los demás.

## Lo mínimo que necesitas saber

**1. Colas: entrega a un único consumidor**

```csharp
await using var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender("cola-envio-emails");
await sender.SendMessageAsync(new ServiceBusMessage(
    JsonSerializer.SerializeToUtf8Bytes(new EnviarEmailConfirmacion { OrderId = pedido.Id })));
```

**2. Topics y subscriptions: entrega a varios consumidores independientes**

Un *topic* es el punto de publicación; cada *subscription* es una cola independiente que recibe su propia copia de cada mensaje publicado.

```csharp
var sender = client.CreateSender("pedidos-eventos"); // el topic
await sender.SendMessageAsync(new ServiceBusMessage(
    JsonSerializer.SerializeToUtf8Bytes(new PedidoConfirmado { OrderId = pedido.Id })));

// Facturación e Inventario tienen cada uno su propia subscription sobre "pedidos-eventos"
```

**3. Reglas de filtrado en las subscriptions**

Cada *subscription* puede definir un filtro para quedarse solo con los mensajes que le interesan, sin que el publicador tenga que saber quién los recibe.

```
Regla SQL-like sobre una propiedad del mensaje: Estado = 'Confirmado'
```

**4. Procesar mensajes con el `ServiceBusProcessor`**

```csharp
var processor = client.CreateProcessor("cola-envio-emails");
processor.ProcessMessageAsync += async args =>
{
    var evento = args.Message.Body.ToObjectFromJson<EnviarEmailConfirmacion>();
    await EnviarEmailAsync(evento.OrderId);
    await args.CompleteMessageAsync(args.Message);
};
processor.ProcessErrorAsync += args =>
{
    logger.LogError(args.Exception, "Error procesando mensaje");
    return Task.CompletedTask;
};
await processor.StartProcessingAsync();
```

**5. Sesiones para garantizar orden cuando hace falta**

Agrupando mensajes relacionados bajo el mismo `SessionId`, Service Bus garantiza que se procesen en orden y de uno en uno dentro de esa sesión (por ejemplo, todos los eventos de un mismo pedido).

## Lo que NO hace

- **No lo administras tú** — no hay servidor que actualizar ni versión que instalar; a cambio, tienes menos control sobre configuraciones muy finas de bajo nivel.
- **No es gratis por volumen ilimitado** — se factura por operaciones y por el nivel de servicio contratado (Basic, Standard, Premium), cada uno con límites distintos de tamaño de mensaje y funcionalidades.
- **No funciona fuera de Azure sin más** — está pensado para integrarse con el resto del ecosistema Azure; migrar a otro proveedor cloud implica cambiar de tecnología de mensajería.
- **No sustituye a un almacén de datos** — igual que cualquier broker, es tránsito, no persistencia a largo plazo.

## Buenas prácticas avanzadas

- **Elige el nivel de servicio (*tier*) según lo que de verdad necesitas, no el más alto por defecto** — el nivel Premium aporta aislamiento de recursos y mayor rendimiento, pero cuesta bastante más que Standard; muchos escenarios con volumen moderado funcionan perfectamente en Standard, y subir de nivel "por si acaso" es gasto innecesario.
- **Configura *dead-lettering* automático por caducidad y por máximo de entregas** — Service Bus puede mover automáticamente a la cola de mensajes muertos los mensajes que superan un número de intentos de entrega o que caducan (`TimeToLive`) sin ser procesados; dejarlo en los valores por defecto sin revisarlos hace que mensajes problemáticos se reintenten indefinidamente o se pierdan silenciosamente.
- **Usa sesiones solo cuando el orden realmente importa** — activar sesiones fuerza a procesar de uno en uno los mensajes de la misma sesión, lo que reduce el paralelismo; es la herramienta correcta cuando el orden es un requisito real (los eventos de un mismo pedido, por ejemplo), pero aplicarla por costumbre a todo limita el rendimiento sin necesidad.
- **Diferencia `AbandonMessageAsync` de no confirmar nada** — si el procesamiento falla, abandonar explícitamente el mensaje (`AbandonMessageAsync`) lo devuelve a la cola de inmediato para reintentarlo; dejar que expire el *lock* sin hacer nada consigue el mismo efecto pero solo después de agotar el tiempo de bloqueo, retrasando el reintento sin motivo.

---

*En resumen: Azure Service Bus da las mismas garantías de un broker de mensajería —colas, topics/subscriptions, entrega fiable— pero como servicio gestionado, para que el equipo se ocupe de la lógica de negocio y no de administrar el broker.*
