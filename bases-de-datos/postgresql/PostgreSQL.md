# PostgreSQL

## ¿Qué es?

PostgreSQL (o "Postgres") es una base de datos **relacional** de código abierto: guarda los datos en tablas con filas y columnas y se consulta con SQL, como SQL Server o MySQL. Su seña de identidad es lo lejos que lleva ese modelo: cumple el estándar SQL con rigor y añade tipos de datos, índices y capacidades de concurrencia que la convierten en un motor todoterreno.

## ¿Por qué existe?

Todas las relacionales hacen lo mismo de base (tablas, `JOIN`, transacciones), así que la pregunta real no es "¿por qué una relacional?" sino "¿por qué *esta*?".

PostgreSQL nació en la universidad de Berkeley con la idea de ser una base de datos **extensible**: en lugar de un conjunto cerrado de funciones, un núcleo al que se le pueden añadir tipos de datos, operadores, métodos de indexación e incluso lenguajes de procedimiento. Décadas después eso se traduce en un motor fiel al estándar, gratuito y sin licencias por core, que trae de serie cosas que en otros motores son de pago o directamente no existen: JSON indexable, arrays como columna, búsqueda de texto completo, tipos de rango, tipos geográficos.

> Si vienes de MySQL o SQL Server, piensa en PostgreSQL como el mismo idioma con un vocabulario mucho más amplio. Lo que ya sabes sigue funcionando; lo que cambia es que hay una palabra exacta para casos en los que antes improvisabas.

## ¿Cuándo y para qué se usa?

Es la elección por defecto sensata para el almacén principal de casi cualquier aplicación. En esta guía el ejemplo conductor es una **tienda online** con tres tablas —`clientes`, `productos` y `pedidos`— y un pedido concreto, el **#4711**, que iremos siguiendo.

Brilla especialmente cuando los datos son relacionales pero algún campo es semiestructurado (los atributos variables de un producto), cuando necesitas búsqueda de texto sin montar un motor aparte, o cuando varios procesos compiten por las mismas filas y quieres repartirlas sin pisarse.

Es peor elección para caché volátil en memoria —ahí encaja mejor [Redis](../caching/README.md)— o para datos tan flexibles y sin relaciones que una base documental como [MongoDB](../mongodb/README.md) te ahorraría el modelado.

Este es el esquema que usaremos:

```sql
CREATE TABLE clientes (
    id     uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    email  text NOT NULL UNIQUE,
    alta   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE productos (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku        text NOT NULL UNIQUE,
    nombre     text NOT NULL,
    precio     numeric(10,2) NOT NULL,
    etiquetas  text[] NOT NULL DEFAULT '{}',
    atributos  jsonb NOT NULL DEFAULT '{}'
);

CREATE TABLE pedidos (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cliente_id  uuid NOT NULL REFERENCES clientes(id),
    estado      text NOT NULL,
    total       numeric(10,2) NOT NULL,
    creado_en   timestamptz NOT NULL DEFAULT now()
);
```

## Tipos de datos que cambian cómo diseñas

Elegir el tipo en Postgres no es un detalle cosmético: condiciona qué índices podrás usar y qué bugs tendrás dentro de seis meses.

| Tipo | Cuándo usarlo | Alternativa que se elige por inercia |
|---|---|---|
| `text` | Cualquier cadena | `varchar(n)` — mismo rendimiento, límite arbitrario |
| `timestamptz` | Cualquier fecha-hora | `timestamp` — sin zona, ambiguo |
| `numeric(10,2)` | Dinero y cantidades contables | `float` / `real` — redondeo binario |
| `uuid` | Identificadores públicos | `text` de 36 caracteres — 2× de espacio |
| `text[]` | Lista corta y cerrada de etiquetas | Tabla puente cuando no hace falta |
| `jsonb` | Atributos genuinamente variables | `json` — sin índices útiles |
| `daterange` | Periodos con inicio y fin | Dos columnas sueltas sin validar |

**`text` frente a `varchar(n)`.** En Postgres son el mismo tipo por dentro; `varchar(n)` solo añade una comprobación de longitud. No es más rápido ni ocupa menos.

```sql
SELECT pg_column_size('Zapatillas'::text), pg_column_size('Zapatillas'::varchar(50));
```

