# GraphQL

## ¿Qué es?

**GraphQL** es un lenguaje de consulta para APIs en el que el **cliente** decide exactamente qué datos quiere recibir, pidiéndolos en una sola consulta. En vez de muchos endpoints fijos, hay normalmente **un único endpoint** y un esquema que describe todo lo que puedes pedir.

## ¿Por qué existe?

En una API REST clásica, cada endpoint devuelve una forma fija de datos. Eso causa dos molestias:

- **Over-fetching**: pides un producto y recibes 30 campos cuando solo necesitabas el nombre y el precio.
- **Under-fetching**: para pintar una pantalla necesitas el usuario, sus pedidos y los productos de cada pedido, así que haces 3 o 4 llamadas encadenadas.

GraphQL le da la vuelta: el cliente describe la "forma" exacta de los datos que necesita y el servidor se la devuelve tal cual, en una sola petición.

> Piensa en un bufé frente a un menú cerrado. En REST te sirven el plato del día completo (te sobre o te falte). En GraphQL te pones en el plato exactamente lo que quieres comer, ni más ni menos.

## ¿Cuándo y para qué se usa?

Brilla cuando una pantalla necesita combinar muchos datos relacionados y distintos clientes (web, móvil) necesitan piezas diferentes de lo mismo. Por ejemplo, una app donde la versión móvil muestra solo nombre y foto, y la web muestra el perfil completo: ambas usan la misma API pidiendo campos distintos. Lo usan productos con datos muy interconectados (redes sociales, paneles de control, catálogos ricos).

## Lo mínimo que necesitas saber

**1. El esquema define lo que existe**

El servidor publica un *schema* con los tipos y sus campos. Es el contrato y la documentación a la vez.

```graphql
type Product {
  id: ID!
  name: String!
  price: Float!
  reviews: [Review!]!
}
```

**2. La consulta (*query*) pide solo lo que necesitas**

Tú escribes la forma; recibes esa forma exacta.

```graphql
query {
  product(id: 42) {
    name
    price
  }
}
```

```json
→ { "data": { "product": { "name": "Teclado", "price": 29.90 } } }
```

**3. Datos anidados en una sola petición**

Lo que en REST serían varias llamadas, aquí es una.

```graphql
query {
  user(id: 1) {
    name
    orders { id total }
  }
}
```

**4. Las mutaciones (*mutations*) cambian datos**

Las consultas leen; las mutaciones crean, actualizan o borran.

```graphql
mutation {
  createProduct(name: "Ratón", price: 19.90) {
    id
  }
}
```

**5. Un solo endpoint, normalmente por POST**

No hay decenas de URLs: todo entra por el mismo punto y el cuerpo de la petición lleva la consulta.

```http
POST /graphql
{ "query": "{ product(id: 42) { name price } }" }
```

## Lo que NO hace

- **No usa los códigos de estado HTTP como REST** — suele responder `200 OK` aunque haya errores; estos van dentro del cuerpo, en un campo `errors`.
- **No cachea tan fácil como REST** — al ir casi todo por `POST` a una sola URL, la caché HTTP estándar no ayuda igual.
- **No protege solo contra consultas abusivas** — un cliente puede pedir datos anidados muy pesados; hay que poner límites de complejidad.
- **No siempre compensa** — para una API CRUD sencilla, REST suele ser más simple y directo.

---

*En resumen: GraphQL es un bufé de datos donde el cliente pide en una sola consulta exactamente los campos que necesita —ideal cuando la información está muy interconectada y hay clientes con necesidades distintas.*
