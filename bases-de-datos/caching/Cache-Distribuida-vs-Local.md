# Caché Distribuida vs. Local

## ¿Qué es?

Son las dos formas de ubicar una caché. La **local** (o *in-process*) vive dentro de la memoria del propio proceso de la aplicación: es un diccionario más. La **distribuida** vive en un servicio aparte accesible por red —normalmente [Redis](Redis.md)— que todas las instancias comparten.

## ¿Por qué existe?

Con una sola instancia, la caché local gana sin discusión: no hay red, no hay serialización, no hay un servicio más que desplegar. El problema aparece con el escalado horizontal. En cuanto la misma aplicación corre en cuatro procesos detrás de un balanceador, hay **cuatro cachés independientes** que nadie sincroniza, y el dato que una instancia acaba de invalidar sigue vivo en las otras tres.

> Piensa en la caché local como la libreta que cada persona del equipo lleva en el bolsillo, y en la distribuida como la pizarra de la sala. Consultar la libreta es instantáneo, pero cuando el precio cambia hay que borrar cuatro libretas y nadie sabe qué apuntó cada cual. La pizarra obliga a levantarse —eso cuesta— pero solo hay una versión de la verdad.

Esta guía da por sabido qué es cachear y qué patrones existen: eso está en [Caching](Caching.md). El escenario de aquí en adelante es una API de tienda online desplegada en **cuatro instancias** detrás de un balanceador, sobre `TiendaDB` (`Productos`, `Pedidos`, `Clientes`).

## ¿Cuándo y para qué se usa?

- **Local**: una aplicación con una sola instancia, o datos que no necesitan estar sincronizados entre instancias — un catálogo de categorías, una tabla de traducciones, la configuración que se lee al arrancar.
- **Distribuida**: una API en varias instancias donde cada petición puede caer en cualquiera de ellas y necesita ver el mismo dato — la sesión, el carrito, cualquier cosa que sea estado de una persona concreta.
- **Las dos, en niveles**: la local como primera parada rapidísima y la distribuida detrás, compartida. Es el destino natural de casi cualquier sistema que crece, y tiene su propio precio.

## El problema, con números: un precio que baila

Esto es lo que justifica la guía entera, así que conviene verlo en concreto y no como advertencia genérica. El producto `producto:123` está cacheado en `IMemoryCache` con un TTL de 10 minutos. Las cuatro instancias han servido peticiones y **cada una tiene su propia copia** con precio 79,90 €. Alguien de la tienda baja el precio a 69,90 €: el `PUT /productos/123` cae en la **instancia A**, que actualiza `TiendaDB` y hace `cache.Remove("producto:123")`. A ha invalidado **su** caché; B, C y D no se han enterado de nada.

Ahora alguien recarga la ficha del producto cuatro veces seguidas. El balanceador reparte:

| Recarga | Instancia | Precio que se ve | Por qué |
|---|---|---|---|
| 1 | A | **69,90 €** | Invalidó y releyó de `TiendaDB` |
| 2 | C | 79,90 € | Su copia sigue viva, le quedan 7 minutos de TTL |
| 3 | A | **69,90 €** | Correcto otra vez |
| 4 | B | 79,90 € | Otra copia vieja, con otro TTL distinto |

El detalle importante no es que haya incoherencia: es su forma. **La ventana no es "un rato": es de hasta 10 minutos, y el precio no converge, oscila entre dos valores** según a qué instancia mande el balanceador cada recarga. Un dato viejo que se corrige a los diez minutos se puede tolerar. Un precio que aparece y desaparece al pulsar F5 se percibe como una aplicación rota, y genera incidencias imposibles de reproducir: quien las investiga desde su portátil pega en la misma instancia y lo ve siempre bien. Y no hay TTL corto que arregle esto de raíz: bajarlo a 30 segundos reduce la ventana pero multiplica por cuatro la carga contra `TiendaDB`, porque cada instancia relee por su cuenta.

## La tabla de decisión