```
 pg_column_size | pg_column_size
----------------+----------------
             14 |             14
```

Idéntico. Y ampliar el límite (`varchar(50)` → `varchar(100)`) es un `ALTER TABLE` que en versiones antiguas reescribía la tabla entera. Usa `text` y valida la longitud con un `CHECK` si de verdad importa.

**`timestamptz`, siempre.** `timestamptz` guarda el instante normalizado a UTC y lo devuelve en la zona del cliente; `timestamp` guarda un número que *parece* una hora pero no dice de dónde.

```sql
SET timezone = 'Europe/Madrid';
SELECT creado_en FROM pedidos WHERE id = 4711;
```

```
       creado_en
------------------------
 2026-03-14 18:22:41+01
```

Ese `+01` es la diferencia. Sin él, el mismo dato leído desde un servidor en otra zona significa otra cosa.

**Arrays.** Una columna puede contener una lista, con operadores propios (`@>` contiene, `&&` se solapa):

```sql
SELECT nombre FROM productos WHERE etiquetas @> ARRAY['rebajas'];
```

Útil para etiquetas que solo lees y filtras. Si necesitas contar por etiqueta, ordenar por popularidad o relacionarlas con otra tabla, haz una tabla puente: un array no tiene claves foráneas.

**Rangos.** `daterange`, `tstzrange` y compañía guardan un intervalo como un valor único y saben si se solapan: `SELECT daterange('2026-03-01','2026-03-15') && daterange('2026-03-10','2026-03-20')` devuelve `t`. Combinado con una restricción `EXCLUDE`, esto impide a nivel de base de datos que dos promociones del mismo producto se pisen en el tiempo — algo que con dos columnas `fecha_inicio`/`fecha_fin` solo puedes comprobar desde la aplicación, y con carreras.

## `jsonb`: lo flexible dentro de lo relacional

Un producto puede tener atributos que no comparte con los demás: talla en ropa, potencia en electrodomésticos. Ese es el caso legítimo de `jsonb`.

```sql
UPDATE productos SET atributos = '{"color": "rojo", "talla": 42}' WHERE sku = 'ZAP-42';

SELECT nombre, atributos ->> 'color' AS color
FROM productos
WHERE atributos @> '{"color": "rojo"}';
```

```
       nombre       | color
--------------------+-------
 Zapatillas running | rojo
```

`->` devuelve JSON, `->>` devuelve texto, y `@>` pregunta "¿contiene este fragmento?". El tercero es el importante: es el único de los tres que puede aprovechar un índice GIN.

**Por qué `jsonb` y no `json`.** El tipo `json` guarda el texto literal tal cual llegó: conserva espacios, orden de claves y duplicados, y lo reparsea en cada consulta. `jsonb` lo almacena descompuesto en forma binaria.

| | `json` | `jsonb` |
|---|---|---|
| Almacenamiento | Texto literal | Binario descompuesto |
| Coste por lectura | Reparsea siempre | Ya parseado |
| Índices GIN | ❌ No | ✅ Sí |
| Claves duplicadas | Las conserva | Se queda con la última |
| Orden de claves | El original | Reordenado |
| Escritura | Marginalmente más rápida | Marginalmente más lenta |

Se ve al vuelo: `SELECT '{"b":1, "a":2, "a":3}'::jsonb` devuelve `{"a": 3, "b": 1}` —clave duplicada resuelta y claves reordenadas—, mientras que el mismo literal como `json` sale tal cual entró. Salvo que necesites reproducir el documento byte a byte (una firma digital, un log de auditoría), usa `jsonb`.

Y una advertencia de diseño: `jsonb` no valida nada. Lo que consultas, filtras y relacionas de verdad merece su propia columna con su tipo y su clave foránea. Meter `cliente_id` dentro de un `jsonb` es renunciar a la integridad referencial que justifica usar una relacional.

## Identidad: `GENERATED ALWAYS AS IDENTITY` frente a `serial`

`serial` fue durante años la forma de tener un entero autoincremental. Sigue funcionando, pero es una macro: crea una secuencia y le pone un `DEFAULT` a la columna. Eso deja dos agujeros.

