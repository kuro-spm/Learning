# Long Polling

**¿Qué es?** Una técnica para simular datos en tiempo real usando solo peticiones HTTP normales: el cliente pregunta al servidor "¿hay algo nuevo?", y el servidor **deja la petición esperando** (sin responder) hasta que de verdad haya una novedad o pase un tiempo límite. Al recibir respuesta, el cliente vuelve a preguntar de inmediato.

---

## ¿Por qué existe?

Antes de [WebSockets](WebSockets.md) y [SSE](Server-Sent-Events.md), la única herramienta era la petición-respuesta de HTTP. El intento ingenuo era el *polling* normal: preguntar cada pocos segundos ("¿algo nuevo?... ¿y ahora?... ¿y ahora?"). Eso desperdicia muchísimas peticiones que casi siempre responden "no".

Long polling es el truco intermedio: en vez de responder "no" enseguida, el servidor **se calla y espera**. Solo contesta cuando hay algo que contar. Así reduces el ruido y consigues casi-tiempo-real sin tecnología nueva.

> Polling normal es llamar al timbre cada minuto a ver si ha llegado tu paquete. Long polling es llamar una vez y que el portero no cuelgue hasta que el paquete llegue de verdad.

---

## ¿Cuándo y para qué se usa?

Como **alternativa de compatibilidad** cuando no puedes usar WebSockets ni SSE (clientes antiguos, redes restringidas, proxies problemáticos). También como mecanismo de respaldo: algunas librerías de tiempo real usan WebSockets si pueden y caen a long polling si no. Para proyectos nuevos sin restricciones, hoy se prefieren WebSockets o SSE.

---

## Lo mínimo que necesitas saber

**1. El cliente pregunta y el servidor retiene la respuesta**

```js
async function escuchar() {
  const res = await fetch("/api/updates");  // el servidor NO responde hasta que haya algo
  const dato = await res.json();
  procesar(dato);
  escuchar();   // en cuanto responde, vuelve a preguntar
}
```

**2. El servidor espera a tener novedad (o agota un tiempo)**

Si en, digamos, 30 segundos no pasa nada, responde vacío para no dejar la conexión colgada para siempre, y el cliente reintenta.

**3. Usa HTTP normal de principio a fin**

No hay protocolos nuevos ni conexiones especiales: son peticiones `GET` corrientes que tardan más en responder. Por eso funciona donde otras técnicas fallan.

**4. Encadena peticiones para simular un flujo continuo**

La sensación de "tiempo real" surge de repetir el ciclo: pregunto → espero → recibo → vuelvo a preguntar.

---

## Lo que NO hace

- **No es tan eficiente como WebSockets/SSE** — sigue abriendo y cerrando peticiones; es un parche, no la solución ideal.
- **No es bidireccional fluido** — para enviar al servidor necesitas peticiones aparte.
- **No escala igual de bien** — muchas conexiones "en espera" consumen recursos del servidor.
- **No es la primera opción hoy** — solo cuando WebSockets o SSE no son viables.

---

*En resumen: long polling es el truco de pedir y que el servidor no cuelgue hasta tener algo que decir —casi-tiempo-real con HTTP normal, útil como respaldo cuando WebSockets o SSE no están disponibles.*
