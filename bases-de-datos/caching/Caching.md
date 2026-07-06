# Caching

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

El *caching* (cacheo) es la técnica de guardar una copia temporal de un dato costoso de obtener, para poder responder peticiones futuras sin repetir ese coste. La copia vive en un lugar más rápido de acceder que el origen original (memoria en vez de disco, memoria en vez de red).

## ¿Por qué existe?

Muchas operaciones son mucho más lentas que otras: consultar una base de datos, llamar a una API externa o calcular un informe con miles de filas cuesta milisegundos o segundos, mientras que leer un valor de memoria cuesta microsegundos. Si el mismo dato se pide una y otra vez sin cambiar entre medias, repetir el trabajo completo cada vez es desperdiciar tiempo y recursos.

> Es como apuntar en un post-it el número de teléfono que acabas de buscar en la guía: la próxima vez que lo necesites, miras el post-it en vez de volver a buscar en la guía entera. El post-it puede quedarse desactualizado si el número cambia — ese es exactamente el problema que resuelve la caché.

## ¿Cuándo y para qué se usa?

El caching aparece en cualquier sistema donde algunos datos se leen mucho más de lo que cambian:

- La página de inicio de una tienda online que muestra "productos destacados": esa lista puede cachearse unos minutos en vez de recalcularla en cada visita.
- El resultado de una llamada a una API externa de tipos de cambio de moneda, que solo se actualiza una vez al día.
- Un informe de ventas agregadas que tarda varios segundos en calcularse y que varias personas consultan a la misma hora.
- Los datos de sesión o el perfil de una persona usuaria, consultados en casi cada petición de una aplicación.

No todo merece cachearse. Evita cachear datos que cambian en cada petición (el saldo exacto de una cuenta bancaria justo antes de una operación), datos muy personalizados y difíciles de reutilizar entre peticiones, o casos donde el coste de mantener la caché sincronizada supera el beneficio de ahorrarte la consulta original. Cachear mal, con datos obsoletos donde la frescura importa, es peor que no cachear.

## Lo mínimo que necesitas saber

**1. Cache-aside (o *lazy loading*): el patrón más común**

La aplicación consulta primero la caché; si el dato no está (*cache miss*), lo busca en el origen y lo guarda en la caché para la próxima vez.

```csharp
public async Task<Producto?> ObtenerProductoAsync(int id)
{
    var claveCache = $"producto:{id}";

    if (cache.TryGetValue(claveCache, out Producto? producto))
        return producto; // cache hit

    producto = await repositorio.ObtenerPorIdAsync(id); // cache miss
    if (producto is not null)
        cache.Set(claveCache, producto, TimeSpan.FromMinutes(10));

    return producto;
}
```

**2. Write-through: escribir en caché y origen a la vez**

Cada escritura actualiza la base de datos y la caché en la misma operación, de forma síncrona. La caché nunca queda desincronizada, a costa de que cada escritura sea un poco más lenta.

```csharp
public async Task ActualizarPrecioAsync(int id, decimal nuevoPrecio)
{
    await repositorio.ActualizarPrecioAsync(id, nuevoPrecio);
    cache.Set($"producto:{id}", await repositorio.ObtenerPorIdAsync(id));
}
```

**3. Write-behind (o *write-back*): escribir en caché y diferir el origen**

La escritura se confirma solo en la caché, y de forma asíncrona (en lote, con un pequeño retraso) se propaga al origen. Es el más rápido de los tres, pero también el que más riesgo tiene: si el proceso muere antes de propagar, esos cambios se pierden.

**4. TTL (*time to live*): la caducidad automática**

Cada entrada de caché puede tener un tiempo de vida tras el cual se considera obsoleta y se elimina o se vuelve a calcular, sin que nadie tenga que borrarla a mano.

```csharp
cache.Set("productos-destacados", productos, TimeSpan.FromMinutes(5));
```

**5. Cache hit / cache miss**

Un *hit* es cuando el dato pedido está en caché; un *miss*, cuando no está y hay que ir al origen. La proporción de *hits* sobre el total de peticiones (*hit ratio*) es la métrica clave para saber si una caché está siendo útil.

## Lo que NO hace

- **No es la fuente de verdad** — el dato "real" sigue viviendo en la base de datos o el sistema de origen; la caché es solo una copia desechable.
- **No garantiza datos frescos** — por diseño, acepta servir datos ligeramente desactualizados a cambio de velocidad. Si necesitas el dato exacto en el instante presente, no lo caches.
- **No resuelve un mal diseño** — cachear una consulta lenta la hace rápida la segunda vez, pero no arregla por qué era lenta la primera.

## Buenas prácticas avanzadas

- **Diseña las claves de caché con namespacing y versión** — una clave como `v2:producto:123` te permite invalidar toda una "generación" de datos cambiando el prefijo (`v2` → `v3`) sin tener que borrar entrada por entrada cuando cambia la forma del dato cacheado.
- **Cuidado con el *cache stampede*** — si una entrada muy solicitada caduca y llegan mil peticiones a la vez, las mil pueden intentar recalcularla simultáneamente y tumbar el origen. Se profundiza en [Estrategias de invalidación](Estrategias-de-Invalidacion.md).
- **No caches objetos mutables por referencia** — si tu caché en memoria del proceso devuelve la misma instancia a todo el que la pide, y alguien modifica esa instancia, acabas de corromper la caché para todos los demás sin pasar por ningún método de escritura. Devuelve copias o usa tipos inmutables.
- **Distingue invalidación por tiempo (TTL) de invalidación activa** — un TTL corto es fácil de razonar pero desperdicia *hits*; una invalidación activa (borrar la entrada exacta cuando el dato cambia) es más precisa pero exige disciplina para no olvidar ningún punto de escritura.
- **Mide el *hit ratio* antes de dar la caché por buena** — una caché con un 10% de aciertos añade complejidad y puede que ni siquiera compense la latencia extra de consultarla. Sin medir, es fácil creer que una caché ayuda cuando en realidad apenas se usa.

---

*En resumen: cachear es cambiar frescura por velocidad de forma deliberada — la clave no es cachear todo, sino saber qué datos aguantan estar un rato desactualizados.*
