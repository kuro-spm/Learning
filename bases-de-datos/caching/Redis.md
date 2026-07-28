# Redis

## ¿Qué es?

Redis es un almacén de datos **en memoria** de tipo clave-valor: guarda los datos en la RAM de un servicio aparte al que se accede por red. Se usa sobre todo como caché, pero también como almacén de sesiones, contador en tiempo real, cola ligera o *broker* de mensajes.

## ¿Por qué existe?

Una caché "casera" dentro del propio proceso de la aplicación —un diccionario en memoria— tiene dos agujeros que no se pueden tapar desde dentro: se pierde en cuanto el proceso se reinicia, y no se comparte entre varias instancias de la misma aplicación. Con tres instancias detrás de un balanceador tienes tres cachés que se contradicen entre sí.

Redis resuelve ambos problemas: vive como un servicio independiente accesible por red, que todas las instancias consultan y que sobrevive a que cualquiera de ellas se reinicie. A cambio de una ida y vuelta por red, obtienes un estado compartido y coherente.

> Piensa en Redis como una base de datos, pero optimizada al extremo para lecturas y escrituras rapidísimas de estructuras simples, a costa de renunciar a las consultas complejas (*joins*, transacciones elaboradas) que sí ofrece una relacional.

## ¿Cuándo y para qué se usa?

- Caché distribuida compartida entre varias instancias de una API (ver [Caché distribuida vs. local](Cache-Distribuida-vs-Local.md)).
- Almacén de sesión de usuario en aplicaciones web con varias instancias detrás de un balanceador.
- Contadores en tiempo real, como el número de visitas a un producto o los "me gusta" de una publicación.
- Colas ligeras y sistemas *pub/sub*, por ejemplo para notificar a varios servicios que un pedido ha cambiado de estado.
- Rankings y *leaderboards* (con el tipo `Sorted Set`), como el top de productos más vendidos del día.

Esta guía cubre **Redis como herramienta**: sus estructuras, su modelo de ejecución y su cliente de .NET. Los patrones de caché en abstracto —*cache-aside*, *write-through*, *hit ratio*— están en [Caching](Caching.md), y las estrategias de invalidación y el *cache stampede*, en [Estrategias de invalidación](Estrategias-de-Invalidacion.md). El ejemplo conductor es una tienda online cuyo origen de verdad es la base de datos `TiendaDB`, con las tablas `Productos`, `Pedidos` y `Clientes`: el producto de referencia es el `123` y el pedido, el **#4711**.

## Puesta en marcha en un minuto

No hace falta instalar nada en el sistema, y `redis-cli` —la consola oficial— viene dentro de la propia imagen:

```bash
docker run -d --name redis-tienda -p 6379:6379 redis:7-alpine
docker exec -it redis-tienda redis-cli
```

`PING` comprueba que el servidor responde, y a partir de ahí ya se puede escribir y leer:

```
127.0.0.1:6379> PING
PONG
127.0.0.1:6379> SET producto:123:nombre "Camiseta azul"
OK
127.0.0.1:6379> GET producto:123:nombre
"Camiseta azul"
127.0.0.1:6379> GET producto:999:nombre
(nil)
```

Tres salidas que verás constantemente: `OK` para una escritura aceptada, el valor entre comillas para una lectura con éxito, y `(nil)` para una clave que no existe. **`(nil)` no es un error**: leer una clave inexistente es una operación válida, y en el código se traduce en un nulo, no en una excepción. Y los dos puntos de `producto:123:nombre` no significan nada para Redis —el espacio de claves es plano y `:` solo simula jerarquías—, pero es una convención universal y respetarla es lo que hace legible un `SCAN producto:*` seis meses después.

## El modelo de datos, estructura por estructura

La diferencia entre Redis y un simple diccionario remoto es que el **valor** puede ser algo más que un texto. Cada tipo trae sus propios comandos, y elegir bien es la decisión de diseño más rentable que tomarás.

### `String`: valores simples y contadores

Un `String` guarda texto o bytes, hasta 512 MB. Su interés real está en los comandos numéricos: si el valor parece un número, Redis sabe sumarle.

```
127.0.0.1:6379> INCR producto:123:visitas
(integer) 1
127.0.0.1:6379> INCRBY producto:123:visitas 5
(integer) 6
127.0.0.1:6379> SET carrito:cliente:42:total 59.90
OK
127.0.0.1:6379> INCRBYFLOAT carrito:cliente:42:total 10.00
"69.9"
```