| | Caché local (`IMemoryCache`) | Caché distribuida (Redis) |
|---|---|---|
| Latencia de lectura | Decenas o cientos de **nanosegundos** | 0,5 – 2 **milisegundos** (ida y vuelta) |
| Coherencia entre instancias | ❌ Ninguna: N copias divergentes | ✅ Una sola copia para todas |
| Sobrevive al reinicio del proceso | ❌ No: se vacía por completo | ✅ Sí, es un servicio aparte |
| Serialización | ✅ No hay: se guarda el objeto | ❌ Obligatoria en cada lectura y escritura |
| Tamaño práctico por entrada | Limitado por la memoria del proceso | Cómodo hasta ~100 KB; más, replantéalo |
| Copias del dato en memoria | Una por instancia (4 en el ejemplo) | Una |
| Complejidad operativa | Ninguna: es el propio proceso | Un servicio que desplegar, monitorizar, asegurar y respaldar |
| Si el almacén no responde | No aplica | Toda lectura de caché falla → hay que degradar al origen |
| Peligro principal | `OutOfMemoryException` / reinicio por OOM | Punto único de fallo y latencia añadida |

Las dos filas que suelen decidir en la práctica son la de coherencia (si el dato es estado de una persona, la local queda descartada) y la de complejidad operativa (si tienes una sola instancia, la distribuida es un servicio que operar a cambio de nada).

## El coste de la red, con números

Que la local sea "más rápida" es cierto pero inútil como criterio; la diferencia son **tres o cuatro órdenes de magnitud**, y eso sí lo es. Una lectura de `IMemoryCache` es una búsqueda en un `ConcurrentDictionary` dentro del mismo proceso: decenas o cientos de nanosegundos, sin salir de la CPU ni asignar memoria. Una lectura de Redis en la misma red local son 0,5 – 2 ms contando ida y vuelta: el `GET` en sí tarda microsegundos, el resto es red y syscalls.

Si una petición de la ficha de producto hace **20 lecturas de caché** (el producto, su stock, sus categorías, los destacados relacionados, la configuración de envíos…):

```text
Local:        20 × ~200 ns   =    0,004 ms   → invisible
Distribuida:  20 × ~1 ms     =   20 ms       → visible en el p99
```

Y sobre la red se apila la serialización. Deserializar un `Producto` pequeño ronda los pocos microsegundos, pero **asigna objetos**: el `byte[]`, las cadenas, el propio `Producto`. Veinte lecturas por petición y mil peticiones por segundo son 20 000 deserializaciones por segundo alimentando al recolector de basura. En la local no existe ninguna de esas asignaciones, porque el objeto que guardas es el objeto que recuperas.

La conclusión no es "usa local". Es que **20 lecturas distribuidas por petición es un problema de diseño**, y se resuelve agrupándolas con *pipelining* (ver [Redis](Redis.md)) o poniendo un nivel local delante.

## `IMemoryCache` en detalle

Se registra en el contenedor de dependencias y se inyecta como cualquier servicio:

```csharp
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 200;                 // ⚠️ ver más abajo
    options.CompactionPercentage = 0.20;     // cuánto expulsa al llegar al límite
});
```

Y estas son las tres operaciones que se usan a diario:

```csharp
// 1. Lectura explícita
if (cache.TryGetValue("producto:123", out Producto? cacheado))
    return cacheado;

// 2. Escritura con opciones
cache.Set("producto:123", producto, new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
    Size = 1,
    Priority = CacheItemPriority.Normal
});

// 3. Las dos cosas de una vez
return await cache.GetOrCreateAsync("producto:123", async entrada =>
{
    entrada.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
    entrada.Size = 1;
    return await db.Productos.FindAsync(123);
});
```

`TryGetValue` devuelve `false` tanto si la clave no existe como si ha expirado. `GetOrCreateAsync` es más corto, pero ojo: **no serializa el acceso**. Si cincuenta peticiones piden `producto:123` a la vez y no está en caché, se ejecutan cincuenta consultas contra `TiendaDB`. Eso es un *cache stampede* y se mitiga como se explica en [Estrategias de invalidación](Estrategias-de-Invalidacion.md).