```sql
-- ❌ Con serial: nada impide saltarse la secuencia
CREATE TABLE pedidos_viejo (id serial PRIMARY KEY, total numeric(10,2));
INSERT INTO pedidos_viejo (id, total) VALUES (4711, 99.00);   -- pasa sin protestar
```

La secuencia sigue en 1, así que el siguiente `INSERT` sin `id` intentará usar el 1, luego el 2… y al llegar a 4711 explotará con una violación de clave primaria, meses después y sin relación aparente con la causa.

```sql
-- ✅ Con IDENTITY: la columna la controla la base de datos
INSERT INTO pedidos (id, cliente_id, estado, total) VALUES (4711, ...);
```

```
ERROR:  cannot insert a non-DEFAULT value into column "id"
HINT:  Use OVERRIDING SYSTEM VALUE to override this restriction.
```

`GENERATED ALWAYS AS IDENTITY` es sintaxis estándar, bloquea el error de raíz y permite saltárselo explícitamente (`OVERRIDING SYSTEM VALUE`) cuando migras datos y de verdad lo necesitas. La variante `BY DEFAULT` se comporta como `serial` pero con la secuencia correctamente asociada a la columna.

Un detalle que sorprende: **las secuencias no son transaccionales**. Si insertas el pedido y haces `ROLLBACK`, el número consumido no vuelve. Los huecos en los IDs son normales y no indican que falten datos.

## `UPSERT`: `INSERT ... ON CONFLICT`

Sincronizar un catálogo significa "inserta si no existe, actualiza si existe". Hacerlo con un `SELECT` previo es una carrera: entre la comprobación y el `INSERT`, otro proceso puede haber insertado la misma fila.

```sql
INSERT INTO productos (sku, nombre, precio)
VALUES ('ZAP-42', 'Zapatillas running', 79.90)
ON CONFLICT (sku) DO UPDATE
    SET nombre = EXCLUDED.nombre,
        precio = EXCLUDED.precio
RETURNING id, (xmax = 0) AS insertado;
```

```
 id  | insertado
-----+-----------
 128 | f
```

`EXCLUDED` es la fila que se intentaba insertar. `RETURNING` devuelve el resultado sin una segunda consulta, y el truco de `xmax = 0` distingue si la fila se insertó (`t`) o se actualizó (`f`).

La cláusula `ON CONFLICT` necesita un índice único sobre la columna del conflicto — aquí lo da `sku text UNIQUE`. Si solo quieres ignorar el duplicado, `ON CONFLICT (sku) DO NOTHING`; ojo, en ese caso `RETURNING` no devuelve nada para las filas ignoradas.

## Leer un plan con `EXPLAIN ANALYZE`

Antes de añadir índices a ciegas, mira cómo ejecuta Postgres la consulta. `EXPLAIN` muestra el plan estimado; `EXPLAIN ANALYZE` **la ejecuta de verdad** y añade los tiempos y las filas reales.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total FROM pedidos
WHERE cliente_id = '3f2a91c4-8b7d-4e11-9f03-2c5a7e6d1b88' AND estado = 'pendiente';
```

```
Seq Scan on pedidos  (cost=0.00..24518.00 rows=12 width=22)
                     (actual time=0.412..118.734 rows=7 loops=1)
  Filter: ((estado = 'pendiente'::text) AND (cliente_id = '3f2a91c4-...'::uuid))
  Rows Removed by Filter: 999993
  Buffers: shared hit=112 read=9406