`INCR` devuelve el valor **después** de sumar, y lo hace de forma atómica: no hay un `GET`, un `+1` y un `SET` que otro proceso pueda entrelazar. Si la clave no existe la crea a cero y suma, así que el contador de visitas no necesita inicializarse.

### `Hash`: objetos con campos

Un `Hash` es un diccionario dentro de la clave: lo natural para un objeto cuyos campos se leen o actualizan por separado.

```
127.0.0.1:6379> HSET producto:123 nombre "Camiseta azul" precio 19.99 stock 40
(integer) 3
127.0.0.1:6379> HGET producto:123 precio
"19.99"
127.0.0.1:6379> HINCRBY producto:123 stock -1
(integer) 39
127.0.0.1:6379> HGETALL producto:123
1) "nombre"
2) "Camiseta azul"
3) "precio"
4) "19.99"
5) "stock"
6) "39"
```

`HSET` devuelve cuántos campos **nuevos** ha creado (aquí 3; repetido devuelve `(integer) 0` porque solo actualiza). La ventaja frente a guardar el producto como JSON en un `String` es directa: descontar una unidad de stock es un `HINCRBY`, no un leer-deserializar-modificar-serializar-escribir que además pisa los cambios de quien vaya en paralelo.

### `List`: colas y pilas

Una `List` es una lista doblemente enlazada de strings: se empuja y se saca por cualquiera de los dos extremos.

```
127.0.0.1:6379> LPUSH pedidos:pendientes "pedido:4711"
(integer) 1
127.0.0.1:6379> LPUSH pedidos:pendientes "pedido:4712"
(integer) 2
127.0.0.1:6379> RPOP pedidos:pendientes
"pedido:4711"
```

`LPUSH` + `RPOP` es una cola FIFO: entra por la izquierda, sale por la derecha, y el `#4711` sale primero porque entró primero (`LPUSH` + `LPOP` sería una pila). `BRPOP pedidos:pendientes 5` es la variante bloqueante: espera hasta 5 segundos a que haya algo en lugar de devolver `(nil)` al instante, y así el consumidor no hace *polling* en bucle.

### `Set`: pertenencia y operaciones de conjuntos

Un `Set` guarda elementos únicos sin orden y responde "¿está esto aquí?" en tiempo constante.

```
127.0.0.1:6379> SADD categorias:producto:123 "ropa" "verano" "rebajas"
(integer) 3
127.0.0.1:6379> SISMEMBER categorias:producto:123 "verano"
(integer) 1
127.0.0.1:6379> SISMEMBER categorias:producto:123 "invierno"
(integer) 0
127.0.0.1:6379> SADD categorias:producto:456 "ropa" "invierno"
(integer) 2
127.0.0.1:6379> SINTER categorias:producto:123 categorias:producto:456
1) "ropa"
```

`SISMEMBER` devuelve `1` o `0`, no `true`/`false`. Y `SINTER`, `SUNION` y `SDIFF` calculan intersecciones, uniones y diferencias **en el servidor**: cruzar dos conjuntos de etiquetas no requiere traerse ambos a la aplicación.

### `Sorted Set`: rankings

Un `Sorted Set` es un `Set` donde cada elemento lleva una puntuación (`score`) numérica, y Redis lo mantiene ordenado por ella permanentemente.

```
127.0.0.1:6379> ZADD productos-destacados 340 "producto:123" 512 "producto:456" 87 "producto:789"
(integer) 3
127.0.0.1:6379> ZREVRANGE productos-destacados 0 2 WITHSCORES
1) "producto:456"
2) "512"
3) "producto:123"
4) "340"
5) "producto:789"
6) "87"
127.0.0.1:6379> ZINCRBY productos-destacados 1 "producto:123"
"341"
127.0.0.1:6379> ZREVRANK productos-destacados "producto:123"
(integer) 1
```

`ZREVRANGE 0 2` devuelve los tres primeros de mayor a menor (`ZRANGE` sería ascendente), y `ZREVRANK` dice en qué posición está uno concreto — el `producto:123` es el segundo, porque los rangos empiezan en 0. El top de ventas del día es un `ZINCRBY` por cada venta y un `ZREVRANGE` para leerlo; nada de ordenar en la aplicación.

### La tabla de decisión

