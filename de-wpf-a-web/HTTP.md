# HTTP

## ¿Qué es?

HTTP es el idioma con el que el navegador y el servidor se hablan. Funciona por
**peticiones y respuestas**: el cliente envía una petición ("dame esta página", "guarda
este formulario") y el servidor devuelve una respuesta.

## ¿Por qué existe?

En WPF no necesitabas nada de esto: cuando querías datos, llamabas a un método y ya
estaba, todo en memoria. Pero como ahora el cliente y el servidor están en máquinas
distintas (ver [Modelo Cliente-Servidor](Modelo-Cliente-Servidor.md)), hace falta un
protocolo común para entenderse por la red. HTTP es ese protocolo, y es el que usa toda
la web.

> Si vienes de WPF, piensa en cada petición HTTP como una llamada a un método que vive
> en otro ordenador: le mandas argumentos, esperas, y recibes un resultado. La gran
> diferencia es que entre llamada y llamada el servidor **no recuerda nada** de ti.

## ¿Cuándo y para qué se usa?

Cada vez que abres una web, envías un formulario o una app consulta datos, hay HTTP por
debajo. Por ejemplo, en una tienda online: abrir la página del catálogo es una petición,
filtrar por categoría es otra, y pulsar "comprar" es otra más. Cada acción del usuario
suele traducirse en una o varias peticiones HTTP.

## Lo mínimo que necesitas saber

**1. Cada petición es independiente: HTTP no tiene memoria (es *stateless*)**

Este es el cambio mental más grande viniendo de escritorio. En WPF guardabas el estado
en campos del `ViewModel` y seguía ahí entre clics. En HTTP, el servidor atiende tu
petición, responde y **se olvida de ti**. Si quieres que recuerde quién eres entre una
página y otra, hay que apañarlo aparte (ver [Autenticación y estado web](Autenticacion-y-Estado-Web.md)).

**2. Cada petición tiene un verbo que dice qué quieres hacer**

```text
GET    /productos          → dame la lista de productos (leer)
POST   /productos          → crea un producto nuevo (con datos en el cuerpo)
PUT    /productos/42       → reemplaza el producto 42
DELETE /productos/42       → borra el producto 42
```

**3. Una petición lleva una URL, cabeceras y, a veces, un cuerpo**

- **URL**: a qué recurso apuntas (`/productos/42`).
- **Cabeceras** (*headers*): metadatos, como el formato que aceptas o tu identificación.
- **Cuerpo** (*body*): los datos que envías, típicamente en JSON al crear o actualizar.

**4. La respuesta trae un código de estado que resume qué pasó**

```text
200 OK                    → todo bien
201 Created               → se creó el recurso
400 Bad Request           → enviaste algo mal
401 Unauthorized          → no has iniciado sesión
404 Not Found             → no existe lo que pides
500 Internal Server Error → reventó algo en el servidor
```

> Truco para recordarlos: 2xx = bien, 4xx = culpa del cliente, 5xx = culpa del servidor.

**5. Los datos suelen viajar como JSON**

JSON es texto plano con la forma de un objeto. Es el equivalente web a serializar tu
clase C# para mandarla por la red:

```json
{ "id": 42, "nombre": "Camiseta", "precio": 19.99 }
```

## Lo que NO hace

- **No mantiene una conexión abierta** entre peticiones por defecto (para eso existen tecnologías aparte como WebSockets o SignalR).
- **No recuerda quién eres** por sí solo — el estado entre peticiones hay que gestionarlo explícitamente.
- **No cifra nada por sí mismo** — eso lo añade **HTTPS** (HTTP sobre una capa segura), que es lo que se usa siempre en producción.

## Buenas prácticas avanzadas

- **Respeta el contrato de GET: leer sin efectos secundarios** — un GET nunca debe modificar datos. No es solo estética: los navegadores precargan enlaces, los proxies cachean GETs y las herramientas los reintentan sin preguntar, porque el protocolo les promete que un GET es inofensivo. Un `GET /pedidos/42/cancelar` puede acabar ejecutándose "solo" por una precarga del navegador. Todo lo que cambie estado va en POST, PUT o DELETE.
- **Distingue qué verbos se pueden repetir sin miedo (idempotencia)** — PUT y DELETE aplicados dos veces dejan el mismo resultado; POST no (dos POST = dos pedidos). Esto importa cuando la red falla y hay que reintentar: un cliente puede repetir un PUT tranquilamente, pero repetir un POST exige protección extra (por ejemplo, que el servidor detecte duplicados). Diseñar sabiendo esto evita los clásicos pedidos dobles por doble clic.
- **La caché HTTP es tu optimización más barata** — la petición más rápida es la que no se hace. Con las cabeceras `Cache-Control` y `ETag`, el servidor puede decirle al navegador "esto vale 1 hora" o responder `304 Not Modified` (sin cuerpo) cuando nada cambió. Casi nadie las configura a propósito y por eso tantas webs repiten descargas idénticas en cada visita.
- **No devuelvas `200 OK` con un error dentro del cuerpo** — responder `200` con `{ "error": "no encontrado" }` rompe el contrato: caches, monitorización y clientes genéricos leen el código de estado, no tu JSON. Si algo falló, que el código lo diga (`404`, `400`, `500`) y el cuerpo lo detalle.
- **En C#, no hagas `new HttpClient()` por petición** — cada instancia abre conexiones que tardan minutos en liberarse; bajo carga agota los *sockets* de la máquina (error real y difícil de diagnosticar). Reutiliza una instancia o, en ASP.NET Core, usa `IHttpClientFactory`, que gestiona el ciclo de vida por ti.

---

*En resumen: HTTP es un protocolo de pregunta-respuesta sin memoria; cada interacción con el servidor es una petición independiente con su verbo, su URL y su código de estado.*