Planning Time: 0.184 ms
Execution Time: 118.802 ms
```

Se lee de dentro hacia fuera y de abajo arriba. Las tres cifras que importan:

- **`Seq Scan`** — recorre la tabla entera. No es malo en sí: para una tabla de 200 filas es lo más rápido. Es una señal de alarma cuando va acompañado de un `Rows Removed by Filter` enorme, como aquí: ha leído un millón de filas para devolver 7.
- **`rows=12` estimadas frente a `rows=7` reales** — cuando la estimación y la realidad se separan por un factor de 10 o más, el planificador está decidiendo con información mala. Se arregla con `ANALYZE pedidos` o, si el problema es que dos columnas están correlacionadas, con `CREATE STATISTICS`.
- **`Buffers: shared hit=112 read=9406`** — `hit` son páginas servidas desde la caché de Postgres; `read` son las que hubo que pedir al sistema operativo. Un `read` alto es lo que se traduce en milisegundos.

`cost` no son milisegundos: son unidades arbitrarias que solo sirven para comparar planes entre sí. El tiempo real es `actual time` y `Execution Time`.

## Índices más allá del B-tree

Añadimos el índice que falta en la consulta anterior:

```sql
CREATE INDEX idx_pedidos_cliente ON pedidos (cliente_id);
```

Y el mismo `EXPLAIN (ANALYZE, BUFFERS)` ahora dice:

```
Index Scan using idx_pedidos_cliente on pedidos  (cost=0.42..38.61 rows=12 width=22)
                                                 (actual time=0.038..0.061 rows=7 loops=1)
  Index Cond: (cliente_id = '3f2a91c4-...'::uuid)
  Filter: (estado = 'pendiente'::text)
  Buffers: shared hit=11
Planning Time: 0.211 ms
Execution Time: 0.089 ms
```

De 118 ms a 0,089 ms, y de 9 518 páginas leídas a 11. El `Filter` sobre `estado` se mantiene, pero ahora se aplica solo a las filas que el índice ya seleccionó.

**Índice GIN para `jsonb`.** Un B-tree sobre la columna `atributos` indexa el documento completo como valor único: sirve para `atributos = '{...}'` y para nada más. Las consultas con `@>` necesitan GIN:

```sql
CREATE INDEX idx_productos_atributos ON productos USING GIN (atributos jsonb_path_ops);
```

`jsonb_path_ops` es la variante recomendada cuando solo consultas con `@>`: el índice ocupa menos de la mitad y es más rápido. La variante por defecto soporta además `?` (existe la clave), a costa de tamaño.

**Índice GIN para búsqueda de texto.** Postgres trae búsqueda full-text sin motor externo:

```sql
CREATE INDEX idx_productos_busqueda ON productos USING GIN (to_tsvector('spanish', nombre));

SELECT nombre FROM productos
WHERE to_tsvector('spanish', nombre) @@ plainto_tsquery('spanish', 'zapatillas correr');
```

El diccionario `'spanish'` aplica *stemming*: "corriendo", "corrió" y "correr" se reducen a la misma raíz. La expresión del índice y la de la consulta deben coincidir **literalmente**, diccionario incluido, o el índice no se usa.

**Índice parcial.** Si el 98 % de los pedidos están ya entregados y tu cola solo mira los pendientes, indexa solo esos:

```sql
CREATE INDEX idx_pedidos_pendientes ON pedidos (creado_en) WHERE estado = 'pendiente';
```

El índice ocupa una fracción, cabe en memoria y se actualiza menos. Postgres solo lo usa si la condición del `WHERE` de la consulta implica lógicamente la del índice.

**Índice por expresión.** Un índice normal sobre `email` no sirve para `WHERE lower(email) = ...`, porque lo indexado es `email`, no su versión en minúsculas:

```sql
-- ❌ este índice no se usa en la consulta de abajo
CREATE INDEX idx_clientes_email ON clientes (email);

-- ✅ este sí
CREATE INDEX idx_clientes_email_lower ON clientes (lower(email));
SELECT id FROM clientes WHERE lower(email) = lower('Ana@Ejemplo.com');
```

Regla general: **si el `WHERE` envuelve la columna en una función, el índice tiene que envolverla igual**. Lo mismo pasa con `LIKE '%rojo%'`: el comodín inicial impide usar un B-tree; ahí toca GIN con la extensión `pg_trgm`.

## MVCC y el `VACUUM`: por qué borrar no libera espacio

Esto es lo que más sorprende a quien viene de otros motores, y explica media docena de problemas de producción.

Postgres implementa **MVCC** (*Multiversion Concurrency Control*): un `UPDATE` no modifica la fila en su sitio. Escribe una **versión nueva** y marca la vieja como muerta a partir de cierta transacción. Un `DELETE` ni siquiera borra: solo marca. Esto es lo que permite que las lecturas nunca bloqueen a las escrituras ni al revés — cada transacción ve la instantánea que le corresponde.

El precio es que las versiones muertas siguen ocupando disco. Cambiar el estado del pedido #4711 diez veces deja diez versiones, nueve de ellas basura.

```sql
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'pedidos';
```

```
 relname | n_live_tup | n_dead_tup |        last_autovacuum
