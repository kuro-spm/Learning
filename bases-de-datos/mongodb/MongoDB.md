# MongoDB

## ¿Qué es?

MongoDB es una base de datos **NoSQL orientada a documentos**. En lugar de guardar los datos en filas de tablas con columnas fijas, los guarda como **documentos** parecidos a JSON (internamente en un formato binario llamado BSON), agrupados en **colecciones**. Cada documento puede tener su propia forma: no hay un esquema rígido impuesto de antemano.

## ¿Por qué existe?

El modelo relacional te obliga a definir el esquema por adelantado y a repartir un objeto entre varias tablas unidas por claves. Para datos que por naturaleza son jerárquicos o que cambian de forma (un producto con atributos que varían según la categoría, un pedido con sus líneas dentro), eso genera fricción: muchos `JOIN` y migraciones cada vez que cambia una columna.

MongoDB guarda el objeto entero como un solo documento, con sus datos anidados incluidos. Lees y escribes la "cosa completa" de una vez, sin `JOIN`, y el esquema puede evolucionar sin migrar toda la tabla.

La equivalencia con lo que ya conoces de SQL es casi directa:

| Relacional (SQL) | MongoDB |
|---|---|
| Base de datos | Base de datos |
| Tabla | Colección |
| Fila | Documento |
| Columna | Campo |
| Clave primaria | Campo `_id` (automático) |
| `JOIN` | Anidar el dato dentro (o `$lookup`) |

## ¿Cuándo y para qué se usa?

Encaja cuando los datos son flexibles o jerárquicos: el catálogo de una tienda online donde cada tipo de producto tiene atributos distintos, el contenido de un blog o CMS, perfiles de usuario con campos que cambian, o registros de eventos y logs que llegan en gran volumen. También es cómoda para prototipar rápido, cuando el esquema aún no está cerrado.

Es peor elección cuando los datos son muy relacionales y necesitas transacciones complejas entre muchas entidades (un sistema contable, por ejemplo): ahí una base de datos relacional sigue siendo más natural.

## Lo mínimo que necesitas saber

**1. Los datos son documentos dentro de colecciones**

Un documento es un objeto tipo JSON; puede contener arrays y otros objetos anidados. Este vive en la colección `products`:

```js
{
  _id: ObjectId("650f..."),
  name: "Teclado mecánico",
  price: 79.99,
  tags: ["periféricos", "gaming"],
  stock: { warehouse: 120, store: 8 }   // objeto anidado, sin tabla aparte
}
```

**2. CRUD básico**

```js
db.products.insertOne({ name: "Teclado", price: 79.99 });
db.products.find({ price: { $lt: 100 } });          // WHERE price < 100
db.products.updateOne({ _id: id }, { $set: { price: 69.99 } });
db.products.deleteOne({ _id: id });
```

**3. Consultas con operadores**

Los filtros se construyen con operadores como `$gte`, `$lte`, `$in`, `$and`:

```js
db.products.find({
  price: { $gte: 50, $lte: 100 },
  tags: { $in: ["gaming"] }
});
```

**4. Modelado: anidar (embed) vs referenciar**

La decisión de diseño clave. **Anida** los datos que se leen siempre juntos (las líneas de un pedido dentro del pedido); **referencia** por `_id` los datos compartidos o que crecen mucho (un usuario al que apuntan muchos pedidos). La regla mental: *los datos que se leen juntos, viven juntos*.

```js
// Anidado: las líneas viven dentro del pedido
{ _id: 1, customer: "Ada", lines: [{ product: "Teclado", qty: 2 }] }
```

**5. Índices**

Sin índice, una búsqueda recorre toda la colección (como un *full scan* en SQL). Un índice sobre el campo por el que filtras lo acelera:

```js
db.products.createIndex({ name: 1 });   // 1 = ascendente
```

**6. Aggregation pipeline (agrupar y transformar)**

Para agregar datos (contar, sumar, agrupar) se encadena una tubería de etapas. Es el equivalente a `GROUP BY` / `SUM`, por pasos:

```js
db.orders.aggregate([
  { $match: { status: "paid" } },                                   // WHERE
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } }    // GROUP BY + SUM
]);
```

## Lo que NO hace

- **No habla SQL ni hace `JOIN` idiomáticos** — existe `$lookup`, pero unir colecciones constantemente es señal de que el modelo debería replantearse (anidando o duplicando datos).
- **No impone un esquema por defecto** — puedes meter documentos de cualquier forma en una colección; la coherencia la garantizas tú.
- **No es la mejor opción para datos muy relacionales** — informes complejos con muchos cruces siguen siendo terreno de las bases relacionales.
- **No garantiza integridad referencial** — no hay claves foráneas que impidan referenciar un `_id` que ya no existe.

## Buenas prácticas avanzadas

- **Modela según tus consultas, no según tus entidades** — es lo contrario de SQL. En relacional normalizas y luego consultas; en MongoDB diseñas el documento para que la consulta más frecuente se resuelva con un solo `find`, sin `$lookup`. Empezar normalizando "por costumbre de SQL" es el error número uno y lleva a un modelo lento y antinatural.
- **Vigila los arrays que crecen sin límite** — anidar los comentarios de un post está bien; anidar los eventos de un log que crece para siempre hincha el documento hasta el límite de 16 MB y degrada el rendimiento. Lo que crece sin techo va en su propia colección referenciada.
- **Los índices compuestos siguen la regla ESR** — al crear un índice sobre varios campos, ordénalos como *Equality, Sort, Range*: primero los campos de igualdad, luego los de ordenación, y al final los de rango. Un orden equivocado hace que el índice no se aproveche aunque exista.
- **Trae solo los campos que necesitas (projection)** — pasa un segundo argumento a `find` (`{ name: 1, price: 1 }`) para no arrastrar documentos enteros por la red y la RAM "por si acaso".
- **Define validación de esquema aunque sea *schemaless*** — que MongoDB permita cualquier forma no significa que debas dejarlo suelto: un validador `$jsonSchema` en la colección rechaza documentos mal formados. La flexibilidad sin ninguna barrera acaba en datos inconsistentes que nadie limpia.

## Recursos didácticos

**MongoDB University** (<https://learn.mongodb.com/>) ofrece cursos gratuitos y oficiales muy buenos para empezar. Para trastear sin instalar nada, **MongoDB Atlas** tiene un clúster gratis en la nube, y su interfaz **Compass** deja explorar los documentos visualmente. Y la propia shell interactiva `mongosh` es la forma más rápida de probar consultas contra una colección de juguete.

---

*En resumen: MongoDB guarda objetos tipo JSON completos como documentos flexibles en colecciones, en vez de repartirlos en tablas con esquema fijo — modelas según cómo vas a leer los datos y te ahorras los `JOIN`, a cambio de garantizar tú mismo la coherencia.*