### El límite de tamaño no es opcional

Aquí está el fallo que más veces tumba un proceso. `AddMemoryCache()` sin más **no tiene ningún límite**: la caché crece mientras haya claves nuevas que meter. El peligro real de la caché local no es "Redis se llena" —eso lo gestiona `maxmemory-policy`— sino que tu propio proceso se queda sin memoria y muere con un `System.OutOfMemoryException`.

En un contenedor la muerte es aún más silenciosa: el orquestador ve que el proceso supera su `memory limit` y lo mata (`OOMKilled`, código de salida 137). No hay excepción, no hay traza, solo un contenedor reiniciándose cada pocas horas y un pico de latencia cada vez que arranca con la caché vacía.

La solución es declarar `SizeLimit` **y** poner `Size` en cada entrada. Las unidades son las que tú decidas: si `SizeLimit = 200` y toda entrada vale `Size = 1`, el límite son 200 entradas; si cuentas bytes aproximados, es un presupuesto de memoria. Lo importante es ser consistente, porque si declaras el límite y olvidas el tamaño en una sola entrada:

```text
System.InvalidOperationException: Cache entry must specify a value for Size when SizeLimit is set.
```

Es un error de ejecución en el camino de una petición, no un aviso al arrancar: merece un test. Y para saber si el límite está bien puesto, instrumenta las expulsiones:

```csharp
entrada.RegisterPostEvictionCallback((clave, valor, razon, _) =>
    logger.LogInformation("Expulsada {Clave} por {Razon}", clave, razon));
```

`EvictionReason.Expired` es normal y sano. `EvictionReason.Capacity` significa que la caché está llena y expulsando cosas que aún eran válidas: tu tasa de aciertos se está hundiendo sin que ningún error lo denuncie.

## `IMemoryCache` guarda referencias, no copias

Esta es la trampa más sutil de la caché local, y no aparece en ningún tutorial de introducción. Cuando haces `cache.Set("producto:123", producto)`, lo que se guarda es **la referencia** al objeto. Todo el que lea esa clave recibe *el mismo* objeto, no una copia. Si alguien modifica una propiedad, ha modificado la caché para todas las peticiones siguientes, sin pasar por ningún método de escritura:

```csharp
// ❌ Producto es una clase con propiedades mutables
var producto = await servicio.ObtenerAsync(123);
producto.Precio *= 0.9m;          // "aplico el descuento para mostrarlo"
return View(producto);
// A partir de aquí, la caché sirve 71,91 € a todo el mundo.
// Y en la siguiente petición se aplica otro 10 % sobre el ya descontado.
```

Nadie escribió en la caché. No hay `Set`, no hay bug de invalidación, y el precio se degrada un 10 % por petición. Es el tipo de fallo que se investiga durante días.

```csharp
// ✅ Tipo inmutable: el compilador impide el error
public sealed record Producto(int Id, string Nombre, decimal Precio, int Stock);

var producto = await servicio.ObtenerAsync(123);
var conDescuento = producto with { Precio = producto.Precio * 0.9m };  // copia nueva
return View(conDescuento);
```

Las tres salidas válidas, en orden de preferencia: cachear un `record` o un tipo con `init`, cachear un DTO de solo lectura, o devolver una copia defensiva en cada lectura (lo peor de los tres, porque paga la copia siempre). Y aquí hay un argumento a favor de la distribuida que casi nadie menciona: **la distribuida no tiene este problema**. Como serializa, cada lectura devuelve un objeto recién construido; su coste de serialización compra, gratis, aislamiento entre peticiones.

## `IDistributedCache` en detalle

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "tienda:";     // prefijo para no colisionar con otras apps
});
```

La interfaz trabaja con `byte[]` y `string`, nunca con tus tipos. Eso no es un descuido: es lo que la hace independiente del backend, y es lo que te obliga a serializar el ida y vuelta a mano.

```csharp
var bytes = await cache.GetAsync("producto:123");
if (bytes is not null)
    return JsonSerializer.Deserialize<Producto>(bytes);