---------+------------+------------+-------------------------------
 pedidos |    1000000 |     384210 | 2026-07-26 03:11:42.118+02
```

384 000 filas muertas frente a un millón vivas: la tabla ocupa un 38 % más de lo que debería. Eso es **bloat**, y no es solo disco — cada `Seq Scan` lee también esas páginas.

El proceso **autovacuum** limpia esas versiones y devuelve el espacio *a la propia tabla* para reutilizarlo. Por defecto arranca cuando las filas muertas superan el 20 % de la tabla, umbral demasiado alto para una tabla grande y muy escrita: en `pedidos` con un millón de filas eso son 200 000 filas muertas antes de mover un dedo. Se ajusta por tabla:

```sql
ALTER TABLE pedidos SET (autovacuum_vacuum_scale_factor = 0.02);
```

Dos cosas que conviene tener claras:

- **`VACUUM` normal no devuelve el disco al sistema operativo**, solo lo marca como reutilizable dentro del fichero. `VACUUM FULL` sí lo compacta, pero toma un bloqueo `ACCESS EXCLUSIVE`: la tabla queda inaccesible mientras dura. En producción se usa `pg_repack`, que hace lo mismo sin bloquear — y en cualquier caso, nunca antes de comprobar que tienes una [copia de seguridad](../../devops/despliegue-en-vps/Copias-de-Seguridad.md) restaurable.
- **Una transacción abierta paraliza el vacuum de toda la base de datos.** Si tu código hace `BEGIN`, llama a una API externa que tarda cuarenta segundos y luego hace `COMMIT`, el autovacuum no puede limpiar nada más nuevo que esa transacción. Es la causa número uno de bloat inexplicable.

```sql
-- corta las sesiones que se quedan con una transacción abierta sin hacer nada
ALTER DATABASE tienda SET idle_in_transaction_session_timeout = '30s';
```

## Transacciones y niveles de aislamiento

Que un `BEGIN` ... `COMMIT` se confirme entero o se deshaga entero ya lo conoces. Lo que cambia entre motores es el **nivel de aislamiento**: cuánto se protege una transacción de lo que hacen las demás.

| Nivel | Lectura sucia | Lectura no repetible | Lectura fantasma | Coste |
|---|---|---|---|---|
| `READ UNCOMMITTED` | No ocurre | Posible | Posible | — |
| `READ COMMITTED` (por defecto) | No ocurre | Posible | Posible | Bajo |
| `REPEATABLE READ` | No ocurre | Evitada | Evitada | Medio |
| `SERIALIZABLE` | No ocurre | Evitada | Evitada | Alto |

Dos particularidades de Postgres frente al estándar:

1. **`READ UNCOMMITTED` no existe de verdad**: se acepta la sintaxis pero se comporta como `READ COMMITTED`. Las lecturas sucias son imposibles por diseño de MVCC.
2. **`REPEATABLE READ` también evita las lecturas fantasma**, porque trabaja sobre una instantánea completa. Es más fuerte de lo que el estándar exige.

En `READ COMMITTED`, cada sentencia ve una instantánea nueva: dos `SELECT` dentro de la misma transacción pueden dar resultados distintos. Para un informe que suma totales y luego los desglosa, eso significa cifras que no cuadran. Ahí sirve `REPEATABLE READ`:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT sum(total) FROM pedidos WHERE creado_en >= date_trunc('month', now());
SELECT estado, count(*) FROM pedidos WHERE creado_en >= date_trunc('month', now()) GROUP BY estado;
COMMIT;
```

Con los niveles altos aparece un error nuevo que hay que gestionar en el código:

```
ERROR:  could not serialize access due to concurrent update
```

No es un fallo: es Postgres diciendo "esta transacción ya no puede cumplir su garantía". La respuesta correcta es **reintentarla entera**, no capturar el error y seguir.

## `FOR UPDATE SKIP LOCKED`: repartir trabajo entre procesos

