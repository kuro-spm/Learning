# WebSockets

## ¿Qué es?

**WebSockets** es una tecnología que abre un **canal de comunicación permanente y bidireccional** entre cliente y servidor sobre una única conexión. Una vez abierta, ambos lados pueden enviarse mensajes en cualquier momento, sin esperar a que el otro pregunte.

## ¿Por qué existe?

El modelo clásico de la web (y de REST) es petición-respuesta: el cliente pregunta y el servidor contesta. El servidor **no puede** iniciar la conversación; solo responde. Eso es un problema para datos "en vivo": un chat, una notificación, una cotización que cambia cada segundo. Sin WebSockets, el cliente tendría que preguntar una y otra vez ("¿hay algo nuevo?... ¿y ahora?"), lo que es ineficiente.

> REST es como enviar cartas: escribes, esperas respuesta, y solo recibes cuando tú has escrito antes. WebSockets es una llamada de teléfono abierta: los dos podéis hablar cuando queráis sin colgar y volver a marcar.

## ¿Cuándo y para qué se usa?

Siempre que necesites datos en tiempo real y en **ambas direcciones**:

- Un **chat** donde los mensajes llegan al instante.
- Un **juego online** multijugador.
- Una **pizarra colaborativa** donde ves escribir a los demás en vivo.
- Un panel de **cotizaciones** o métricas que se actualiza solo.

Si la comunicación va sobre todo del servidor al cliente y no necesitas el canal de vuelta, a veces basta con algo más simple ([Server-Sent Events](Server-Sent-Events.md)).

## Lo mínimo que necesitas saber

**1. La conexión empieza como HTTP y "asciende" (*upgrade*)**

El cliente pide pasar de HTTP a WebSocket mediante un *handshake*. Si el servidor acepta, la conexión queda abierta.

```http
GET /chat
Connection: Upgrade
Upgrade: websocket
```

**2. La URL usa el esquema `ws://` o `wss://`**

Como `http`/`https`, pero para WebSockets. `wss://` es la versión cifrada (la recomendada).

```
wss://api.miapp.com/chat
```

**3. Ambos lados envían y reciben en cualquier momento**

```js
const socket = new WebSocket("wss://api.miapp.com/chat");

socket.onmessage = (event) => console.log("Mensaje:", event.data);
socket.send("Hola a todos");   // el cliente envía cuando quiere
```

**4. El servidor puede empujar (*push*) datos sin que se los pidan**

Esta es la gran diferencia con REST: el servidor inicia el envío.

```js
// En cuanto hay un mensaje nuevo, el servidor lo manda a todos los conectados
socket.onmessage = (event) => mostrarMensaje(event.data);
```

**5. La conexión hay que mantenerla y cuidarla**

Una conexión abierta consume recursos. Hay que gestionar reconexiones si se cae y, a veces, enviar *pings* periódicos para comprobar que sigue viva.

## Lo que NO hace

- **No reemplaza a REST** — para operaciones normales de petición-respuesta, REST sigue siendo más simple; conviven.
- **No es gratis en recursos** — mantener miles de conexiones abiertas pesa más que atender peticiones sueltas.
- **No cachea ni se beneficia de la infraestructura HTTP** — al ser un canal persistente y propio, pierde las ventajas de caché y proxies del HTTP normal.
- **No garantiza la entrega por sí solo** — si la conexión se cae, los mensajes en vuelo pueden perderse; la lógica de reintento es cosa tuya.

---

*En resumen: WebSockets es una línea de teléfono siempre abierta entre cliente y servidor —ambos hablan cuando quieren— ideal para chats, juegos y cualquier dato en tiempo real bidireccional.*