| Estructura | Qué modela | Comandos clave | Cuándo elegirla |
|---|---|---|---|
| `String` | Un valor suelto o un contador | `SET`, `GET`, `INCR`, `INCRBYFLOAT` | El valor se lee y escribe entero, o es un número |
| `Hash` | Un objeto con campos | `HSET`, `HGET`, `HGETALL`, `HINCRBY` | Necesitas leer o tocar **un** campo sin traer el resto |
| `List` | Una secuencia ordenada por inserción | `LPUSH`, `RPOP`, `BRPOP`, `LLEN` | Cola o pila de trabajo; historial reciente |
| `Set` | Una colección sin duplicados ni orden | `SADD`, `SISMEMBER`, `SINTER` | La pregunta es "¿pertenece?" o hay que cruzar conjuntos |
| `Sorted Set` | Elementos con puntuación, siempre ordenados | `ZADD`, `ZINCRBY`, `ZREVRANGE` | Ranking, top-N, o una cola por prioridad o por tiempo |

Hay más tipos, y conviene saber que existen aunque no los uses el primer día: **bitmaps** (millones de flags booleanos en muy poca memoria, por ejemplo qué días entró cada cliente), **HyperLogLog** (cuenta elementos distintos con un 0,8 % de error usando 12 KB fijos, sea un millar o mil millones), **streams** (log de eventos con grupos de consumidores y confirmación, una cola de verdad), y **tipos geoespaciales** (búsquedas por radio sobre coordenadas).

## Expiración y TTL

Que una clave caduque sola es lo que convierte a Redis en una caché y no en un almacén que crece hasta reventar. El TTL (*time to live*) se pone al escribir o después.

```
127.0.0.1:6379> SET sesion:abc123 "{\"clienteId\":42}" EX 1800
OK
127.0.0.1:6379> TTL sesion:abc123
(integer) 1795
127.0.0.1:6379> SET producto:123:nombre "Camiseta azul"
OK
127.0.0.1:6379> TTL producto:123:nombre
(integer) -1
127.0.0.1:6379> TTL sesion:noexiste
(integer) -2
```

Los dos valores negativos se confunden constantemente y significan cosas muy distintas: **`-1`** es "la clave existe y no caduca nunca"; **`-2`** es "la clave no existe" (caducó, o nunca estuvo). Y dos comandos más: `EXPIRE clave 3600` añade caducidad a una clave que ya existía y devuelve `(integer) 1` si lo consigue; `PERSIST clave` se la quita y la vuelve inmortal.

### La trampa clásica: el `SET` que borra el TTL

Esta es la causa número uno de claves eternas en una caché que "tiene TTL en todo". Un `SET` normal **reemplaza la clave entera, metadatos incluidos**, y eso borra la caducidad:

```
127.0.0.1:6379> SET producto:123:nombre "Camiseta azul" EX 600
OK
127.0.0.1:6379> TTL producto:123:nombre
(integer) 598
127.0.0.1:6379> SET producto:123:nombre "Camiseta azul marino"
OK
127.0.0.1:6379> TTL producto:123:nombre
(integer) -1
```

Ese `-1` es el problema: la clave acaba de volverse inmortal sin que nadie lo pidiera. El código que refresca el valor tras un cambio en `TiendaDB` funciona, la caché devuelve datos correctos, y la memoria sube mes a mes sin explicación. La solución es explícita:

- ✅ `SET clave valor EX 600` — refresca valor **y** caducidad.
- ✅ `SET clave valor KEEPTTL` — cambia el valor y conserva la caducidad que hubiera.
- ❌ `SET clave valor` sobre una clave con TTL — la vuelve eterna.

`HSET`, `LPUSH`, `SADD` y compañía **no** borran el TTL, porque modifican el contenido de una clave existente. El problema es exclusivo de los comandos que sustituyen la clave completa.

## Un solo hilo: por qué es una ventaja y una trampa

Redis ejecuta los comandos en **un único hilo**, uno detrás de otro. Suena a limitación y en realidad es la mitad de su diseño: significa que **cada comando es atómico por construcción**. Cuando cien peticiones simultáneas hacen `INCRBYFLOAT carrito:cliente:42:total 10.00`, no hay carrera posible, porque el servidor los pone en fila y los ejecuta enteros, uno a uno. No necesitas *locks*, ni transacciones, ni comparar-y-cambiar. Lo mismo con `HINCRBY producto:123 stock -1`: descontar stock desde diez instancias de la API es seguro sin coordinación ninguna.

El precio es la otra cara exacta de la misma moneda: **un comando lento bloquea a todos los demás**, porque no hay otro hilo que atienda mientras. Un comando que tarde 300 ms son 300 ms en los que ninguna petición avanza, y la aplicación lo ve como *timeouts* repartidos por todas partes.

## `KEYS` frente a `SCAN`: el comando que tumba un servidor