Tres workers procesan pedidos pendientes de la misma tabla. Sin precauciones, los tres cogen el #4711 y lo procesan tres veces.

`SELECT ... FOR UPDATE` bloquea las filas seleccionadas, pero hace que los otros workers **esperen** a que se liberen: se serializan y el paralelismo desaparece. `SKIP LOCKED` cambia esperar por saltar:

```sql
BEGIN;
SELECT id FROM pedidos
WHERE estado = 'pendiente'
ORDER BY creado_en
FOR UPDATE SKIP LOCKED
LIMIT 10;
-- ... procesar los 10 ids devueltos ...
UPDATE pedidos SET estado = 'procesado' WHERE id = ANY($1);
COMMIT;
```

Cada worker se lleva diez pedidos **distintos** sin coordinación externa: el primero coge del 4711 al 4720, el segundo salta esos y coge los diez siguientes. El `COMMIT` libera los bloqueos; si el worker muere antes, el `ROLLBACK` implícito devuelve sus pedidos a la cola.

Esto convierte a Postgres en una cola de trabajo perfectamente decente para volúmenes moderados, sin añadir RabbitMQ o SQS a la arquitectura. El límite aparece con miles de mensajes por segundo, donde el bloat de la tabla-cola empieza a pesar.

## Conexiones: por qué Postgres sufre y qué es un pooler

Aquí hay una diferencia arquitectónica de fondo: **Postgres crea un proceso del sistema operativo por cada conexión**, no un hilo. Cada uno consume unos megabytes de memoria base, más `work_mem` por cada operación de ordenación o hash que ejecute.

`max_connections` viene en 100 por una razón. Con 500 conexiones abiertas, aunque el 90 % esté ocioso, el servidor dedica gigabytes a procesos que no hacen nada y el planificador de la CPU pierde el tiempo cambiando de contexto.

```
FATAL:  sorry, too many clients already
```

Un backend .NET con un pool de 100 conexiones por instancia y cuatro instancias ya se pasa del límite (ver [Acceso a datos en .NET](../acceso-a-datos-dotnet/README.md)). La solución no es subir `max_connections`: es poner un **pooler** delante.

**PgBouncer** es un proceso ligero que acepta miles de conexiones de aplicación y las multiplexa sobre unas pocas conexiones reales a Postgres. Tiene tres modos:

| Modo | Cuándo devuelve la conexión al pool | Uso |
|---|---|---|
| `session` | Al desconectar el cliente | Compatible con todo, apenas multiplexa |
| `transaction` | Al terminar cada transacción | El habitual — mucha multiplexación |
| `statement` | Al terminar cada sentencia | Rompe las transacciones, casi nunca |

El modo `transaction` es el que da la ganancia real, pero rompe todo lo que dependa del estado de sesión: `SET` fuera de transacción, tablas temporales, `LISTEN/NOTIFY`, advisory locks de sesión y —muy importante— los *prepared statements* del servidor, que muchos drivers usan por defecto y hay que desactivar explícitamente en la cadena de conexión.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `operator does not exist: character varying = uuid` | Comparas columnas de tipos distintos; Postgres es estricto con los casts, hace falta `::uuid` |
| Creaste un índice y `EXPLAIN` sigue mostrando `Seq Scan` | La tabla es pequeña (correcto), el `WHERE` envuelve la columna en una función, o faltan estadísticas: `ANALYZE tabla` |
| El índice sobre `jsonb` no se usa con `@>` | Es un B-tree; `@>` necesita `USING GIN` |
| `LIKE '%rojo%'` va lentísimo pese al índice | El comodín inicial impide el B-tree; usa GIN con `pg_trgm` |
| Borraste media tabla y el disco no bajó | MVCC: las filas muertas siguen ahí hasta el `VACUUM`, y el normal no devuelve espacio al sistema operativo |
| Una tabla crece sin parar aunque el volumen sea estable | Bloat por autovacuum insuficiente o por una transacción abierta que lo bloquea |
| `sorry, too many clients already` | Sin pooler, o pools de aplicación sobredimensionados |
| Duplicados pese a comprobar antes de insertar | Carrera entre el `SELECT` y el `INSERT`; usa una restricción `UNIQUE` con `ON CONFLICT` |
| `could not serialize access due to concurrent update` | Aislamiento `REPEATABLE READ` o `SERIALIZABLE`; hay que reintentar la transacción |
| Los IDs tienen huecos | Normal: las secuencias no son transaccionales y un `ROLLBACK` no devuelve el número |
| `duplicate key value violates unique constraint "pedidos_pkey"` tras migrar datos | La secuencia se quedó atrás; `SELECT setval(pg_get_serial_sequence('pedidos','id'), max(id)) FROM pedidos` |
| Un `ALTER TABLE` deja la aplicación colgada | Espera un bloqueo `ACCESS EXCLUSIVE` detrás de una consulta larga, y bloquea a todo lo que llega después |

