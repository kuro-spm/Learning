# MongoDB

## ¿Qué es?

MongoDB es una base de datos **NoSQL orientada a documentos**: en lugar de repartir un objeto entre filas de varias tablas, guarda el objeto entero —con sus arrays y sus subobjetos dentro— como un único **documento** parecido a JSON (internamente BSON, su versión binaria con tipos). Los documentos se agrupan en **colecciones**, y cada uno puede tener su propia forma.

## ¿Por qué existe?

En el modelo relacional, un pedido no existe como tal en ninguna parte: existe una fila en `pedidos`, tres en `lineas_pedido`, una en `clientes` y otra en `direcciones`. Para verlo entero hay que reconstruirlo con `JOIN` cada vez, y para cambiarle un campo hay que migrar una tabla que quizá tenga millones de filas.

> Una base relacional es un archivador de fichas: cada dato en su cajón, ordenadísimo, y para consultar un expediente vas abriendo cajones y cruzando referencias. MongoDB es una estantería de carpetas: el expediente del pedido #4711 está entero en su carpeta, lo sacas de una vez y lo lees. A cambio, si el nombre del cliente aparece en cincuenta carpetas y cambia, te toca abrirlas todas.

Esa es toda la negociación. Ganas lecturas de una sola parada y un esquema que evoluciona sin migraciones dolorosas; pierdes la normalización, la integridad referencial garantizada y los `JOIN` baratos. **En MongoDB no hay joins baratos**: existe `$lookup`, pero es una operación de agregación que no se parece en rendimiento a un `INNER JOIN` con índices en PostgreSQL. Si tu diseño depende de cruzar colecciones constantemente, el diseño está mal, no la base de datos.

## ¿Cuándo y para qué se usa?

Encaja cuando los datos son jerárquicos o de forma variable y se leen como una unidad: el catálogo de una tienda donde cada categoría tiene atributos distintos (una camiseta tiene talla, un portátil tiene RAM), documentos de contenido, perfiles, eventos y telemetría en gran volumen, o el histórico de un pedido con todas sus líneas. También cuando el esquema aún no está cerrado y quieres iterar sin escribir una migración por cada campo nuevo.

## El vocabulario, traducido

Casi todo lo que sabes de SQL tiene equivalente directo:

| Relacional (SQL) | MongoDB | Nota |
|---|---|---|
| Tabla | Colección | No declara columnas |
| Fila | Documento | Máximo 16 MB cada uno |
| Columna | Campo | Puede faltar en unos documentos y estar en otros |
| Clave primaria | `_id` | Se genera solo (`ObjectId`) si no lo pones |
| Clave foránea | Campo con un `_id` | **Sin integridad referencial**: nada impide apuntar a algo borrado |
| `JOIN` | Incrustar el dato, o `$lookup` | Incrustar es lo idiomático |
| `WHERE` | Filtro del `find` / `$match` | |
| `SELECT col1, col2` | *Projection* | Segundo argumento del `find` |
| `GROUP BY` + `SUM` | `$group` con `$sum` | Dentro del aggregation pipeline |
| `ALTER TABLE ADD COLUMN` | Nada: escribes el campo | Los documentos viejos simplemente no lo tienen |

Ese último punto es el que más cuesta interiorizar: añadir un campo no es una operación de esquema. El coste no desaparece, se traslada al código, que tiene que saber tratar documentos de dos generaciones distintas.

## La decisión central: incrustar o referenciar

Es el 80 % del diseño de una base documental y donde más se equivoca quien viene de SQL, porque el instinto de normalizar juega en contra. **Incrustar** (*embed*) es meter el subobjeto dentro del documento padre; **referenciar** es guardar solo su `_id`. Así queda el pedido #4711 en la colección `pedidos`:

```js
{
  _id: ObjectId("6634a1f0c8e4b2001f3a7d11"),
  numero: 4711,
  fecha: ISODate("2026-03-14T10:22:00Z"),
  estado: "pagado",
  cliente: { _id: ObjectId("6612b0aa9d1c4400aa10ff02"),   // incrustado, pero con su _id
             nombre: "Ada Lovelace", email: "ada@example.com" },
  envio: { ciudad: "Valencia", cp: "46001" },
  lineas: [                                               // en SQL serían lineas_pedido
    { productoId: ObjectId("65f0aa11bb22cc33dd44ee55"), nombre: "Teclado mecánico", cantidad: 2, precio: NumberDecimal("79.99") },
    { productoId: ObjectId("65f0aa11bb22cc33dd44ee77"), nombre: "Ratón vertical",   cantidad: 1, precio: NumberDecimal("34.50") }
  ],
  total: NumberDecimal("194.48")
}
```

Hay tres decisiones tomadas ahí: las líneas van incrustadas (nunca se consultan sin su pedido), el cliente va incrustado con su `_id` para poder ir a `clientes` si hace falta el resto de la ficha, y el producto va referenciado, con nombre y precio copiados como fotografía del momento de la compra. Los criterios que llevan a una u otra:

| Criterio | Incrustar | Referenciar |
|---|---|---|
| Cardinalidad | 1:1 y 1:pocos (decenas) | 1:muchos sin techo, y N:M |
| ¿Se consulta el hijo por su cuenta? | No, siempre con el padre | Sí, tiene vida propia |
| Frecuencia de cambio del hijo | Baja o irrelevante (es una foto) | Alta: cambia y debe verse al día en todas partes |
| Tamaño | El documento entero cabe holgado en 16 MB | El array crecería sin límite |
| Coste de lectura | Una sola parada, sin `$lookup` | Dos consultas o una agregación |

Por eso el pedido apunta al cliente y no al revés:

```js
// ❌ Pedidos INCRUSTADOS en el cliente: un cliente fiel acumula miles y revienta los 16 MB
{ _id: ..., nombre: "Ada Lovelace", pedidos: [ /* ...y creciendo para siempre */ ] }

// ✅ Pedidos REFERENCIADOS: cada pedido guarda a quién pertenece
{ numero: 4711, cliente: { _id: ObjectId("6612b0..."), nombre: "Ada Lovelace" } }
```

La regla mental es **"lo que se lee junto, vive junto"**, con dos matices que evitan casi todos los desastres:

1. **Un array sin techo conocido no se incrusta jamás.** Si no puedes responder "¿cuántos elementos como máximo?", va en su propia colección. El límite de 16 MB por documento no es un consejo: es un error de escritura cuando lo superas.
2. **Duplicar datos está bien, pero decide si la copia es una foto o una caché.** El `precio` de la línea es una **foto**: si mañana sube el teclado, el pedido #4711 debe seguir diciendo 79,99 €. El `nombre` del cliente es una **caché**: si Ada cambia de apellido, alguien tiene que propagarlo. Confundir las dos cosas es el origen de la mayoría de las incoherencias.

## Consultas: `find`, proyecciones y operadores

Un `find` recibe un filtro y, opcionalmente, una proyección con los campos que quieres de vuelta:

```js
db.pedidos.find(
  { estado: "pagado", total: { $gte: NumberDecimal("100") },
    "envio.ciudad": { $in: ["Valencia", "Alicante"] } },   // notación de punto para anidados
  { numero: 1, total: 1, "cliente.nombre": 1, _id: 0 }     // 1 = traer, 0 = omitir
)
```

Es el equivalente a `SELECT numero, total, ... WHERE estado = 'pagado' AND total >= 100 AND ciudad IN (...)`, y devuelve documentos recortados:

```js
{ numero: 4711, total: NumberDecimal("194.48"), cliente: { nombre: "Ada Lovelace" } }
```

Proyectar no es cosmético: sin ella arrastras el documento entero —líneas incluidas— por la red y la RAM del servidor. Y con arrays hay una trampa clásica, porque estas dos consultas **no** son iguales:

```js
// ❌ Cada condición puede cumplirse en una línea DISTINTA: casa un pedido con
//    un teclado (cantidad 1) y un ratón (cantidad 3)
db.pedidos.find({ "lineas.nombre": "Teclado mecánico", "lineas.cantidad": { $gte: 2 } })

// ✅ $elemMatch exige que UNA MISMA línea cumpla ambas condiciones
db.pedidos.find({ lineas: { $elemMatch: { nombre: "Teclado mecánico", cantidad: { $gte: 2 } } } })
```

Desde C# con el driver oficial `MongoDB.Driver`, lo mismo se escribe con `Builders<T>` (ver [acceso a datos en .NET](../acceso-a-datos-dotnet/README.md) para el panorama completo):

```csharp
var pedidos = database.GetCollection<Pedido>("pedidos");

var filtro = Builders<Pedido>.Filter.And(
    Builders<Pedido>.Filter.Eq(p => p.Estado, "pagado"),
    Builders<Pedido>.Filter.Gte(p => p.Total, 100m),
    Builders<Pedido>.Filter.ElemMatch(p => p.Lineas, l => l.Cantidad >= 2));

var resumen = await pedidos.Find(filtro)
    .Project(p => new { p.Numero, p.Total })   // se traduce a projection del servidor
    .ToListAsync();
```

El driver serializa las clases de C# a BSON solo; `[BsonId]` marca el `_id` y `[BsonElement("nombre")]` mapea nombres distintos.

## Actualizaciones: `$set`, `$inc`, `$push` y `upsert`

Una actualización nunca reemplaza el documento entero salvo que se lo pidas: se describe con **operadores** que tocan solo los campos indicados, de forma atómica sobre ese documento.

```js
db.pedidos.updateOne(
  { numero: 4711 },
  {
    $set:  { estado: "enviado", "envio.seguimiento": "ES-9981-TRK" },  // fija valores
    $inc:  { intentosEnvio: 1 },                                       // suma (o resta con -1)
    $push: { historial: { estado: "enviado", fecha: new Date() } }     // añade al array
  }
)
```

Devuelve un recibo con lo que ha pasado:

```js
{ acknowledged: true, matchedCount: 1, modifiedCount: 1, upsertedId: null }
```

`matchedCount: 0` significa que el filtro no encontró nada; `modifiedCount: 0` con `matchedCount: 1`, que lo encontró pero los valores ya eran esos. En C# es la misma operación con `Builders<T>.Update`:

```csharp
var update = Builders<Pedido>.Update
    .Set(p => p.Estado, "enviado")
    .Inc(p => p.IntentosEnvio, 1)
    .Push(p => p.Historial, new Evento("enviado", DateTime.UtcNow));

var res = await pedidos.UpdateOneAsync(p => p.Numero == 4711, update);   // res.ModifiedCount == 1
```

Con `upsert: true`, si el filtro no casa con nada MongoDB **inserta** el documento en lugar de no hacer nada, y `$setOnInsert` aplica solo en ese caso:

```js
db.productos.updateOne(
  { sku: "TEC-MEC-87" },
  { $inc: { stock: -2 },
    $setOnInsert: { nombre: "Teclado mecánico", creado: new Date() } },
  { upsert: true }
)
```

Es el `INSERT ... ON CONFLICT DO UPDATE` de PostgreSQL: resuelve en una sola operación atómica el clásico "mira si existe, y si no créalo" que hecho en dos pasos sufre condiciones de carrera.

## El aggregation pipeline

Para todo lo que en SQL sería `GROUP BY`, `HAVING`, subconsultas o `JOIN`, MongoDB usa una **tubería de etapas**: cada una recibe documentos, los transforma y se los pasa a la siguiente. Los cinco productos más facturados del año:

```js
db.pedidos.aggregate([
  { $match: { estado: "pagado", fecha: { $gte: ISODate("2026-01-01") } } },   // 1
  { $unwind: "$lineas" },                                                     // 2
  { $group: {                                                                 // 3
      _id: "$lineas.productoId",
      unidades:  { $sum: "$lineas.cantidad" },
      facturado: { $sum: { $multiply: ["$lineas.cantidad", "$lineas.precio"] } }
  }},
  { $sort: { facturado: -1 } },                                               // 4
  { $limit: 5 },                                                              // 5
  { $lookup: { from: "productos", localField: "_id",                          // 6
               foreignField: "_id", as: "producto" } },
  { $project: { _id: 0, unidades: 1, facturado: 1,
                nombre: { $first: "$producto.nombre" } } }
])
```