var producto = await db.Productos
    .Select(p => new Producto(p.Id, p.Nombre, p.Precio, p.Stock))   // proyección, no la entidad
    .FirstOrDefaultAsync(p => p.Id == 123);

await cache.SetAsync("producto:123", JsonSerializer.SerializeToUtf8Bytes(producto),
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
        SlidingExpiration = TimeSpan.FromMinutes(2)
    });
```

`SerializeToUtf8Bytes` evita el paso intermedio por `string` que hacen `SetStringAsync`/`GetStringAsync`: una asignación menos por operación. Y fíjate en el `Select`: se cachea una proyección, no la entidad de EF Core, por lo que se explica en los errores frecuentes.

Sobre las opciones, `AbsoluteExpirationRelativeToNow` es un techo duro y `SlidingExpiration` renueva el TTL en cada lectura; combinar las dos es lo habitual —"caduca a los 10 minutos como máximo, o a los 2 sin usarse"—, y conviene saber que la implementación de Redis emula el *sliding* renovando la expiración en cada `GetAsync`, lo que convierte cada lectura en lectura **y** escritura.

La limitación grande de `IDistributedCache` es que **solo sabe de claves y valores opacos**. No hay `HSET`, ni `ZADD`, ni `INCR`, ni pub/sub. Si necesitas incrementar un contador de visitas sin leer-modificar-escribir, o un *sorted set* para el top de ventas, tienes que bajar al cliente nativo:

```csharp
await multiplexer.GetDatabase()
    .HashIncrementAsync("producto:123:contadores", "visitas", 1);
```

Conviven sin problema: `IDistributedCache` para cachear objetos y `IConnectionMultiplexer` para lo demás. Las estructuras y sus comandos están en [Redis](Redis.md).

## La caché en dos niveles: L1 local + L2 distribuida

Casi todo el mundo acaba aquí, y por buenas razones: el L1 se come la latencia de las claves calientes y el L2 aporta coherencia y supervivencia al reinicio. La lectura recorre tres paradas:

```csharp
var clave = "producto:123";

// L1 — nanosegundos
if (memoria.TryGetValue(clave, out Producto? local))
    return local;

// L2 — milisegundos, compartido por las cuatro instancias
var bytes = await distribuida.GetAsync(clave);
if (bytes is not null)
{
    var deL2 = JsonSerializer.Deserialize<Producto>(bytes);
    memoria.Set(clave, deL2, opcionesL1);   // ⚠️ TTL de 30 s, mucho más corto que el L2
    return deL2;
}

// Origen — decenas de milisegundos
var producto = await LeerDeTiendaDbAsync(123);
await distribuida.SetAsync(clave, JsonSerializer.SerializeToUtf8Bytes(producto), opcionesL2);
memoria.Set(clave, producto, opcionesL1);
return producto;
```

La escritura va al revés: se actualiza `TiendaDB`, se borra el L2 y se borra el L1 **local**. Y ahí está el problema que introduce el patrón: **el L1 de las otras tres instancias sigue con el dato viejo**. Volvemos exactamente al precio que baila, solo que ahora con una ventana igual al TTL del L1 en lugar del TTL completo. Hay dos formas de acotarlo:

1. **TTL del L1 muy corto** (segundos, no minutos). No elimina la incoherencia, la reduce a algo tolerable, y no requiere infraestructura. Es la opción sensata para el 80 % de los casos.
2. **Avisar a las demás instancias por pub/sub.** Al invalidar, se publica en un canal de Redis; cada instancia está suscrita y borra su L1 al recibir el mensaje. Es lo correcto cuando la ventana no es tolerable, y tiene sus propios matices —el mensaje se puede perder, el emisor recibe su propio aviso, hay que versionar el formato—. Están en [Estrategias de invalidación](Estrategias-de-Invalidacion.md).

## `HybridCache`: lo anterior, ya resuelto

.NET 9 incorpora `HybridCache`, la abstracción oficial que hace exactamente este trabajo: L1 en memoria, L2 distribuido opcional, protección contra *stampede* de serie, serialización configurable e invalidación por etiquetas.

```csharp
builder.Services.AddHybridCache();     // usa el IDistributedCache registrado como L2

