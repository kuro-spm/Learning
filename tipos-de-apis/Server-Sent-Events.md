# Server-Sent Events (SSE)

**¿Qué es?** Una tecnología que permite al **servidor enviar un flujo continuo de mensajes al cliente** sobre una única conexión HTTP, en **una sola dirección** (del servidor al cliente). El cliente abre la conexión una vez y va recibiendo actualizaciones a medida que ocurren.

---

## ¿Por qué existe?

Muchas veces solo necesitas que el servidor te avise de cosas nuevas, sin necesidad de un canal de vuelta: notificaciones, una barra de progreso, una lista de noticias que crece. Montar [WebSockets](WebSockets.md) (bidireccional, más complejo) para eso es matar moscas a cañonazos.

> Si WebSockets es una llamada de teléfono (los dos hablan), SSE es una emisora de radio: el servidor emite y tú escuchas. No puedes contestarle por el mismo canal, pero para "recibir novedades" es justo lo que necesitas.

SSE existe para ese caso —**solo servidor → cliente**— de forma sencilla y sobre HTTP normal, aprovechando que el navegador trae soporte integrado.

---

## ¿Cuándo y para qué se usa?

Cuando el flujo de datos va sobre todo del servidor al cliente: notificaciones en vivo, el avance de una tarea larga (procesar un archivo), un *feed* de noticias o el "está escribiendo..." de respuestas que llegan poco a poco. Si necesitas que el cliente también envíe en tiempo real, entonces sí tocan WebSockets.

---

## Lo mínimo que necesitas saber

**1. El cliente se suscribe con `EventSource`**

El navegador trae esta API de fábrica; reconecta sola si se cae.

```js
const source = new EventSource("/api/notifications");
source.onmessage = (event) => console.log("Novedad:", event.data);
```

**2. El servidor responde con un tipo especial y va emitiendo**

La respuesta es de tipo `text/event-stream` y no se cierra: el servidor escribe mensajes con el formato `data: ...`.

```http
GET /api/notifications
→ Content-Type: text/event-stream

data: { "type": "order", "id": 87 }

data: { "type": "order", "id": 88 }
```

**3. Va sobre HTTP normal**

No hay *upgrade* ni protocolo nuevo: es una respuesta HTTP que se mantiene abierta. Por eso atraviesa bien proxies y firewalls.

**4. Es unidireccional**

El cliente no puede enviar datos por este canal; si necesita hablar con el servidor, usa una petición HTTP aparte.

---

## Lo que NO hace

- **No permite enviar del cliente al servidor** — solo recibe; para ida y vuelta, [WebSockets](WebSockets.md).
- **No transmite binario cómodamente** — está pensado para texto (típicamente JSON dentro de `data:`).
- **No abre conexiones ilimitadas** — los navegadores limitan cuántas conexiones SSE puede tener abiertas una misma página.
- **No reemplaza a REST** — para pedir o cambiar datos sigues usando peticiones normales; SSE solo aporta el flujo de novedades.

---

## Buenas prácticas avanzadas

- **Usa `id:` en cada evento para reanudar sin huecos** — si el servidor envía un `id:` con cada mensaje, el navegador lo recuerda y, al reconectar tras una caída, manda la cabecera `Last-Event-ID` automáticamente. Si tu servidor la usa para reenviar lo ocurrido desde ese punto, la reconexión no pierde eventos; si la ignoras, cada microcorte deja huecos silenciosos en el flujo.
- **El enemigo en producción es el *buffering* intermedio** — el clásico "en local llegan al instante y en producción a golpes de 30 segundos": un proxy (nginx) o la compresión están acumulando la respuesta antes de reenviarla. Desactiva el buffering para estas rutas (`X-Accel-Buffering: no`, sin gzip en el stream) y envía un comentario `: ping` periódico para que nada dé la conexión por muerta.
- **El límite de conexiones se esfuma con HTTP/2** — el tope de ~6 conexiones SSE por dominio es por *conexión* HTTP/1.1; sobre HTTP/2 los streams se multiplexan y el problema prácticamente desaparece. Si tu app abre varias pestañas o varios flujos, servir SSE sobre HTTP/2 (o compartir un único `EventSource` entre pestañas) marca la diferencia.

---

*En resumen: SSE es una emisora de radio del servidor al cliente —flujo continuo, una sola dirección, sobre HTTP normal— perfecta para notificaciones y progreso sin la complejidad de WebSockets.*
