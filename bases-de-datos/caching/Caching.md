# Caching

## ¿Qué es?

El *caching* (cacheo) es la técnica de guardar una copia temporal de un dato costoso de obtener, para responder peticiones futuras sin repetir ese coste. La copia vive en un sitio más rápido de alcanzar que el origen: memoria en vez de disco, memoria local en vez de red.

## ¿Por qué existe?

Muchas operaciones son órdenes de magnitud más lentas que otras. Consultar una base de datos, llamar a una API externa o agregar un informe de miles de filas cuesta milisegundos o segundos; leer un valor de la memoria del proceso cuesta microsegundos. Si el mismo dato se pide una y otra vez sin cambiar entre medias, repetir el trabajo completo cada vez es tirar tiempo y recursos.

> Es como apuntar en un post-it el número de teléfono que acabas de buscar en la guía: la próxima vez miras el post-it en vez de recorrer la guía entera. El post-it puede quedarse desactualizado si el número cambia — ese es exactamente el problema que trae consigo la caché.

## ¿Cuándo y para qué se usa?

El caching aparece en cualquier sistema donde algunos datos **se leen mucho más de lo que cambian**:

- La página de inicio de una tienda online con sus "productos destacados": esa lista puede cachearse unos minutos en vez de recalcularla en cada visita.
- El resultado de una API externa de tipos de cambio, que solo se actualiza una vez al día.
- Un informe de ventas agregadas que tarda segundos en calcularse y que varias personas consultan a la misma hora.
- Los datos de perfil de la persona autenticada, consultados en casi cada petición.