Paso a paso: (1) se queda con los pedidos pagados de este año, (2) `$unwind` convierte cada línea del array en un documento independiente —como pasar de `pedidos` a `lineas_pedido`—, (3) agrupa por producto sumando unidades e importe, (4) y (5) ordena y recorta, (6) va a buscar el nombre a `productos` y (7) da forma a la salida:

```js
{ unidades: 312, facturado: NumberDecimal("24956.88"), nombre: "Teclado mecánico" }
{ unidades: 190, facturado: NumberDecimal("6555.00"),  nombre: "Ratón vertical" }
```

**Por qué `$match` va primero**, y no es un detalle de estilo: solo una etapa `$match` situada al principio puede aprovechar un **índice**. En cuanto se ejecuta un `$group` o un `$unwind`, lo que circula por la tubería son documentos intermedios en memoria, sobre los que no hay índice posible; un `$match` colocado ahí filtra a fuerza bruta. Además reduce el volumen que entra en las etapas caras: filtrar de 500 000 pedidos a 8 000 antes del `$unwind` cambia el tiempo de ejecución en un orden de magnitud.

El mismo razonamiento vale para `$limit` antes de `$lookup`: hacer la búsqueda cruzada sobre 5 documentos en vez de sobre 4 000 es la diferencia entre milisegundos y segundos. `$lookup` es la etapa más cara del catálogo; úsala al final, sobre pocos documentos, y solo para adornar resultados ya calculados.

## Índices y la regla ESR

Sin índice, una consulta recorre la colección entera (`COLLSCAN`, el *full table scan* de siempre). Un índice compuesto cubre varios campos, y **el orden en que los declaras decide si sirve o no**:

```js
db.pedidos.createIndex({ estado: 1, fecha: -1, total: 1 })
//                       ^Equality  ^Sort     ^Range
```

La regla **ESR** dice que los campos van en este orden:

- **E**quality — los que se comparan por igualdad (`estado: "pagado"`). Van primero porque acotan el índice a un tramo contiguo.
- **S**ort — los que ordenan (`sort({ fecha: -1 })`). Si van aquí, el resultado ya sale ordenado del índice y se evita una ordenación en memoria (que además falla si supera los 100 MB).
- **R**ange — los de rango (`total: { $gte: 100 }`). Van al final porque a partir del primer rango el índice deja de poder acotar los campos siguientes.

Para comprobar si un índice se está usando, `explain("executionStats")`:

```js
db.pedidos.find({ estado: "pagado", total: { $gte: NumberDecimal("100") } })
          .sort({ fecha: -1 })
          .explain("executionStats")
```

Lo que hay que mirar son tres números:

```
❌ "stage": "COLLSCAN",  "totalDocsExamined": 128430,  "nReturned": 87
✅ "stage": "IXSCAN",    "totalDocsExamined": 87,      "nReturned": 87
```

Si `totalDocsExamined` se parece a `nReturned`, el índice está haciendo su trabajo. Si lo multiplica por mil, o falta el índice o el orden de sus campos no respeta ESR. La pestaña *Explain Plan* de MongoDB Compass enseña lo mismo en forma de árbol, que se lee mucho mejor.

## Esquema flexible ≠ sin esquema

Que MongoDB no exija un esquema no significa que tu aplicación no lo tenga: lo tiene, solo que vive implícito en el código. Puedes hacerlo explícito con un validador `$jsonSchema` en la colección:

```js
db.createCollection("pedidos", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["numero", "estado", "lineas", "total"],
      properties: {
        numero: { bsonType: "int" },
        estado: { enum: ["pendiente", "pagado", "enviado", "cancelado"] },
        total:  { bsonType: "decimal", minimum: 0 },
        lineas: { bsonType: "array", minItems: 1, items: {
          bsonType: "object",
          required: ["productoId", "cantidad", "precio"],
          properties: { cantidad: { bsonType: "int", minimum: 1 } }
        }}
      }
    }
  },
  validationAction: "error"       // "warn" solo lo registra en el log
})
```

Un `insertOne` con `estado: "pagao"` o sin líneas falla en el servidor:

```
MongoServerError: Document failed validation
```

Sobre una colección que ya existe se aplica con `db.runCommand({ collMod: "pedidos", validator: {...} })`. Para no romper los documentos antiguos, `validationLevel: "moderate"` valida solo las inserciones y las actualizaciones de documentos que ya cumplían. Esa convivencia de versiones —añadir un campo, rellenarlo poco a poco, hacerlo obligatorio al final— es exactamente la misma disciplina que en [migraciones de esquema](../migraciones-de-esquema/README.md) relacionales, aunque aquí la ejecute tu código en vez de un `ALTER TABLE`.

## Transacciones: existen, pero suelen ser una señal

Una escritura sobre **un** documento siempre ha sido atómica, incluso si toca veinte campos anidados. Desde la versión 4.0 hay además transacciones **multi-documento** con ACID completo, y desde la 4.2 abarcan varias colecciones (requieren *replica set*, no funcionan en un servidor suelto):

```csharp
using var session = await client.StartSessionAsync();

await session.WithTransactionAsync(async (s, ct) =>
{
    await pedidos.InsertOneAsync(s, nuevoPedido, cancellationToken: ct);
    await productos.UpdateOneAsync(s,
        p => p.Sku == "TEC-MEC-87",
        Builders<Producto>.Update.Inc(p => p.Stock, -2),
        cancellationToken: ct);
    return true;
});
```

Funcionan bien y son la respuesta correcta para casos puntuales como ese. Pero si te descubres envolviendo en transacciones la mayoría de tus operaciones, el mensaje que te está mandando el diseño es que has modelado en tablas y lo has guardado en colecciones: los datos que deben cambiar juntos y de forma consistente son, casi por definición, los que deberían vivir en el mismo documento. Antes de añadir una transacción, pregunta si el problema se disuelve incrustando.

## Cuándo NO elegir MongoDB

Descártalo cuando:

- **Necesitas cruzar entidades constantemente.** Informes con cinco `JOIN`, cuadros de mando analíticos, cualquier cosa donde las consultas no se conocen de antemano y varían cada semana.
- **Las transacciones multi-documento son la norma.** Contabilidad, facturación, reservas con inventario compartido: dominios donde una escritura parcial es inaceptable.
- **La integridad referencial la debe garantizar el motor.** MongoDB no tiene `FOREIGN KEY`: nada impide que un `productoId` apunte a un producto borrado.

En esos casos, [PostgreSQL](../postgresql/PostgreSQL.md) con columnas `jsonb` te da lo mejor de ambos: tablas normalizadas con claves foráneas y transacciones de verdad, más un campo flexible para los atributos variables.

```sql
CREATE TABLE productos (
  id         bigserial PRIMARY KEY,
  sku        text NOT NULL UNIQUE,
  nombre     text NOT NULL,
  atributos  jsonb NOT NULL DEFAULT '{}'   -- talla, RAM, color... lo que varíe
);
CREATE INDEX idx_prod_attr ON productos USING gin (atributos);
```

Esa tabla admite `WHERE atributos->>'color' = 'negro'` con índice, y a la vez `JOIN` y `FOREIGN KEY`. Para muchos proyectos que empiezan pensando en MongoDB "por la flexibilidad", esta es la opción que menos duele a los dos años.

## Errores frecuentes