El caso más común de comando lento es `KEYS producto:*`. Recorre **todo** el espacio de claves comparando cada una con el patrón y no devuelve nada hasta acabar; con un millón de claves, eso son cientos de milisegundos con el hilo ocupado. La cuenta lo hace tangible: si el servidor atiende 20 000 peticiones por segundo, un `KEYS` de 300 ms deja 6 000 peticiones encoladas, y muchas habrán superado su *timeout* antes de que les toque el turno.

La alternativa es `SCAN`, que hace el mismo recorrido **en trozos**:

```
127.0.0.1:6379> SCAN 0 MATCH producto:* COUNT 100
1) "1024"
2) 1) "producto:123"
   2) "producto:456"
   3) "producto:789"
127.0.0.1:6379> SCAN 1024 MATCH producto:* COUNT 100
1) "0"
2) 1) "producto:812"
```

La primera línea de la respuesta es el **cursor** con el que pedir el siguiente lote; la segunda, las claves de este. Empiezas en `0` y terminas cuando el cursor vuelve a `"0"`. `COUNT` es un tamaño de lote sugerido, no un límite, y un lote puede volver vacío sin que eso signifique el final. `SCAN` garantiza que verás todas las claves presentes durante todo el recorrido, pero puede repetir alguna: trata el resultado como un conjunto, no como una lista. `HSCAN`, `SSCAN` y `ZSCAN` hacen lo mismo dentro de una estructura grande.

Otros comandos que bloquean igual, y que conviene reconocer antes de escribirlos:

- **`FLUSHALL` / `FLUSHDB`** — borran la base de datos entera de forma síncrona. Existe `FLUSHALL ASYNC`, pero el riesgo real es el hábito de ejecutarlos "para limpiar" en un servidor que resulta ser el de producción.
- **`DEL` de una clave enorme** — borrar un `Hash` con un millón de campos libera un millón de objetos en el hilo principal. `UNLINK clave` delega esa liberación a un hilo de fondo y devuelve al instante: para colecciones grandes, `UNLINK` siempre.
- **`SMEMBERS` o `HGETALL` sobre colecciones enormes** — el coste es proporcional al tamaño, y además hay que serializarlo y enviarlo todo. Usa `SSCAN`/`HSCAN`.
- **Scripts Lua largos** — también corren en el hilo único, así que un bucle de cien mil iteraciones bloquea igual que un `KEYS`.

## Atomicidad más allá de un comando

Un comando suelto es atómico, pero "lee el stock, comprueba que llega, réstalo" son tres. Redis ofrece tres mecanismos, y solo uno resuelve el caso de verdad.

**`MULTI`/`EXEC`** encola comandos y los ejecuta juntos, sin que nada se cuele entre ellos:

```
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> HINCRBY producto:123 stock -1
QUEUED
127.0.0.1:6379> LPUSH pedidos:pendientes "pedido:4711"
QUEUED
127.0.0.1:6379> EXEC
1) (integer) 38
2) (integer) 1
```

Los comandos responden `QUEUED` en lugar de ejecutarse, y `EXEC` devuelve el array de resultados en orden. Lo que **no** hay es *rollback*: si el segundo comando falla en ejecución (un `LPUSH` sobre una clave que resulta ser un `Hash`, por ejemplo), el primero **ya se aplicó y se queda aplicado** — `EXEC` devuelve el error en su posición y sigue con el resto. No es una transacción de base de datos relacional; es un lote sin interrupciones. Y dentro de un `MULTI` **no puedes decidir nada**, porque no ves los resultados hasta el `EXEC`: un "resta stock solo si hay suficiente" no cabe ahí.

**`WATCH`** añade optimismo. Vigila una clave y hace que `EXEC` falle si alguien la ha modificado mientras tanto:

```
127.0.0.1:6379> WATCH producto:123
OK
127.0.0.1:6379> HGET producto:123 stock
"38"
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> HSET producto:123 stock 37
QUEUED
127.0.0.1:6379> EXEC
(nil)
```

Ese `(nil)` significa que la transacción **no se ejecutó**, porque `producto:123` cambió entre el `WATCH` y el `EXEC`. Es correcto, pero obliga a reintentar en bucle desde la aplicación, y con mucha concurrencia sobre la misma clave se reintenta sin parar.

**Los scripts Lua con `EVAL`** son la forma real de hacer lógica atómica. El script entero se ejecuta como un solo comando en el hilo único, así que puede leer, decidir y escribir sin que nadie se entrometa:

```lua
local stock = tonumber(redis.call('HGET', KEYS[1], 'stock'))
if stock == nil or stock < tonumber(ARGV[1]) then
  return -1
end
return redis.call('HINCRBY', KEYS[1], 'stock', -tonumber(ARGV[1]))
```

