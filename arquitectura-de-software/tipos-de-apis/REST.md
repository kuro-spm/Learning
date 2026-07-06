# REST

## ¿Qué es?

**REST** (*Representational State Transfer*) es un estilo de diseño de APIs web que organiza todo en torno a **recursos** (cosas: productos, usuarios, pedidos) identificados por una URL, sobre los que actúas usando los verbos estándar de HTTP (`GET`, `POST`, `PUT`, `DELETE`). Es, con diferencia, el estilo más extendido.

## ¿Por qué existe?

Antes de REST, cada API inventaba sus propias reglas y operaciones (`/getUser`, `/createOrderNow`, `/deleteThing`), y aprender una no servía para la siguiente. REST propuso un convenio sencillo aprovechando lo que la web ya tenía: las URLs para nombrar cosas y los verbos HTTP para decir qué hacer con ellas.

> Piensa en una biblioteca bien ordenada: cada libro tiene su ubicación fija (la URL), y las acciones son siempre las mismas —consultar, añadir, reemplazar, retirar— independientemente del libro. Una vez aprendes el sistema, sabes moverte por cualquier sección.

El resultado: APIs predecibles. Si sabes usar una API REST, sabes usar casi cualquier otra, porque todas siguen el mismo patrón.

## ¿Cuándo y para qué se usa?

Es la opción por defecto para la mayoría de APIs web públicas e internas: la API de una tienda online, la de una app de tareas, la de un blog. Funciona especialmente bien cuando los datos encajan de forma natural en "colecciones de cosas" (una lista de productos, un pedido, un usuario) y las operaciones son las típicas de crear, leer, actualizar y borrar (el llamado **CRUD**).

## Lo mínimo que necesitas saber

**1. Todo es un recurso con su URL**

La URL identifica la *cosa*, no la acción. El verbo HTTP dice qué hacer.

```http
/api/products        → la colección de productos
/api/products/42     → el producto concreto nº 42
/api/products/42/reviews → las reseñas de ese producto
```

**2. Los verbos HTTP definen la operación**

```http
GET    /api/products       → listar productos
GET    /api/products/42    → obtener el producto 42
POST   /api/products       → crear un producto nuevo
PUT    /api/products/42    → reemplazar el producto 42
DELETE /api/products/42    → borrar el producto 42
```

**3. Los códigos de estado HTTP comunican el resultado**

El servidor responde con un número estándar que indica qué pasó.

```http
200 OK            → todo bien
201 Created       → recurso creado
404 Not Found     → no existe
400 Bad Request   → la petición venía mal
401 Unauthorized  → no estás identificado
500 Internal...   → error del servidor
```

**4. Los datos viajan normalmente en JSON**

```json
POST /api/products
{ "name": "Teclado mecánico", "price": 59.90 }

→ 201 Created
{ "id": 87, "name": "Teclado mecánico", "price": 59.90 }
```

**5. Es *stateless* (sin estado)**

Cada petición es independiente y lleva toda la información necesaria (incluida la credencial). El servidor no "recuerda" la petición anterior, lo que facilita escalar a muchos servidores.

```http
GET /api/orders
Authorization: Bearer eyJhbGciOi...   ← cada petición se identifica sola
```

## Lo que NO hace

- **No es un protocolo** — es un *estilo*; usa HTTP, que sí es el protocolo.
- **No evita el *over-fetching*** — un endpoint devuelve un objeto fijo; a veces recibes más datos de los que necesitas (problema que ataca [GraphQL](GraphQL.md)).
- **No es ideal para tiempo real** — está pensado para petición-respuesta; para datos "en vivo" hay otros estilos ([WebSockets](WebSockets.md), [SSE](Server-Sent-Events.md)).
- **No obliga a cumplir todas sus reglas** — muchas APIs se llaman "REST" sin serlo del todo; en la práctica hay grados.

## Buenas prácticas avanzadas

- **Respeta la idempotencia de cada verbo (y apóyate en ella)** — `GET`, `PUT` y `DELETE` deben poder repetirse sin cambiar el resultado; `POST` no. Esto no es teoría: los proxies y las librerías HTTP reintentan automáticamente los verbos idempotentes cuando la red falla. Un `PUT` que "suma 1" en vez de "fija el valor" duplicará efectos en cuanto haya un reintento. Para un `POST` crítico (un cobro), acepta una cabecera `Idempotency-Key` y devuelve la misma respuesta si llega repetido.
- **Nunca devuelvas una colección sin paginar** — `GET /api/products` sin límite funciona en desarrollo con 20 filas y tumba producción con 2 millones. Y si los datos cambian mientras se pagina, la paginación por `offset` salta o repite elementos; la paginación por **cursor** (`?after=abc123`) es estable. Añadirla después es un cambio que rompe clientes: hazlo desde el primer día.
- **Modela las acciones como recursos, no como verbos** — cuando algo no encaja en el CRUD (cancelar un pedido), la tentación es `POST /api/orders/42/cancel`, estilo RPC. La versión que escala es tratar la acción como una *cosa*: `POST /api/orders/42/cancellation` crea una cancelación, que luego puedes consultar (`GET`) con su estado y su fecha. El truco de convertir verbos en sustantivos resuelve el 90 % de los casos "esto no es CRUD".
- **Aprovecha la caché HTTP con `ETag`** — es la ventaja más desaprovechada de REST. El servidor devuelve un `ETag` (huella de la respuesta); el cliente lo reenvía en `If-None-Match` y, si nada cambió, recibe un `304 Not Modified` sin cuerpo. Gratis para el ancho de banda y para tu base de datos. El mismo mecanismo, con `If-Match`, evita que dos usuarios editando a la vez se pisen (*optimistic locking*).
- **Distingue `400`, `404`, `409` y `422` con intención** — devolver `400` para todo obliga al cliente a parsear mensajes. Un `404` en un `POST` a una colección existente, o un `200` con `{ "error": ... }` dentro, rompen la promesa central de REST: que el código de estado ya cuenta qué pasó sin abrir el cuerpo.

---

*En resumen: REST trata tu API como una biblioteca de recursos con URL fija, manejados con los verbos de HTTP —predecible, sencillo y el estándar de facto de la web.*