| Síntoma | Causa probable |
|---|---|
| Consulta lentísima que antes iba bien | No hay índice, o hay un `$sort` que no encaja con él. Confírmalo con `explain()`: busca `COLLSCAN` |
| `BSONObjectTooLarge` al actualizar | Un array incrustado ha crecido sin techo y el documento roza los 16 MB. Sácalo a su propia colección |
| El índice existe pero no se usa | El orden de sus campos no sigue ESR, o filtras por un campo que no es prefijo del índice compuesto |
| Un `find` sobre un array devuelve documentos de más | Condiciones sueltas en vez de `$elemMatch`: cada una casa con un elemento distinto |
| El nombre del cliente aparece desactualizado en pedidos viejos | Duplicación sin plan de propagación; decide si esa copia era foto o caché |
| `$lookup` deja la agregación en varios segundos | Está antes de `$match`/`$limit` y cruza miles de documentos en vez de decenas |
| Totales que no cuadran por céntimos | Importes guardados como `double` en vez de `Decimal128` |
| Documentos incoherentes que nadie sabe de dónde salen | Colección sin validador `$jsonSchema` |

## Buenas prácticas avanzadas

- **Escribe las cinco consultas más frecuentes antes que el esquema.** En SQL normalizas primero y consultas después; aquí es al revés. Anota literalmente "listar pedidos de un cliente por fecha", "top de productos del mes", y diseña el documento para que cada una se resuelva con un `find` y un índice. Empezar normalizando por costumbre es el error número uno y produce modelos que necesitan `$lookup` para todo.
- **Ningún array incrustado sin un techo que puedas nombrar.** Antes de incrustar, responde "¿cuántos como máximo?". Si la respuesta es "depende", va referenciado. Para casos intermedios está el *bucket pattern*: agrupar eventos en documentos de, por ejemplo, un día o cien elementos, en lugar de uno por evento o todos en uno.
- **Audita los índices, no solo los crees.** `db.pedidos.aggregate([{ $indexStats: {} }])` dice cuántas veces se ha usado cada uno desde el último arranque; los que marcan `ops: 0` ocupan RAM y ralentizan cada escritura sin dar nada a cambio. En el otro extremo, si un índice contiene todos los campos que devuelve la proyección, la consulta es *covered* y se resuelve sin tocar los documentos.
- **Dinero en `Decimal128`, tiempo en UTC.** Guarda importes con `NumberDecimal("79.99")` (`decimal` en C#), nunca `double`, o los totales acumularán error de coma flotante. Y almacena siempre fechas en UTC con `DateTime.UtcNow`, convirtiendo a zona local solo al presentar.
- **Prueba la restauración, no solo la copia.** `mongodump`/`mongorestore` sirven para volúmenes moderados; en réplicas grandes se usan instantáneas del sistema de ficheros con el *journal* incluido. Sea cual sea el método, una copia que nunca se ha restaurado no es una copia: programa una restauración real cada trimestre sobre una base de pruebas. Ver [copias de seguridad](../../devops/despliegue-en-vps/Copias-de-Seguridad.md).

## Recursos didácticos

- [MongoDB Playground](https://mongoplayground.net/) — un entorno en el navegador donde pegas documentos y una consulta o un pipeline y ves la salida al instante, sin instalar ni registrarte. Es la forma más rápida de entender qué hace `$unwind` o de depurar una agregación paso a paso.
- [MongoDB University](https://learn.mongodb.com/) — cursos oficiales gratuitos con laboratorios ejecutables; el itinerario de *Data Modeling* es justo la parte que no se aprende leyendo referencias de API.
- [Building with Patterns](https://www.mongodb.com/blog/post/building-with-patterns-a-summary) — catálogo de patrones de modelado documental (bucket, outlier, subset, extended reference) con el problema que resuelve cada uno. Cuando "incrustar o referenciar" se te queda corto, la respuesta suele tener nombre en esta lista.

---

*En resumen: MongoDB guarda cada objeto entero en un documento en vez de repartirlo en tablas, así que modelas según cómo vas a leer y no según cómo se descompone la entidad — la decisión de incrustar o referenciar es todo el diseño, y necesitar `$lookup` o transacciones a todas horas es la señal de que la has tomado mal.*