Se invoca declarando cuántos de los argumentos son claves:

```
127.0.0.1:6379> EVAL "<script>" 1 producto:123 5
(integer) 32
127.0.0.1:6379> EVAL "<script>" 1 producto:123 500
(integer) -1
```

El `1` dice que el primer argumento (`producto:123`) es una clave y va en `KEYS[1]`; el resto va en `ARGV`. La primera llamada descuenta 5 y devuelve el stock resultante; la segunda devuelve `-1` sin tocar nada, porque no hay 500 unidades. Sin reintentos y sin ventana de carrera. Mantén los scripts cortos —recuerda que bloquean— y declara siempre las claves en `KEYS` en lugar de construirlas dentro del script.

## Cuando la memoria se llena

Redis vive en RAM, y la RAM se acaba. Dos parámetros deciden qué pasa entonces:

```bash
redis-cli CONFIG SET maxmemory 2gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

`maxmemory` es el techo; `maxmemory-policy`, la política de **expulsión** (*eviction*) que se aplica al llegar a él.

| Política | Qué hace al llenarse | Cuándo toca |
|---|---|---|
| `noeviction` | No borra nada: **falla la escritura** | Cuando Redis es el origen de verdad y perder datos es inaceptable |
| `allkeys-lru` | Expulsa las claves menos usadas recientemente, con o sin TTL | Caché pura — la opción por defecto sensata |
| `allkeys-lfu` | Expulsa las **menos frecuentes**, no las más antiguas | Caché con un núcleo estable de claves populares |
| `volatile-lru` | LRU, pero solo entre las claves **con TTL** | Mezclas caché y datos permanentes en la misma instancia |
| `volatile-ttl` | Expulsa primero las que caducan antes | Los TTL reflejan de verdad la importancia del dato |
| `allkeys-random` | Expulsa al azar | Casi nunca; solo si todas las claves valen lo mismo |

El aviso importante: **`noeviction` es el valor por defecto**, y en una instancia usada como caché es justo lo contrario de lo que quieres. Al llegar al techo, cualquier escritura devuelve `(error) OOM command not allowed when used memory > 'maxmemory'.` La caché deja de admitir entradas nuevas, la aplicación lanza excepciones al intentar cachear, y todo el tráfico cae sobre `TiendaDB` a la vez. Para un uso puro de caché la política casi siempre debe ser `allkeys-lru`: preferir perder lo menos usado a dejar de funcionar. Cuidado también con las `volatile-*`: si ninguna clave tiene TTL, no hay nada expulsable y se comportan exactamente como `noeviction`, con el mismo error.

Para diagnosticar, dos comandos:

```
127.0.0.1:6379> INFO memory
used_memory_human:1.84G
maxmemory_human:2.00G
maxmemory_policy:allkeys-lru
evicted_keys:18402
127.0.0.1:6379> MEMORY USAGE producto:123
(integer) 152
```

`evicted_keys` subiendo es la señal de que estás expulsando datos de forma continua: o falta memoria, o hay claves sin TTL acumulándose. `MEMORY USAGE` da los bytes reales de una clave concreta, metadatos incluidos — útil para descubrir que un valor "pequeño" pesa 40 KB porque alguien serializó el objeto entero con sus relaciones.

## Persistencia: RDB, AOF o nada

Redis puede escribir en disco para sobrevivir a un reinicio. Hay tres respuestas razonables:

| Opción | Cómo funciona | Qué se pierde | Coste |
|---|---|---|---|
| **RDB** | Volcado binario completo cada X segundos o cada N cambios | Todo lo escrito desde el último volcado (minutos) | Un `fork` del proceso y un pico de CPU y memoria por volcado |
| **AOF** | Registra cada comando de escritura en un log que se reproduce al arrancar | Como máximo un segundo con `appendfsync everysec` | Escritura continua en disco y reescrituras periódicas del log |
| **Ninguna** | Solo RAM | Absolutamente todo al reiniciar | Cero |

Se pueden combinar (AOF para la durabilidad fina, RDB para copias puntuales fáciles de mover), pero para una **caché pura** la respuesta razonable suele ser la tercera: *no me importa perderla*. El origen de verdad es `TiendaDB`, y una caché vacía tras un reinicio se rellena sola con las primeras peticiones; activar persistencia añade E/S de disco, picos de latencia en el `fork` del RDB y un arranque más lento para proteger datos que por definición son reconstruibles. Lo que sí hay que prever es el arranque en frío, con todo el tráfico cayendo en la base de datos hasta que la caché se calienta — eso se mitiga con las técnicas de [Estrategias de invalidación](Estrategias-de-Invalidacion.md). La persistencia se justifica cuando Redis guarda algo que **no** está en otro sitio: sesiones que no quieres invalidar en cada despliegue, o una cola cuyos mensajes no se pueden perder. Si es tu caso, plantéate primero si ese dato debería estar en Redis.

## Desde C# con `StackExchange.Redis`

`StackExchange.Redis` es el cliente estándar de .NET. Tiene una regla que domina todo lo demás.

### `ConnectionMultiplexer` se crea una sola vez

`ConnectionMultiplexer` es **caro de crear y completamente thread-safe**. Está diseñado para multiplexar todas las operaciones de la aplicación sobre unas pocas conexiones TCP. Se crea una vez, para todo el proceso, y se comparte:

```csharp
// ✅ registrado como singleton, una instancia para toda la aplicación
builder.Services.AddSingleton<IConnectionMultiplexer>(_ =>
    ConnectionMultiplexer.Connect("localhost:6379,abortConnect=false"));