return await cache.GetOrCreateAsync(
    "producto:123",
    factory: token => LeerDeTiendaDbAsync(123, token),
    options: new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(10),             // TTL del L2
        LocalCacheExpiration = TimeSpan.FromSeconds(30)    // TTL del L1
    },
    tags: ["productos", "producto:123"],
    cancellationToken: ct);
```

Una llamada sustituye a las tres paradas del apartado anterior, y `await cache.RemoveByTagAsync("productos")` invalida el grupo completo. La diferencia técnica que importa: si veinte peticiones concurrentes piden la misma clave ausente, `HybridCache` ejecuta la *factory* **una vez** y las demás esperan el resultado — algo que `GetOrCreateAsync` de `IMemoryCache` no hace.

Y la frase honesta: **si empiezas hoy, empieza por aquí.** Combinar `IMemoryCache` e `IDistributedCache` a mano sigue siendo válido y es lo que verás en el código existente, pero es reimplementar —peor— lo que ya viene resuelto. Entiende las dos piezas por separado, que es de lo que va esta guía, y luego usa la abstracción.

## Qué dato va en qué nivel

| Dato | Nivel | Por qué |
|---|---|---|
| Catálogo de categorías | **Local**, TTL de horas | Cambia una vez al trimestre; cuatro copias idénticas no molestan a nadie |
| Configuración leída al arrancar | **Local**, sin TTL | Es inmutable durante la vida del proceso; un reinicio la recarga |
| `productos-destacados` | **Dos niveles** | Se lee en cada portada (justifica el L1) y cambia a diario (justifica el L2) |
| `producto:123` | **Dos niveles**, L1 de segundos | Muy leído; la incoherencia de precio se acota con un L1 corto |
| `carrito:cliente:42` | **Distribuida**, obligatorio | Es estado de una persona: no puede depender de a qué instancia caiga la petición |
| Sesión de usuario | **Distribuida**, obligatorio | Igual, y además debe sobrevivir a un despliegue |
| Respuesta de una API externa con cuota | **Distribuida** | Con caché local, cuatro instancias gastan cuatro veces la cuota del mismo dato |

La regla que resume la tabla: **si el dato es estado de alguien, distribuida; si es referencia compartida y estable, local; si es caliente y cambiante, los dos niveles.**

## Las *sticky sessions* no son la solución

Es la primera idea que se le ocurre a todo el mundo: si el problema es que la petición cae en otra instancia, que el balanceador mande siempre a la misma persona a la misma instancia (*session affinity*, normalmente con una cookie). Así la caché local vuelve a ser coherente para ella. Funciona, y rompe tres cosas:

- **El escalado.** El reparto deja de ser uniforme: una instancia puede acumular las sesiones pesadas mientras otra está ociosa. Y una instancia recién levantada no recibe tráfico existente, solo sesiones nuevas, así que tarda en ayudar justo cuando la necesitas.
- **El despliegue.** Cuando esa instancia se reinicia —y se reinicia en cada despliegue— toda la gente adherida a ella pierde su estado de golpe: carrito vacío, sesión caída, formulario a medias perdido.
- **El problema original solo a medias.** El precio de `producto:123` no es estado de nadie: es un dato compartido. Con afinidad, dos personas distintas siguen viendo precios distintos, indefinidamente.

La afinidad es un parche para *ocultar* estado que debería estar fuera del proceso. Si necesitas que el carrito sobreviva, la respuesta es sacarlo del proceso, no pegar a la persona a un proceso concreto.

## Qué pasa cuando el almacén distribuido no responde

Esta preocupación no existe en la caché local: si el proceso vive, su caché vive. Con la distribuida hay una dependencia de red más en el camino de cada petición, y hay que decidir qué se hace cuando falla:

```text
StackExchange.Redis.RedisTimeoutException: Timeout performing GET (5000ms),
inst: 0, qu: 0, qs: 6, aw: False, ...
```

Cinco segundos de espera para acabar sin dato. Si eso se propaga como excepción, una caída de Redis es una caída de tu API — con el agravante de que los hilos bloqueados esperando agotan el *thread pool* y la latencia se dispara incluso en los endpoints que no cachean nada. El patrón correcto es **degradar al origen**: si la caché no responde, se lee de `TiendaDB` y la petición se sirve más lenta, pero se sirve.

```csharp
try
{
    var bytes = await distribuida.GetAsync("producto:123");
    if (bytes is not null)
        return JsonSerializer.Deserialize<Producto>(bytes);
}
catch (RedisConnectionException ex) { logger.LogWarning(ex, "Caché no disponible"); }
catch (RedisTimeoutException ex)    { logger.LogWarning(ex, "Caché lenta"); }

