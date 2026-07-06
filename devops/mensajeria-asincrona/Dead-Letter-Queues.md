# Dead Letter Queues

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

**¿Qué es?** Una *dead-letter queue* (cola de mensajes muertos, a menudo abreviada **DLQ**) es una cola aparte donde un broker aparta automáticamente los mensajes que no se han podido procesar correctamente tras varios intentos, en lugar de perderlos o de reintentarlos para siempre.

---

## ¿Por qué existe?

Sin una DLQ, un mensaje "veneno" —con un dato corrupto, un formato inesperado, un caso que el consumidor no contempla— provoca que el consumidor falle, el broker lo reintente, vuelva a fallar, y así indefinidamente. Mientras tanto, ese mensaje puede bloquear el procesamiento del resto de la cola (si el consumo es en orden) o simplemente consumir recursos en bucle.

La *dead-letter queue* corta ese bucle: tras un número máximo de intentos fallidos, el broker mueve el mensaje a una cola separada donde espera, sin bloquear nada, hasta que alguien lo revise.

> Piensa en la bandeja de "correo no entregado" del servicio postal: si una carta no se puede entregar tras varios intentos, no se destruye ni se sigue intentando para siempre; se aparta a un sitio donde alguien la revisa a mano.

---

## ¿Cuándo y para qué se usa?

En cualquier sistema de colas de producción: para que un pedido con un dato inesperado no bloquee el procesamiento del resto de pedidos de una tienda online, o para poder revisar más tarde, con calma, por qué falló el envío de un email de confirmación concreto sin perder el mensaje ni frenar los demás.

---

## Lo mínimo que necesitas saber

**1. Se activa tras un número máximo de intentos**

```csharp
// Azure Service Bus: tras 10 intentos fallidos, el mensaje va a la DLQ automáticamente
var options = new ServiceBusProcessorOptions { MaxDeliveryCount = 10 };
```

**2. También se puede enviar a la DLQ explícitamente**

Cuando el propio código detecta que un mensaje no tiene sentido (por ejemplo, un `orderId` que no existe), puede apartarlo sin esperar a agotar los reintentos.

```csharp
await args.DeadLetterMessageAsync(args.Message, reason: "OrderId no encontrado");
```

**3. La DLQ es una cola más, hay que vigilarla**

Los mensajes en la DLQ no desaparecen ni se resuelven solos: alguien (o alguna alerta) tiene que revisarlos, entender qué falló y decidir si se reprocesan o se descartan.

**4. Reprocesar desde la DLQ tras corregir el problema**

Una vez arreglado el motivo del fallo (un bug en el consumidor, un dato corregido), el mensaje se puede volver a enviar a la cola original para que se procese con normalidad.

---

## Lo que NO hace

- **No arregla el mensaje ni el error que lo causó** — solo evita que bloquee al resto; alguien tiene que investigar por qué llegó ahí.
- **No es automática en todos los brokers por defecto** — hay que configurarla explícitamente (número de reintentos, cola de destino).
- **No sustituye a las alertas** — una DLQ que nadie monitoriza es un cementerio de mensajes silencioso, tan malo como no tener DLQ.

---

## Buenas prácticas avanzadas

- **Diferencia expiración (TTL) de agotar reintentos** — un mensaje puede acabar en la DLQ por dos motivos distintos (falló demasiadas veces, o caducó sin que nadie lo consumiera); revisar el motivo (`DeadLetterReason`) antes de decidir si reprocesarlo evita relanzar un mensaje que en realidad ya no tiene sentido reenviar.
- **No reintentes automáticamente desde la DLQ sin cambiar nada** — reenviar sin más un mensaje que falló por un bug en el consumidor solo lo devuelve a la DLQ de nuevo; antes de reprocesar, confirma que la causa raíz está corregida.
- **Pon una alerta sobre el tamaño de la DLQ, no solo sobre la cola principal** — una DLQ que crece sin que nadie se entere es indistinguible de perder mensajes en silencio; una alerta simple sobre "hay N mensajes esperando en la DLQ" suele faltar en los sistemas que luego tienen sorpresas.

---

*En resumen: la dead-letter queue es la red de seguridad que aparta los mensajes que no se pueden procesar, para que un solo mensaje problemático no bloquee ni pierda el resto de la cola.*