## Buenas prácticas avanzadas

- **Compara las filas estimadas con las reales en cada plan, no solo el tiempo.** En `EXPLAIN ANALYZE`, un `rows=12` estimado frente a `rows=340000` real significa que el planificador eligió con información falsa, y ningún índice lo arreglará. Ejecuta `ANALYZE`; si el desvío persiste, suele deberse a columnas correlacionadas (`ciudad` y `código_postal`, `estado` y `fecha_envio`) y se corrige con `CREATE STATISTICS ... (dependencies)`.
- **En producción, todo DDL con `lock_timeout` y todo índice con `CONCURRENTLY`.** Un `CREATE INDEX` normal bloquea las escrituras de la tabla; `CREATE INDEX CONCURRENTLY` no, a cambio de tardar el doble y poder quedar inválido (revisa `pg_index.indisvalid` después). Y `SET lock_timeout = '3s'` antes de un `ALTER TABLE` evita el peor escenario: la migración espera un bloqueo, y toda la aplicación se encola detrás de ella. Más sobre esto en [Migraciones de esquema](../migraciones-de-esquema/README.md).
- **Ajusta el autovacuum por tabla, no globalmente.** El `scale_factor` del 20 % por defecto es razonable para tablas pequeñas y desastroso para una tabla de millones de filas muy actualizada. Baja el factor en las tablas calientes (`autovacuum_vacuum_scale_factor = 0.02`) y déjalo en paz en el resto. Vigila `n_dead_tup` en `pg_stat_user_tables` como métrica de rutina.
- **Borra los índices que nadie usa; cada uno frena todas las escrituras.** `SELECT indexrelname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0` lista los que no se han usado desde el último reinicio de estadísticas. Un índice sin escaneos solo cuesta: espacio, tiempo de `INSERT`/`UPDATE` y trabajo extra para el vacuum. Comprueba antes que no sea el de una restricción `UNIQUE`.
- **No abras una transacción alrededor de una llamada de red.** Cobrar con una pasarela de pago *dentro* de un `BEGIN`/`COMMIT` mantiene bloqueos y frena el vacuum de toda la base de datos durante segundos. El patrón correcto es: transacción corta para reservar, llamada externa fuera, transacción corta para confirmar. Pon `idle_in_transaction_session_timeout` como red de seguridad, no como solución.

## Recursos didácticos

- [explain.dalibo.com](https://explain.dalibo.com/) — pegas la salida de `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` y te dibuja el plan como un árbol navegable, resaltando en rojo los nodos que consumen el tiempo y donde la estimación falla. Es la forma más rápida de aprender a leer planes.
- [PG Exercises](https://pgexercises.com/) — ejercicios progresivos contra una base de datos PostgreSQL real en el navegador, sin instalar nada, con solución y explicación. Cubre desde `SELECT` básico hasta funciones de ventana y CTEs recursivas.
- [Use The Index, Luke!](https://use-the-index-luke.com/) — libro gratuito en web sobre indexación y planes de ejecución con capítulos específicos de PostgreSQL. Explica *por qué* un índice no se usa, que es justo lo que la documentación oficial da por sabido.

---

*En resumen: PostgreSQL te da tipos ricos (`jsonb`, arrays, rangos), índices que van mucho más allá del B-tree y una concurrencia MVCC que nunca bloquea las lecturas — a cambio de entender el `VACUUM`, leer planes con `EXPLAIN ANALYZE` y poner un pooler delante antes de que las conexiones te desborden.*