return await LeerDeTiendaDbAsync(123);   // el origen sigue ahí
```

Cachear defensivamente es distinto de cachear. Cachear es "guardo esto para no repetir trabajo". Cachear defensivamente es aceptar que **la caché es un optimizador, no una dependencia**: ningún fallo suyo, ni de lectura ni de escritura, debe convertirse en un error de la petición. Dos cautelas que lo completan: baja el `syncTimeout` a algo como 500 ms (esperar cinco segundos por un dato opcional no tiene sentido) y ten presente que si Redis cae, **todo** el tráfico va al origen a la vez — el mismo *stampede* de [Estrategias de invalidación](Estrategias-de-Invalidacion.md), pero global.

## Cuándo la local es suficiente

Añadir un servicio a la arquitectura tiene un coste permanente: desplegarlo, monitorizarlo, ponerle contraseña y TLS, actualizarlo, respaldarlo si guarda algo que importa, y responder a sus incidencias de madrugada. No lo pagues sin necesidad. La caché local basta cuando se cumple alguna de estas:

- **Una sola instancia.** No hay divergencia posible. Y si mañana hay dos, la migración es real pero acotada.
- **Datos inmutables o casi.** Catálogos, traducciones, tablas maestras, configuración: si el dato cambia con un despliegue, el reinicio del proceso ya es la invalidación.
- **Una ventana de incoherencia de minutos es aceptable.** Un contador de "productos vistos hoy" que difiera entre instancias no molesta a nadie. Sé honesto en este juicio: para un precio o un stock la respuesta es no.
- **El dato no es de nadie en particular.** En cuanto sea estado de una persona, la local queda descartada sin más discusión.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| Un `static Dictionary<string, Producto>` como caché y el proceso creciendo sin parar | No tiene límite ni TTL: nada expulsa nunca. Además no es seguro para acceso concurrente — usa `IMemoryCache` con `SizeLimit` |
| El carrito aparece vacío al recargar, y con otro F5 vuelve | Carrito o sesión en caché local con balanceador: la petición cayó en otra instancia. Va en la distribuida, siempre |
| El contenedor se reinicia cada pocas horas, sin excepción en los logs | `IMemoryCache` sin `SizeLimit`: el orquestador mata el proceso por exceder su límite de memoria (`OOMKilled`, exit 137) |
| `InvalidOperationException: Cache entry must specify a value for Size when SizeLimit is set.` | Declaraste `SizeLimit` y una entrada no pone `Size`. Basta con que se te escape un `Set` |
| `JsonException: A possible object cycle was detected` al escribir en la distribuida | Estás serializando una entidad de EF Core con propiedades de navegación (`Pedido.Cliente.Pedidos`). Cachea un DTO o un `record` proyectado, nunca la entidad |
| El precio cacheado se degrada un poco en cada petición | La caché local devuelve referencias y alguien mutó el objeto. Cachea tipos inmutables |
| El precio salta entre dos valores al recargar | Caché local en varias instancias, invalidada solo en la que atendió la escritura |
| La API entera se pone lenta cuando Redis tiene un problema | No hay `try`/`catch` que degrade al origen, y el `syncTimeout` por defecto de 5 s bloquea hilos |
| Tras invalidar, unas instancias sirven el dato nuevo y otras el viejo durante minutos | L1 con TTL largo y sin pub/sub. Baja el TTL del L1 a segundos o avisa a las demás |
| La cuota de una API externa se agota cuatro veces más rápido de lo previsto | Su respuesta está cacheada en local: cada instancia gasta su propia cuota |

## Buenas prácticas avanzadas

- **Mide la ganancia neta antes de meter un almacén distribuido.** Si el origen es una consulta indexada que tarda 2 ms, cachearla en Redis a 1 ms de ida y vuelta más serialización no gana casi nada, y te deja un servicio que operar y un modo de fallo nuevo. Existe una franja —orígenes rápidos, datos poco reutilizados— donde la caché distribuida empeora el sistema. Compara el p99 con y sin, no la media.
- **El TTL del L1 debe ser un orden de magnitud menor que el del L2.** Es la palanca que acota la incoherencia sin montar pub/sub: 30 segundos en el L1 y 10 minutos en el L2 significan que la ventana peor es de 30 segundos, no de 10 minutos, y a cambio el L2 sigue absorbiendo casi toda la carga del origen. Quien pone el mismo TTL en los dos niveles paga la complejidad de dos niveles y se queda con la incoherencia de uno.
- **Vigila `EvictionReason.Capacity`, no solo la tasa de aciertos.** Una caché local que expulsa por capacidad sigue funcionando y sigue devolviendo datos correctos: solo es cada vez menos útil, y ningún error lo denuncia. Registra las razones de expulsión con `RegisterPostEvictionCallback` y trata `Capacity` como señal de que hay que subir el `SizeLimit` o revisar qué se está cacheando de más.
- **No mezcles en el mismo Redis lo que es caché y lo que no puedes perder.** Una caché quiere `maxmemory-policy allkeys-lru`, es decir: expulsa lo que haga falta cuando te llenes. Si en esa misma instancia guardas locks distribuidos, colas o sesiones, la política los expulsará también, en el peor momento y sin avisar. Instancias o bases distintas, políticas distintas.
- **Trata el fallo de la caché como camino esperado, no como excepción.** Timeout agresivo, `try`/`catch` en lectura y escritura, y un *circuit breaker* que deje de intentarlo tras varios fallos seguidos en lugar de gastar 500 ms por petición en llamar a un servicio que está caído. Prueba el escenario de verdad: para Redis en local y comprueba que tu API sigue respondiendo.

## Recursos didácticos

- [Latency Numbers Every Programmer Should Know](https://colin-scott.github.io/personal_website/research/interactive_latency.html) — visualización interactiva de los órdenes de magnitud: acceso a memoria, red local, disco. Es la forma más rápida de internalizar por qué la diferencia entre local y distribuida son cuatro ceros y no un porcentaje.
- [`redis-cli --latency-history`](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency-monitor/) — mide la latencia real de ida y vuelta contra *tu* Redis, desde *tu* red. Los 0,5 – 2 ms de esta guía son una referencia; el número que importa para decidir es el tuyo, y un contenedor mal ubicado puede dar 15 ms.
- [BenchmarkDotNet](https://benchmarkdotnet.org/) — mide el coste de serializar tus propios objetos y compara `IMemoryCache` frente a `IDistributedCache` con datos reales. Sorprende cuánto pesa la deserialización cuando la entrada es grande.
- [Documentación de `HybridCache`](https://learn.microsoft.com/aspnet/core/performance/caching/hybrid) — la referencia oficial del patrón de dos niveles, con la configuración de serializadores y el detalle de la invalidación por etiquetas.

---

*En resumen: la caché local es tres o cuatro órdenes de magnitud más rápida pero solo la ve una instancia, y la distribuida es coherente y sobrevive al reinicio a cambio de red y serialización — el dato que es de alguien va siempre en la distribuida, y en cuanto quieras las dos ventajas la respuesta es `HybridCache`, no dos cachés pegadas a mano.*