```

Con `IDistributedCache`, `builder.Services.AddStackExchangeRedisCache(...)` ya lo gestiona así por dentro. Y este es el error de rendimiento más frecuente con esta librería:

```csharp
// ❌ un multiplexer por petición: abre y cierra conexiones sin parar
using var redis = ConnectionMultiplexer.Connect("localhost:6379");
var db = redis.GetDatabase();
```

Funciona en desarrollo y se derrumba en producción. Lo que se ve cuando pasa: agotamiento de sockets (miles de conexiones en `TIME_WAIT`, visibles con `netstat`), latencias crecientes porque cada petición paga el *handshake* completo, y finalmente `RedisConnectionException: It was not possible to connect to the redis server(s)`, que apunta al servidor cuando el problema está en el cliente. Un solo `ConnectionMultiplexer` compartido por miles de peticiones concurrentes es el uso previsto, no un atajo.

### `IDatabase` y las operaciones básicas

`IDatabase` sí es barato: se obtiene del multiplexer cuando hace falta y no se guarda.

```csharp
IDatabase db = redis.GetDatabase();

await db.StringSetAsync("producto:123:nombre", "Camiseta azul", TimeSpan.FromMinutes(30));
string? nombre = await db.StringGetAsync("producto:123:nombre");

await db.HashSetAsync("producto:123", new[]
{
    new HashEntry("nombre", "Camiseta azul"),
    new HashEntry("precio", "19.99"),
    new HashEntry("stock", "40")
});
await db.KeyExpireAsync("producto:123", TimeSpan.FromMinutes(30));
long stock = await db.HashDecrementAsync("producto:123", "stock");
```

Los nombres siguen los comandos: `StringSetAsync` es `SET`, `HashSetAsync` es `HSET`, `KeyExpireAsync` es `EXPIRE`. Un detalle propio de C#: `StringGetAsync` devuelve un `RedisValue`, que al convertirse a `string?` da `null` si la clave no existía. Comprueba ese nulo — es el `(nil)` de la consola, y significa "no estaba en caché", no "hubo un fallo".

### Serialización: Redis guarda bytes

Redis no sabe qué es un `Producto`: guarda bytes, así que la conversión es tuya, y JSON es la opción por defecto sensata.

```csharp
using System.Text.Json;

public record Producto(int Id, string Nombre, decimal Precio, int Stock);

var producto = new Producto(123, "Camiseta azul", 19.99m, 40);
await db.StringSetAsync(
    $"producto:{producto.Id}",
    JsonSerializer.Serialize(producto),
    TimeSpan.FromMinutes(30));

string? json = await db.StringGetAsync("producto:123");
Producto? recuperado = json is null ? null : JsonSerializer.Deserialize<Producto>(json);
```

Ese `json is null` es el *cache miss*: toca ir a `TiendaDB` y guardar el resultado de vuelta. Dos avisos: el JSON de un objeto con colecciones anidadas crece muy rápido —revisa el tamaño real con `MEMORY USAGE`—, y cambiar la forma del `record` deja en la caché documentos con la forma antigua que fallarán al deserializar. Un prefijo con versión (`producto:v2:123`) resuelve eso en el propio despliegue.

### `IDistributedCache` frente al cliente nativo

`IDistributedCache` es la abstracción de .NET: portable entre Redis, SQL Server o memoria, con una interfaz mínima de *get*/*set* de bytes con caducidad. Úsala cuando lo único que quieras sea cachear valores y no atarte al proveedor; usa `IDatabase` directamente en cuanto necesites las estructuras propias de Redis (`Hash`, `Sorted Set`), scripts Lua o *pub/sub*, que la abstracción no expone.

### Batching y pipelining

Cada comando es una ida y vuelta por red. Con 1 ms de latencia, 100 comandos secuenciales son **100 ms de espera pura**. Agrupados, el cliente los envía juntos y recibe todas las respuestas juntas: unos pocos milisegundos. En `StackExchange.Redis` basta con no esperar cada tarea por separado.

```csharp
// ❌ 100 idas y vueltas: ~100 ms
foreach (int id in ids)
    await db.StringGetAsync($"producto:{id}");