No todo merece cachearse. Evita cachear datos que cambian en cada petición, datos muy personalizados y difíciles de reutilizar entre peticiones, o casos donde el coste de mantener la caché sincronizada supera el beneficio de ahorrarte la consulta original. Cachear mal, con datos obsoletos donde la frescura importa, es peor que no cachear. Hay una sección entera sobre esto más abajo: [qué no cachear](#qué-no-cachear).

Dos cosas que conviene fijar desde el principio, porque explican muchas decisiones posteriores:

- **La caché no es la fuente de verdad.** El dato real sigue viviendo en la base de datos. La caché es una copia desechable: si se borra entera, el sistema tiene que seguir funcionando (más lento, pero correcto).
- **La caché no arregla un mal diseño.** Cachear una consulta de 3 segundos la hace rápida a partir de la segunda vez, pero no arregla por qué era lenta la primera, y la primera le toca a alguien. Antes de cachear, mira si falta un índice.

En esta guía el ejemplo conductor es una tienda online con base de datos `TiendaDB`, tablas `Productos`, `Pedidos` y `Clientes`, y un pedido concreto —el **#4711**— que iremos siguiendo. Los ejemplos usan `IMemoryCache` de .NET; dónde conviene que viva la caché se trata en [Caché distribuida vs. local](Cache-Distribuida-vs-Local.md).

## El vocabulario, definido de una vez

Estos seis términos aparecen en el resto de la ficha y en toda la bibliografía del tema. Conviene tenerlos claros antes de seguir.

| Término | Qué significa |
|---|---|
| **Cache hit** | La clave pedida estaba en la caché y se sirve desde ahí, sin tocar el origen. |
| **Cache miss** | La clave no estaba: hay que ir al origen, y normalmente guardar el resultado para la próxima. |
| **Hit ratio** | `hits / (hits + misses)`. La métrica que decide si la caché sirve para algo. |
| **TTL** (*time to live*) | Tiempo de vida de una entrada. Al agotarse, la entrada se considera inválida sin que nadie la borre a mano. |
| **Eviction** (expulsión) | Sacar una entrada **antes** de que caduque, porque la caché se ha llenado y hay que hacer sitio. |
| **Staleness** (obsolescencia) | La distancia entre lo que dice la caché y lo que dice el origen ahora mismo. Cachear es aceptar cierta staleness a cambio de velocidad. |

La distinción entre **caducidad** (TTL, decidida por el tiempo) y **expulsión** (eviction, decidida por la presión de memoria) es la que más se confunde, y son cosas distintas: una entrada con un TTL de una hora puede desaparecer en treinta segundos si la caché se llena. Nunca escribas código que asuma que un dato guardado sigue ahí.

## El cálculo que justifica cachear

Aquí está el número que hace útil todo lo demás. Supongamos una consulta a `Productos` que tarda **50 ms** y un tráfico de **1000 peticiones por segundo** que la necesitan. Sin caché, la base de datos recibe las 1000 consultas por segundo. Con caché, recibe solo los *misses*:

| Hit ratio | Consultas/s a `TiendaDB` | Consultas simultáneas en vuelo | Latencia media percibida |
|---|---|---|---|
| 0 % | 1000 | 50 | 50 ms |
| 80 % | 200 | 10 | 10,1 ms |
| 95 % | 50 | 2,5 | 2,6 ms |
| 99 % | 10 | 0,5 | 0,6 ms |

Las consultas en vuelo salen de multiplicar consultas por segundo por su duración (200 × 0,050 s = 10), y esa columna es la que de verdad duele: **a 0 % de hit ratio necesitas 50 conexiones del pool ocupadas de forma permanente solo para esta consulta**, y un pool de .NET viene con 100 por defecto. La latencia media asume 0,1 ms para leer de una caché local.

Mira ahora el salto del 80 % al 95 %: de 200 a 50 consultas por segundo. **Cuatro veces menos carga por 15 puntos de hit ratio.** No es intuitivo, y la razón es que lo que paga la base de datos no es el hit ratio sino su complemento, el *miss ratio*: pasar del 20 % al 5 % de *misses* es dividir por cuatro. Del 95 % al 99 % vuelves a dividir por cinco con solo 4 puntos más.

De ahí salen dos consecuencias prácticas. **Una caché al 50 % no es "media caché", es apenas una mejora del doble**, y el trabajo de mantenerla probablemente no se paga. Y cuando ya estás en el 90 %, exprimir los puntos que faltan —afinar el TTL, precalentar las claves calientes— rinde mucho más que ampliar la caché a datos nuevos.

## Los patrones de caching

Los cinco patrones se distinguen por dos preguntas: **quién** habla con el origen y **cuándo** lo hace.

### Cache-aside (o *lazy loading*)

La aplicación es la que orquesta: consulta la caché, y si hay *miss*, va al origen y guarda el resultado. Es el patrón por defecto y el que verás en el 90 % del código.

```csharp
public async Task<Producto?> ObtenerProductoAsync(int id)
{
    var clave = $"producto:{id}";

    if (cache.TryGetValue(clave, out Producto? producto))
        return producto;                                    // cache hit

    producto = await repositorio.ObtenerPorIdAsync(id);      // cache miss
    if (producto is not null)
        cache.Set(clave, producto, TimeSpan.FromMinutes(10));

    return producto;
}
```

Devuelve el producto en microsegundos si estaba cacheado, y en ~50 ms la primera vez. La caché solo se llena con lo que alguien pide de verdad, así que no gastas memoria en datos que nadie consulta. El precio es que **el primer acceso a cada clave siempre es lento** y que el `if (producto is not null)` deja los "no existe" fuera de la caché — un agujero que se tapa con [caché negativa](#caché-negativa-cachear-el-no-existe).

### Read-through

La aplicación solo le habla a la caché; es la caché (o una capa que la envuelve) la que sabe cómo cargar el dato si falta. En .NET la aproximación más directa es `GetOrCreateAsync`:

```csharp
public Task<Producto?> ObtenerProductoAsync(int id) =>
    cache.GetOrCreateAsync($"producto:{id}", async entrada =>
    {
        entrada.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
        return await repositorio.ObtenerPorIdAsync(id);
    });
```

Es el mismo comportamiento que cache-aside, pero con la carga centralizada en un sitio: no puedes olvidarte de guardar el resultado ni poner un TTL distinto en cada llamada. La contrapartida es que la política de caché queda escondida en la infraestructura y cuesta más ver, leyendo la llamada, qué frescura te está dando.

### Write-through

Cada escritura actualiza el origen y la caché en la misma operación síncrona. La caché nunca queda por detrás. Este era el ejemplo original, y tiene dos defectos que merece la pena ver:

```csharp
// ❌ dos problemas en dos líneas
public async Task ActualizarPrecioAsync(int id, decimal nuevoPrecio)
{
    await repositorio.ActualizarPrecioAsync(id, nuevoPrecio);
    cache.Set($"producto:{id}", await repositorio.ObtenerPorIdAsync(id));
}
```

El primero es la consulta extra: acabas de escribir el dato, ya sabes cómo quedó, no hace falta volver a leerlo. El segundo es más grave: **un `Set` sin TTL crea una entrada que no caduca nunca**, así que si el proceso muere entre las dos líneas o alguien cambia el precio por fuera, ese precio equivocado se sirve hasta que la entrada sea expulsada o se reinicie el proceso.

```csharp
// ✅ una sola ida a la base de datos y una entrada que se cura sola
public async Task ActualizarPrecioAsync(int id, decimal nuevoPrecio)
{
    var producto = await repositorio.ActualizarPrecioAsync(id, nuevoPrecio); // devuelve la fila
    cache.Set($"producto:{id}", producto, TimeSpan.FromMinutes(10));
}
```

Orden importante: **primero el origen, después la caché**. Al revés, si el `UPDATE` falla, la caché queda anunciando un precio que nunca se guardó.

### Write-behind (o *write-back*)

La escritura se confirma solo en la caché y se propaga al origen de forma asíncrona, normalmente en lotes. Es el más rápido y el único que puede **perder datos**: si el proceso muere antes de vaciar la cola, esos cambios no existieron.

```csharp
// contador de visitas: se acumula en memoria y se vuelca cada 10 segundos
private readonly Channel<int> _visitas = Channel.CreateUnbounded<int>();

public void RegistrarVisita(int productoId) => _visitas.Writer.TryWrite(productoId);

protected override async Task ExecuteAsync(CancellationToken ct)
{
    var temporizador = new PeriodicTimer(TimeSpan.FromSeconds(10));
    while (await temporizador.WaitForNextTickAsync(ct))
    {
        var lote = new List<int>();
        while (_visitas.Reader.TryRead(out var id)) lote.Add(id);
        if (lote.Count > 0) await repositorio.IncrementarVisitasAsync(lote, ct); // UPDATE agrupado
    }
}
```

Convierte 1000 `UPDATE` por segundo en uno cada diez segundos. Úsalo solo donde perder los últimos segundos sea aceptable: contadores, telemetría, "última vez visto". Nunca para el importe del pedido #4711.

### Refresh-ahead

La caché se refresca **antes** de que caduque, de forma que ninguna petición llega a ver un *miss*. Tiene sentido para pocas claves, muy consultadas y caras de calcular:

```csharp
public sealed class RefrescoDestacados(IMemoryCache cache, IProductoRepositorio repositorio)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var destacados = await repositorio.ObtenerDestacadosAsync(ct);
            cache.Set("productos-destacados", destacados, TimeSpan.FromMinutes(10));
            await Task.Delay(TimeSpan.FromMinutes(4), ct);   // refresca antes del TTL
        }
    }
}
```

Con un refresco cada 4 minutos y un TTL de 10, la entrada nunca llega a expirar: el hit ratio es del 100 % y nadie paga la latencia del cálculo. El coste es que la consulta se ejecuta cada 4 minutos **aunque no la pida nadie**, así que no escala a miles de claves; y necesitas decidir qué pasa en el arranque, antes del primer refresco (lo habitual es combinarlo con cache-aside como red).

### Tabla de decisión

| Patrón | Latencia de lectura | Latencia de escritura | Riesgo de datos obsoletos | Riesgo de perder datos | Cuándo elegirlo |
|---|---|---|---|---|---|
| Cache-aside | Rápida, salvo el primer *miss* | Sin cambios | Medio (hasta el TTL) | Ninguno | Por defecto, para lecturas por identificador |
| Read-through | Igual que cache-aside | Sin cambios | Medio | Ninguno | Mismo caso, cuando quieres la política en un solo sitio |
| Write-through | Rápida y casi siempre fresca | Peor: dos escrituras | Bajo | Ninguno | Datos que se leen mucho justo después de escribirse |
| Write-behind | Rápida | La mejor de todas | Bajo en la caché, alto en el origen | **Alto** | Contadores y métricas que toleran perder segundos |
| Refresh-ahead | Siempre rápida | Sin cambios | Bajo y acotado | Ninguno | Pocas claves calientes y caras de calcular |

Se combinan: lo normal en producción es cache-aside para todo, refresh-ahead para las dos o tres claves más caras, y write-behind solo para contadores.

## Diseño de claves de caché

Una clave es un contrato: dice "todo lo que produjo este valor está aquí dentro". Se escribe con *namespacing*, segmentos separados por `:` de lo general a lo concreto: `producto:123`, `productos-destacados`, `cliente:88:pedidos`. Así se lee de un vistazo qué es cada entrada y se puede filtrar por prefijo al medir o al invalidar.

La regla dura es esta: **si dos peticiones que generan la misma clave pueden devolver resultados distintos, la clave está incompleta**. Y una clave incompleta no es un fallo de rendimiento: es un fallo de corrección por el que una persona ve los datos de otra.

```csharp
// ❌ la descripción depende del idioma, pero la clave no
var clave = $"producto:{id}";
var producto = await repositorio.ObtenerPorIdAsync(id, idioma);
cache.Set(clave, producto, TimeSpan.FromMinutes(10));
```

La primera visita en catalán deja cacheada la ficha en catalán bajo `producto:123`. La siguiente visita, en castellano, es un *hit* y devuelve el texto en catalán. Nadie ve una excepción; solo un cliente confuso. Cambia `idioma` por `tenantId` y el mismo bug pasa de ser molesto a ser una fuga de datos entre clientes.

```csharp
// ✅ la clave contiene todas las dimensiones que hacen variar el resultado
var claveFicha = $"producto:{idioma}:{id}";

// y en un listado, todas de verdad son todas:
var claveListado = $"tienda:{tenantId}:productos:{categoria}:{idioma}:p{pagina}:t{tamanoPagina}";
```

Cómo encontrar las dimensiones que faltan: recorre **todo** lo que entra en el cálculo, no solo los parámetros del método. Suelen colarse el idioma o la cultura, el identificador de *tenant*, el rol o los permisos del usuario, la divisa, los parámetros de paginación y ordenación, y la versión del contrato de salida (si el DTO cambia de forma, las entradas viejas se deserializan mal o de menos).

Dos avisos más. Cada dimensión que añades multiplica el número de claves posibles y divide el hit ratio: `producto:{idioma}:{id}` con tres idiomas triplica las entradas necesarias para el mismo ratio. Y **no metas datos personales en la clave** — las claves aparecen en logs, en volcados y en un `SCAN` cualquiera; usa el identificador interno, nunca el email. Para versionar claves y poder invalidar generaciones enteras cambiando el prefijo, ve a [Estrategias de invalidación](Estrategias-de-Invalidacion.md).

## TTL absoluto frente a TTL deslizante

`MemoryCacheEntryOptions` ofrece dos relojes distintos, y confundirlos es una de las causas más discretas de datos obsoletos.

```csharp
// absoluto: caduca 10 minutos después de guardarse, se pida o no
var absoluto = new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
};
cache.Set($"producto:{id}", producto, absoluto);

// deslizante: caduca si pasan 2 minutos sin que nadie la pida
var deslizante = new MemoryCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(2)
};
cache.Set($"cliente:{id}:carrito", carrito, deslizante);
```

El absoluto acota la staleness: pase lo que pase, el dato no tiene más de 10 minutos. El deslizante acota el desuso: las entradas que nadie mira se van solas y las calientes se quedan.

Y aquí está la trampa: **una entrada con solo `SlidingExpiration` y tráfico constante no caduca nunca**. Si `producto:123` se consulta cada 30 segundos y el deslizamiento es de 2 minutos, la ventana se renueva en cada lectura y ese precio puede llevar semanas cacheado. Justo las claves más populares, que son las que más importa tener frescas, son las que jamás se refrescan. La combinación de los dos es lo que casi siempre quieres:

```csharp
// ✅ se va si nadie la usa, y en cualquier caso no vive más de 15 minutos
var opciones = new MemoryCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(2),
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15)
};
```

Un detalle de implementación que sorprende al medir: en `IMemoryCache` la caducidad es **pasiva**. Nadie mira el reloj para borrar la entrada en el instante exacto; se descarta cuando alguien la pide o cuando pasa un barrido posterior. Una entrada caducada puede seguir ocupando memoria un buen rato, así que el TTL controla la frescura, no el consumo. Para eso hace falta un límite.

## Expulsión y límites de memoria

**Una caché en memoria sin límite de tamaño es una fuga de memoria con otro nombre.** El proceso crece mientras aparecen claves nuevas hasta que el contenedor toca su `memory limit` y el orquestador lo mata. Es especialmente fácil de provocar con claves derivadas de la entrada del usuario: un buscador que cachea `busqueda:{texto}` tiene infinitas claves posibles.

`IMemoryCache` no tiene límite por defecto. Se le pone así:

```csharp
services.AddMemoryCache(opciones =>
{
    opciones.SizeLimit = 10_000;              // 10 000 unidades de "tamaño"
    opciones.CompactionPercentage = 0.10;     // al llenarse, desaloja el 10 %
});
```

`SizeLimit` **no tiene unidades**: son las que tú decidas al declarar cada entrada. Lo más simple es contar entradas (`Size = 1` en todas, de modo que el límite son 10 000 entradas); si unas pesan mucho más que otras, aproxima bytes o número de elementos.

```csharp
cache.Set($"producto:{id}", producto, new MemoryCacheEntryOptions
{
    Size = 1,                                                    // una entrada, una unidad
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
});

// en una lista tiene más sentido cobrar por elemento
cache.Set("productos-destacados", destacados,
    new MemoryCacheEntryOptions { Size = destacados.Count }.SetSlidingExpiration(TimeSpan.FromMinutes(5)));
```

Y este es el aviso concreto: **en cuanto declaras `SizeLimit`, todas las entradas están obligadas a declarar su tamaño**. La que se olvide no se guarda silenciosamente, lanza:

```
InvalidOperationException: Cache entry must specify a value for Size when SizeLimit is set.
```

La excepción salta en el `Set`, en tiempo de ejecución, en el camino de código concreto que se olvidó. Un `Set` en una rama poco frecuente puede tardar semanas en explotar, y lo hará en producción. La defensa es no llamar nunca a `Set` a pelo: un método propio que construya las `MemoryCacheEntryOptions` con `Size` siempre puesto, y prohibir el resto por convención.

Cuando la caché se llena hay que decidir a quién se echa. Las políticas clásicas:

| Política | A quién expulsa | Cuándo encaja |
|---|---|---|
| **LRU** (*least recently used*) | A la que hace más tiempo que nadie pide | Por defecto: el pasado reciente predice el futuro inmediato |
| **LFU** (*least frequently used*) | A la que se ha pedido menos veces en total | Cuando hay un núcleo estable de claves calientes y ruido alrededor |
| **FIFO** | A la más antigua, se use o no | Casi nunca: simple, pero ignora el uso |

La compactación de `IMemoryCache` no es un LRU estricto: primero descarta lo caducado, después ordena por `CacheItemPriority` y solo dentro de cada prioridad va por uso reciente. Ojo con `CacheItemPriority.NeverRemove`: si lo pones "porque este dato es importante", esa entrada queda exenta de la expulsión y el límite deja de protegerte. Las políticas de un almacén distribuido se configuran de otra forma, y eso está en [Redis](Redis.md).

## El coste de la serialización

Cachear no es gratis, y el coste no siempre es visible. Una caché local guarda una referencia al objeto, así que leerla es prácticamente gratis. Una caché fuera del proceso obliga a **serializar al escribir y deserializar en cada lectura**, y ese trabajo se paga en CPU y en presión sobre el recolector de basura en todas las lecturas, incluidos los *hits*.

Cuando el objeto es pequeño y la consulta está indexada, eso puede costar más que la consulta que evitas. Antes de decidir, mídelo:

```csharp
var json = JsonSerializer.Serialize(destacados);
var sw = Stopwatch.StartNew();
for (var i = 0; i < 100; i++) JsonSerializer.Deserialize<List<Producto>>(json);
Console.WriteLine($"{json.Length} bytes | deserializar {sw.Elapsed.TotalMilliseconds / 100:F2} ms");
```

```
214883 bytes | deserializar 2,84 ms
```

Casi 3 ms por cada lectura de `productos-destacados`. Compáralo con lo que cuesta la consulta original: si el `SELECT` sale en 0,4 ms porque va por índice, esta caché es **siete veces más lenta** que no tenerla, y encima ha añadido un ida y vuelta de red. Si la consulta agrega media tabla y tarda 800 ms, los 3 ms son una ganga.

Dos cosas que la media esconde: un payload por encima de 85 000 bytes va al *Large Object Heap*, cuya recogida es mucho más cara, así que un objeto grande cacheado a alta frecuencia empeora las pausas de GC de todo el proceso; y lo que hay que mirar es el percentil 99, no la media, porque es ahí donde aparecen esas pausas. Mide con [BenchmarkDotNet](https://benchmarkdotnet.org/) si la decisión es importante y con un `Stopwatch` si solo quieres el orden de magnitud. La regla que se deduce es corta: **cachea lo caro de calcular, no lo caro de mover**. Un agregado que resume un millón de filas es el candidato perfecto; una fila por clave primaria, casi nunca.

## Caché negativa: cachear el "no existe"

En el cache-aside de más arriba, el `if (producto is not null)` significa que un identificador inexistente **nunca se cachea**. Una ráfaga de peticiones a `/productos/999999` golpea la base de datos en cada una, y da igual que sean un millón: el resultado es siempre el mismo "no existe" y siempre cuesta una consulta. Es un vector de ataque trivial y también un accidente común (un enlace roto indexado por un buscador).

La solución es cachear también la ausencia. Como `null` en la caché es ambiguo —cuesta distinguir "guardé que no existe" de "no hay entrada"—, lo más claro es envolver el resultado:

```csharp
private sealed record ResultadoProducto(Producto? Valor);

public async Task<Producto?> ObtenerProductoAsync(int id)
{
    var clave = $"producto:{id}";

    if (cache.TryGetValue(clave, out ResultadoProducto? cacheado))
        return cacheado!.Valor;                 // hit, exista el producto o no

    var producto = await repositorio.ObtenerPorIdAsync(id);

    cache.Set(clave, new ResultadoProducto(producto), new MemoryCacheEntryOptions
    {
        Size = 1,
        AbsoluteExpirationRelativeToNow = producto is null
            ? TimeSpan.FromSeconds(30)          // el "no existe" caduca pronto
            : TimeSpan.FromMinutes(10)
    });

    return producto;
}
```

El TTL corto del caso negativo no es un detalle: cuando alguien cree por fin ese producto, no puede estar diez minutos invisible. Treinta segundos bastan para absorber una ráfaga y son un retraso que nadie nota.

Y una precaución que rara vez se menciona: la caché negativa **es rellenable desde fuera**. Quien recorra identificadores al azar puede llenarla de entradas inútiles y expulsar las buenas, empeorando el hit ratio real justo mientras te felicitas por haber tapado el agujero. Con un `SizeLimit` puesto y un TTL negativo corto, el daño está acotado.

## Las capas de caché no son la misma cosa

"La caché" suele referirse a cinco cachés distintas que operan a la vez, y confundirlas lleva a buscar el problema en el sitio equivocado:

| Capa | Qué guarda | Quién la invalida | Qué problema resuelve |
|---|---|---|---|
| Navegador | Respuestas HTTP: imágenes, JS, JSON | Las cabeceras que mandó el servidor; el usuario con un recargado forzado | Peticiones que ni salen del dispositivo |
| CDN / proxy | Las mismas respuestas, compartidas entre usuarios | Igual, más un purgado explícito por API | Latencia geográfica y ancho de banda del origen |
| **Aplicación** (esta ficha) | Objetos ya construidos, en el proceso o en Redis | Tu código: TTL o borrado explícito | Consultas y cálculos repetidos |
| ORM (*first-level cache*) | Entidades ya materializadas en el `DbContext` | Se muere con el `DbContext`, normalmente al acabar la petición | La misma entidad pedida dos veces en la misma unidad de trabajo |
| Base de datos (*buffer pool*, *plan cache*) | Páginas de disco y planes de ejecución | El propio motor | Lectura física de disco y replanificación de SQL |

Tres consecuencias que se pagan cuando se ignoran. La primera: un *hit* en una capa alta impide que las de abajo vean la petición, así que tu hit ratio del 99 % puede convivir con una base de datos ahogada por el 1 % restante, o tu caché de aplicación puede parecer inútil porque la CDN ya se comió todo lo cacheable.

La segunda: **invalidar tu caché no invalida la del navegador**. Si respondes con `Cache-Control: max-age=3600`, el cliente guarda el precio del producto una hora y no tienes ninguna forma de alcanzarlo. Para eso existe `ETag`: el servidor manda una etiqueta con la respuesta, el navegador la reenvía en `If-None-Match` y recibe un `304 Not Modified` si nada cambió — la revalidación cuesta una petición, pero te devuelve el control. Es la capa HTTP y no se desarrolla aquí.

La tercera: el *first-level cache* del ORM ya está activo aunque no lo hayas pedido. En EF Core, pedir la misma entidad dos veces dentro del mismo `DbContext` ejecuta una sola consulta, y `AsNoTracking()` desactiva ese comportamiento. Más sobre esto en [Acceso a datos en .NET](../acceso-a-datos-dotnet/README.md).

## Qué no cachear

Cuatro casos donde la respuesta es no, y el segundo es el importante:

- **Datos que cambian en cada petición.** Un contador en vivo, un *token* de un solo uso, la hora. El hit ratio es cero por construcción y solo has añadido latencia.
- **Datos donde una lectura obsoleta tiene consecuencias.** El stock justo antes de confirmar el pedido #4711 es el ejemplo canónico. Con 3 unidades cacheadas 5 minutos y 40 personas comprando a la vez, las 40 pasan la validación y aceptas 40 pedidos para 3 unidades. Y la distinción fina: **cachear para mostrar sí, cachear para decidir no**. El "quedan pocas unidades" del listado puede ir cacheado; el descuento de stock al confirmar tiene que ser una operación transaccional contra `TiendaDB`, leyendo el valor real.
- **Datos muy personalizados con hit ratio previsiblemente cercano a cero.** Unas recomendaciones calculadas por persona, con 200 000 clientes que entran una vez a la semana, se cachean para nadie: la siguiente visita llega cuando la entrada ya caducó. Antes de cachear algo por usuario, estima el techo del ratio a mano — si cada persona hace 1,2 peticiones antes de que la entrada expire, el máximo alcanzable es 0,2 / 1,2 ≈ 17 %, y eso ya contando con que acierte siempre.
- **Cualquier cosa cuyo coste de mantener sincronizada supere el ahorro.** Si el dato se modifica desde catorce sitios del código, invalidarlo bien en los catorce es una deuda garantizada. Ahí gana un TTL corto, o no cachear.

Hay un quinto motivo, medio técnico y medio legal: los datos personales cacheados viven en un sitio más, con su propio ciclo de vida y sus propias copias. Si guardas la ficha del cliente en un almacén compartido, ese almacén entra en tus políticas de retención y de borrado.

## Observabilidad de la caché

Sin instrumentar es imposible saber si una caché ayuda; se cree que sí porque se escribió con esa intención. Las cinco métricas mínimas son **hits**, **misses**, **hit ratio**, **expulsiones** y **tamaño o número de entradas**. `IMemoryCache` las expone, pero hay que pedirlo:

```csharp
services.AddMemoryCache(opciones =>
{
    opciones.SizeLimit = 10_000;
    opciones.TrackStatistics = true;      // desactivado por defecto
});

// y en cualquier punto donde exportes métricas:
var e = ((MemoryCache)cache).GetCurrentStatistics()!;
var ratio = (double)e.TotalHits / (e.TotalHits + e.TotalMisses);
Console.WriteLine($"hits={e.TotalHits} misses={e.TotalMisses} ratio={ratio:P1} entradas={e.CurrentEntryCount}");
```

```
hits=1483 misses=47912 ratio=3,0 % entradas=9998
```

Así se ve una caché que no sirve para nada, y se lee en dos pasos. El ratio del 3 % dice que casi nada se reutiliza. Y `entradas=9998` sobre un límite de 10 000 dice por qué: la caché está llena y expulsando sin parar, así que las claves se van antes de que nadie las vuelva a pedir. La causa habitual es una clave con demasiadas dimensiones o una clave por usuario. Esta caché cuesta memoria, CPU y complejidad, y no devuelve nada: hay que quitarla o rediseñar la clave.

Dos refinamientos que separan una métrica útil de una decorativa. **Etiqueta las métricas por prefijo de clave**: un ratio global del 85 % puede ser `producto:` al 97 % y `busqueda:` al 4 %, y el promedio esconde exactamente lo que hay que arreglar. Y **vigila las expulsiones junto al ratio**: muchas expulsiones con ratio alto significa que el límite está bien ajustado y la caché trabaja al máximo; muchas expulsiones con ratio bajo es puro churn. La misma cifra significa cosas opuestas según con qué se lea.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| El hit ratio es del 3 % y nadie se había dado cuenta | Nunca se activó `TrackStatistics` ni se exportó la métrica. Casi siempre, claves demasiado específicas o expulsión constante por límite pequeño |
| Un cliente ve datos de otro, o la ficha sale en otro idioma | Clave incompleta: falta una dimensión (idioma, *tenant*, rol) que sí hace variar el resultado |
| La memoria del proceso crece hasta que el contenedor se reinicia | Caché sin `SizeLimit`, o con `CacheItemPriority.NeverRemove` en entradas que sí crecen |
| `InvalidOperationException: Cache entry must specify a value for Size when SizeLimit is set.` | Un `Set` que se olvidó de `Size` después de haber configurado `SizeLimit` |
| Un objeto cacheado aparece corrupto y nadie lo ha escrito | La caché en memoria devuelve **la misma instancia** a todos; alguien la mutó y el cambio quedó dentro de la caché |
| Alguien accede a datos que no le corresponden | Se cacheó el resultado *antes* de comprobar permisos: el segundo usuario recibe el *hit* sin pasar por la autorización |
| El dato cacheado no se refresca nunca aunque tenga TTL | `SlidingExpiration` sin `AbsoluteExpirationRelativeToNow` y tráfico constante: la ventana se renueva en cada lectura |
| Al desplegar, la base de datos se cae en el primer minuto | La caché arranca vacía: todo son *misses* a la vez. Hay que precalentar las claves calientes o desplegar de forma escalonada |
| Cuando caduca la clave más popular, sube el pico de CPU de la base de datos | *Cache stampede*: N peticiones simultáneas recalculan lo mismo. Se mitiga en [Estrategias de invalidación](Estrategias-de-Invalidacion.md) |
| Con caché la aplicación va más lenta que sin ella | El coste de serializar y deserializar supera el de la consulta evitada, o el hit ratio es tan bajo que casi todo paga las dos cosas |

El orden de la caché y la autorización merece una frase más, porque es el único de la lista que es un fallo de seguridad: la comprobación de permisos va **siempre antes** de tocar la caché, y si el resultado depende del rol, el rol va en la clave. Cachear la respuesta ya autorizada y servirla a quien pida la misma clave es exactamente cómo se filtran datos entre usuarios.

## Buenas prácticas avanzadas

- **Diseña las claves con namespacing y versión desde el primer día.** Una clave `v2:producto:{idioma}:{id}` te permite invalidar una generación entera cambiando el prefijo cuando cambia la *forma* del objeto cacheado, sin recorrer entradas una a una y sin que un despliegue nuevo deserialice datos viejos con un contrato que ya no existe. Añadir el prefijo después, con la caché en producción, es mucho más caro que ponerlo al principio.
- **Nunca caches objetos mutables por referencia.** Una caché en memoria devuelve la misma instancia a todo el que la pide: si alguien le cambia el precio a ese `Producto` para hacer un cálculo, acaba de corromper la caché para todos los demás sin pasar por ningún método de escritura, y no hay traza que lo explique. Guarda `record` inmutables o colecciones congeladas. El detalle contraintuitivo es que una caché distribuida no tiene este problema, porque deserializar entrega una copia nueva en cada lectura — la caché más rápida es la más peligrosa.
- **Distingue caducar de invalidar, y usa las dos.** Un TTL corto es fácil de razonar pero desperdicia *hits*; borrar la entrada exacta cuando el dato cambia es preciso pero exige acordarse en cada punto de escritura, y siempre hay uno que se olvida. La combinación robusta es invalidación activa donde la conoces y un TTL de seguridad detrás, que es lo que hace que un olvido se cure solo en minutos en vez de quedarse hasta el reinicio.
- **Mide el hit ratio por prefijo de clave antes de dar la caché por buena, y vuelve a medirla tres meses después.** Una caché nace con buen ratio y se degrada cuando alguien añade una dimensión a la clave o un cliente nuevo cambia el patrón de tráfico. El ratio agregado esconde estas cosas; el ratio por prefijo, no.
- **Haz que un fallo de la caché degrade el servicio, no que lo tumbe.** Si el almacén de caché no responde y tu código propaga la excepción, una caché caída se convierte en una aplicación caída, y eso es peor que no tener caché. Trata el error como un *miss* y ve al origen — pero acompáñalo de un límite de concurrencia o un *circuit breaker*, porque mandar de golpe el 100 % del tráfico a una base de datos dimensionada para el 5 % es la otra forma de caerse.
- **Asume que cualquier entrada puede desaparecer en cualquier momento.** Entre la expulsión por memoria, la caducidad y un reinicio, no hay ninguna garantía de que lo que guardaste hace un segundo siga ahí. Cualquier lógica que dependa de que la caché *recuerde* algo —un contador de intentos que decide un bloqueo, un paso intermedio de un proceso de pago— está usando como almacén persistente algo que por definición no lo es.

## Recursos didácticos

- [Interactive Latency Numbers](https://colin-scott.github.io/personal_website/research/interactive_latency.html) — una tabla interactiva con lo que cuesta leer de caché de CPU, de memoria, de SSD, de disco y de red, y cómo ha evolucionado por años. Es la forma más rápida de interiorizar los órdenes de magnitud que justifican todo el caching: si no ves de un golpe que la memoria es cien mil veces más rápida que un disco, las decisiones de esta ficha parecen arbitrarias.
- [Cache in-memory in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/caching/memory) — la referencia de `IMemoryCache`: todas las opciones de `MemoryCacheEntryOptions`, el comportamiento exacto de `SizeLimit`, la compactación, las prioridades y los *callbacks* de expulsión, que es donde se engancha la telemetría propia.
- [BenchmarkDotNet](https://benchmarkdotnet.org/) — para responder con datos a la pregunta de si cachear un objeto sale más caro que consultarlo. Mide con calentamiento, descarta valores atípicos y da percentiles y bytes asignados, que es justo lo que un `Stopwatch` a mano no te da.
- [Cache-Aside pattern](https://learn.microsoft.com/azure/architecture/patterns/cache-aside) — la descripción canónica del patrón y de sus problemas, con los enlaces a los demás patrones de la misma colección.

---

*En resumen: cachear es cambiar frescura por velocidad de forma deliberada — la clave no es cachear todo, sino saber qué datos aguantan estar un rato desactualizados, poner un límite de tamaño y medir el hit ratio para comprobar que la caché existe para algo.*
