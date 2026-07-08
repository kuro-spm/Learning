# Redis

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Redis es un almacén de datos en memoria, de tipo clave-valor, usado sobre todo como caché, pero también como broker de mensajes o almacén de sesiones. Es extremadamente rápido porque, a diferencia de una base de datos relacional, mantiene los datos en memoria RAM en vez de en disco (aunque puede persistirlos de forma opcional).

## ¿Por qué existe?

Una caché "casera" dentro del propio proceso de la aplicación (por ejemplo, un diccionario en memoria) se pierde en cuanto el proceso se reinicia y no se puede compartir entre varias instancias de la misma aplicación. Redis resuelve ambos problemas: vive como un servicio aparte, accesible por red, que todas las instancias de tu aplicación pueden consultar y que sobrevive a que cualquiera de ellas se reinicie.

> Piensa en Redis como una base de datos, pero optimizada al extremo para lecturas y escrituras rapidísimas de estructuras simples, a costa de renunciar a las consultas complejas (*joins*, transacciones elaboradas) que sí ofrece SQL Server.

## ¿Cuándo y para qué se usa?

- Caché distribuida compartida entre varias instancias de una API (ver [Caché distribuida vs. local](Cache-Distribuida-vs-Local.md)).
- Almacén de sesión de usuario en aplicaciones web con varias instancias detrás de un balanceador.
- Contadores en tiempo real, como el número de visitas a un producto o "me gusta" de una publicación.
- Colas ligeras y sistemas *pub/sub*, por ejemplo para notificar a varios servicios que un pedido ha cambiado de estado.
- Rankings y *leaderboards* (con el tipo *sorted set*), como el top de productos más vendidos del día.

## Lo mínimo que necesitas saber

**1. Es clave-valor, con estructuras de datos más ricas que un simple string**

```bash
SET producto:123:nombre "Camiseta azul"
GET producto:123:nombre
```

**2. Strings, para valores simples**

```bash
SET carrito:usuario:42:total 59.90
INCRBYFLOAT carrito:usuario:42:total 10.00
```

**3. Hashes, para objetos con varios campos**

```bash
HSET producto:123 nombre "Camiseta azul" precio 19.99 stock 40
HGET producto:123 precio
```

**4. Listas y sets, para colecciones**

```bash
LPUSH pedidos:pendientes "pedido:987"
SADD categorias:producto:123 "ropa" "verano"
```

**5. Expiración con TTL, la base de su uso como caché**

```bash
SET sesion:abc123 "{...datos de sesion...}" EX 1800
```

**6. Desde C#, con `StackExchange.Redis`**

```csharp
var redis = ConnectionMultiplexer.Connect("localhost:6379");
IDatabase db = redis.GetDatabase();

await db.StringSetAsync("producto:123:nombre", "Camiseta azul", TimeSpan.FromMinutes(30));
string? nombre = await db.StringGetAsync("producto:123:nombre");
```

## Lo que NO hace

- **No es un reemplazo de tu base de datos principal** — no tiene un lenguaje de consultas como SQL, ni relaciones, ni integridad referencial.
- **No garantiza persistencia total por defecto** — puede configurarse para guardar en disco (RDB, AOF), pero su caso de uso natural asume que perder la caché no es catastrófico: el origen de verdad sigue siendo la base de datos.
- **No escala bien para datos enormes por clave** — está pensado para valores pequeños y accesos rapidísimos, no para guardar documentos gigantes.

## Buenas prácticas avanzadas

- **Elige la estructura de datos correcta, no solo `String`** — guardar un objeto completo serializado en JSON dentro de un `String` es válido, pero un `Hash` te permite leer o actualizar un solo campo (`HGET`, `HSET`) sin traer ni reescribir el objeto entero. La estructura correcta ahorra ancho de banda y CPU.
- **Pon TTL a (casi) todo** — una clave sin expiración es una fuga de memoria en potencia. Salvo que tengas una razón explícita para que un dato viva para siempre en Redis, dale una caducidad razonable.
- **Evita el comando `KEYS` en producción** — recorre todo el *key-space* y bloquea el servidor mientras lo hace. Para explorar claves por patrón usa `SCAN`, que es incremental y no bloqueante.
- **Vigila el tamaño de las claves "calientes"** — una sola clave accedida por miles de peticiones por segundo (un contador global, por ejemplo) puede convertirse en cuello de botella aunque Redis en conjunto tenga capacidad de sobra; a veces conviene particionarla.
- **Usa *pipelining* para ráfagas de comandos** — si necesitas ejecutar muchos comandos seguidos, agruparlos en un *pipeline* evita pagar la latencia de ida y vuelta de red en cada uno por separado.

## Recursos didácticos

[try.redis.io](https://try.redis.io/) deja probar comandos de Redis en una consola interactiva desde el navegador, sin instalar nada.

---

*En resumen: Redis es una base de datos en memoria pensada para ser extremadamente rápida en lo simple — clave-valor, TTL y estructuras básicas — y ese enfoque estrecho es precisamente lo que la hace tan buena como caché compartida.*