// ✅ una ráfaga, unos pocos ms
var tareas = ids.Select(id => db.StringGetAsync($"producto:{id}")).ToArray();
RedisValue[] valores = await Task.WhenAll(tareas);
```

Ojo con el orden: lanzar todas las tareas **antes** del primer `await` es lo que permite al multiplexer agruparlas. Un `await` dentro del bucle las serializa otra vez. `db.CreateBatch()` hace lo mismo de forma explícita —se crean las tareas, se llama a `batch.Execute()` y se esperan— y `CreateTransaction()` es el equivalente de `MULTI`/`EXEC`. Para lecturas de muchas claves también existe `StringGetAsync(RedisKey[])`, que es un único `MGET`.

## Claves calientes

Una clave concreta con miles de peticiones por segundo puede saturar el hilo único aunque el servidor tenga capacidad de sobra en memoria y CPU: todo el tráfico converge en el mismo punto y se serializa. El síntoma es latencia alta con métricas de recursos tranquilas, y la salida es **partir la clave**: un contador global de visitas al `producto:123` se reparte en `producto:123:visitas:0` … `producto:123:visitas:9`, cada escritura elige un fragmento al azar, y la lectura suma los diez. Para un valor muy leído, la otra vía es una caché local de vida cortísima delante de Redis que absorba la ráfaga — la caché en dos niveles de [Caché distribuida vs. local](Cache-Distribuida-vs-Local.md).

## Cuándo NO usar Redis

- **Cuando consultas por contenido.** Redis busca por clave. "Dame los productos de menos de 20 € con stock" no tiene respuesta sin recorrer el espacio de claves entero. Eso es trabajo de una relacional; Redis no tiene lenguaje de consultas, ni `JOIN`, ni integridad referencial.
- **Cuando hay relaciones que mantener.** No existen claves foráneas ni `ON DELETE CASCADE`: la coherencia entre `producto:123` y `categorias:producto:123` la sostiene tu código, y solo tu código.
- **Cuando el dato no puede perderse** y no estás dispuesto a configurar, probar y vigilar persistencia de verdad. Redis puede ser durable, pero por defecto no lo es y hacerlo bien tiene coste operativo.
- **Cuando el valor por clave es enorme.** El límite es 512 MB, pero el problema aparece muchísimo antes: valores grandes saturan la red, ocupan el hilo al serializarse y hacen que borrarlos bloquee. Si guardas ficheros o documentos de megabytes, eso va a un almacenamiento de objetos.
- **Cuando una caché local resolvería lo mismo.** Con una sola instancia de la aplicación y datos que se pueden repetir por proceso, `IMemoryCache` da latencia de nanosegundos y no añade un servicio que desplegar, monitorizar y mantener. Redis es infraestructura; se paga en operación.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `RedisTimeoutException: Timeout performing GET (5000ms)` | Un comando lento ocupando el hilo (`KEYS`, `HGETALL` gigante, script Lua), o un `ConnectionMultiplexer` por petición saturando conexiones |
| `No connection is available to service this operation` | El multiplexer perdió la conexión y sigue reintentando; o se creó y se desechó por petición y no queda ninguno vivo |
| `RedisConnectionException: It was not possible to connect` | Servidor caído, puerto no publicado, o `abortConnect=true` (por defecto) haciendo fallar el arranque si Redis aún no está listo |
| `OOM command not allowed when used memory > 'maxmemory'` | Techo de memoria alcanzado con `maxmemory-policy noeviction` (el valor por defecto), o una política `volatile-*` sin claves con TTL que expulsar |
| `WRONGTYPE Operation against a key holding the wrong kind of value` | Un `GET` sobre una clave que es un `Hash`, o un `LPUSH` sobre un `String`. Redis no convierte tipos: la misma clave no puede ser dos cosas |
| La memoria crece sin parar aunque "todo tiene TTL" | Un `SET` sin `KEEPTTL` sobre claves que ya tenían caducidad. Compruébalo con `TTL clave`: un `(integer) -1` donde esperabas segundos lo confirma |
| El servidor entero se congela unos cientos de ms cada cierto tiempo | Un `KEYS` en producción, un `DEL` de una colección enorme (usa `UNLINK`), o el `fork` del volcado RDB |
| Latencia alta con CPU y memoria del servidor tranquilas | Clave caliente: todo el tráfico sobre la misma clave, serializado en el hilo único |
| `TTL` devuelve `(integer) -2` para una clave que acabas de escribir | La clave no existe: caducó, la expulsó la política de *eviction*, o escribiste en otra base de datos lógica o en otra instancia |
| Al deserializar salta `JsonException` tras un despliegue | En la caché quedan documentos con la forma antigua del objeto; versiona el prefijo de la clave |

## Buenas prácticas avanzadas

- **Elige la estructura, no el `String` por inercia.** Guardar el producto serializado en JSON dentro de un `String` obliga a leer, deserializar, modificar, serializar y reescribir para tocar el stock — y ese ciclo pisa los cambios concurrentes. Un `Hash` convierte lo mismo en `HINCRBY producto:123 stock -1`: un comando, atómico, sin mover el resto del objeto por la red.
- **Pon TTL a (casi) todo, y verifica que sigue ahí.** Una clave sin caducidad es una fuga de memoria en potencia, y el `SET` sin `KEEPTTL` la crea sin que nadie se dé cuenta. Un `TTL` de muestreo sobre las claves que tu código refresca es una comprobación de treinta segundos que evita meses de crecimiento inexplicado. Y fija siempre `maxmemory-policy allkeys-lru` en una instancia de caché: el `noeviction` por defecto convierte un techo de memoria en una caída.
- **Trata `KEYS` como un comando prohibido, y `UNLINK` como el `DEL` normal.** En un servidor de un solo hilo, cada comando O(n) es una parada del sistema completo. `KEYS` se sustituye por `SCAN` con cursor; `HGETALL` y `SMEMBERS` sobre colecciones grandes, por `HSCAN` y `SSCAN`; y `DEL` de una estructura con muchos elementos, por `UNLINK`, que libera la memoria en un hilo de fondo y devuelve al instante.
- **Un `ConnectionMultiplexer` por proceso, y agrupa las ráfagas.** Es la mitad de todos los problemas de rendimiento con `StackExchange.Redis`: crearlo por petición agota sockets y produce *timeouts* que parecen del servidor. Y cuando necesites muchas claves, crea todas las tareas antes del primer `await` (o usa `MGET`): 100 comandos con 1 ms de ida y vuelta son 100 ms en serie y unos pocos agrupados.
- **Para lógica de varios pasos, Lua antes que `MULTI`/`WATCH`.** `MULTI` no permite decidir con resultados intermedios y no hace *rollback* si un comando falla; `WATCH` obliga a reintentar desde el cliente y degrada justo donde hay contención. Un script `EVAL` corto se ejecuta como un solo comando atómico y expresa "comprueba y actúa" sin ventanas de carrera — declara siempre las claves en `KEYS` y mantenlo breve, porque bloquea igual que cualquier otro comando.
- **Vigila `evicted_keys` y las claves calientes, no solo `used_memory`.** `INFO memory` con `evicted_keys` creciendo de forma sostenida significa que Redis está tirando datos continuamente, y eso se traduce en *cache misses* y carga extra en la base de datos aunque la memoria "quepa". Igual con la latencia: si sube con los recursos ociosos, busca la clave que concentra el tráfico y particiónala.

## Recursos didácticos

- [try.redis.io](https://try.redis.io/) — una consola interactiva de Redis en el navegador, sin instalar nada. Es la forma más rápida de probar los comandos de esta guía y ver la salida real de un `ZREVRANGE` o un `TTL`.
- [Documentación de comandos de Redis](https://redis.io/docs/latest/commands/) — la referencia completa, con la **complejidad algorítmica** de cada comando en la ficha. Ese dato es el que importa en un servidor de un solo hilo: buscar `O(N)` antes de escribir un comando nuevo evita casi todos los bloqueos.
- [Redis University](https://university.redis.io/) — cursos gratuitos con vídeo y ejercicios; el de introducción cubre estructuras y persistencia con más calma de la que cabe aquí.

---

*En resumen: Redis es rapidísimo porque hace pocas cosas en un solo hilo — elige bien la estructura, pon TTL a todo y no le mandes nunca un comando O(N), porque el que bloquea el hilo bloquea a toda la aplicación.*
