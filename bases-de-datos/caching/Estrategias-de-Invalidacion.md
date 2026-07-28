# Estrategias de Invalidación de Caché

## ¿Qué es?

La invalidación de caché es el conjunto de técnicas para decidir **cuándo una copia cacheada deja de valer** y hay que borrarla o refrescarla, en lugar de seguir sirviéndola desactualizada.

## ¿Por qué existe?

Cachear es la parte fácil: guardas el resultado y lo devuelves. El problema aparece cuando el dato de origen cambia y la copia no se entera. Nadie lanza una excepción, ningún test se pone rojo, ninguna alerta salta: la aplicación simplemente miente, y lo hace en silencio hasta que alguien de negocio pregunta por qué el precio de la ficha no coincide con el del carrito.

> Hay una frase muy citada en programación: *"solo hay dos cosas difíciles en informática: invalidación de caché y poner nombres a las cosas"*. Es una broma, pero la primera mitad se sostiene por una razón concreta: cachear es una decisión local (una función, una clave) mientras que invalidar es una decisión **global** — exige saber todos los sitios del sistema que dependen de ese dato. Y ese conocimiento no cabe en ninguna función.

## ¿Cuándo y para qué se usa?

Esta ficha da por sabido qué es cachear, qué es un *hit* y un *miss* y en qué consisten los patrones cache-aside y write-through: eso está en [Caching](Caching.md). Aquí se trata solo el problema de la frescura.

Todo sistema con caché tiene una estrategia de invalidación, aunque sea implícita — "esperar a que caduque" también es una estrategia, solo que sin decidirla. La pregunta útil no es "¿invalido o no?" sino **"¿cuántos segundos de dato viejo puede tolerar este caso concreto?"**, porque la respuesta cambia radicalmente el diseño.

El ejemplo conductor de esta ficha es una tienda online sobre `TiendaDB`, con las tablas `Productos`, `Pedidos` y `Clientes`. Las claves de caché serán `producto:123`, `productos-destacados`, `categoria:ropa:productos` y `cliente:42:pedidos-activos`, y el pedido que iremos siguiendo es el **#4711**.

## El espectro de frescura: la tabla de decisión

Esta tabla es el corazón de la ficha. No hay una estrategia correcta: hay cinco puntos de un espectro que cambia coste por frescura.

| Estrategia | Datos obsoletos hasta | Acoplamiento que introduce | Coste de mantenerlo | Cuándo elegirla |
|---|---|---|---|---|
| **TTL corto** (5-60 s) | El TTL completo | Ninguno | Casi nulo | Datos que cambian solos y sin evento claro (contadores, agregados, respuestas de APIs externas) |
| **TTL largo** (10 min-24 h) | El TTL completo, y se nota | Ninguno | Casi nulo | Datos casi inmutables: catálogo de países, tipos de IVA, traducciones |
| **Invalidación activa al escribir** | Milisegundos | Alto — cada punto de escritura debe conocer las claves afectadas | Medio, y crece con cada vista cacheada nueva | Hay un punto de escritura claro y pocas vistas del dato |
| **Invalidación por evento** (bus, *outbox*, CDC) | Latencia del bus (ms-s) | Bajo entre módulos, a cambio de infraestructura | Alto: hace falta bus, reintentos y observabilidad | Varios servicios cachean el mismo dato, o el que escribe no debe conocer a quien cachea |
| **Versionado de claves** | Milisegundos, para miles de claves a la vez | Bajo — solo hay que saber a qué grupo pertenece la clave | Bajo, una vez montado | Invalidaciones masivas: recarga de catálogo, cambio de formato del objeto cacheado |

Dos lecturas que conviene fijar. Primero: **las dos filas de TTL no tienen acoplamiento porque no tienen conocimiento**, y eso es su gran virtud, no un defecto — un TTL no se olvida de nada, mientras que la invalidación activa se olvida en cuanto alguien añade una vista cacheada y no actualiza el punto de escritura. Segundo: **las filas no se excluyen**; lo normal en producción es la tercera *más* la primera.

## TTL: dejar que caduque sola

La opción más simple: la entrada nace con fecha de caducidad y nadie tiene que acordarse de borrarla.

```csharp
cache.Set($"producto:{id}", producto, TimeSpan.FromMinutes(5));
```

Ventaja: cero coordinación. Desventaja: durante esos cinco minutos, un cambio real en `Productos` no se refleja — si el precio cambia en el segundo 1, hay 299 segundos de precio equivocado en la ficha pública.

Un matiz que se cuela a menudo: el TTL puede ser **absoluto** (caduca cinco minutos después de guardarse, se lea o no) o **deslizante** (cada lectura reinicia la cuenta). El deslizante suena mejor y casi nunca lo es para datos de negocio: una clave leída cada segundo nunca caduca, así que su dato puede quedarse viejo indefinidamente. Reserva el deslizante para lo que de verdad es una sesión. La distinción está detallada en [Caching](Caching.md).

## Invalidación activa: y el acoplamiento que trae debajo

El código que modifica el dato borra también su entrada de caché.

```csharp
public async Task ActualizarPrecioAsync(int id, decimal nuevoPrecio)
{
    await repositorio.ActualizarPrecioAsync(id, nuevoPrecio);
    cache.Remove($"producto:{id}"); // se recalculará en la próxima lectura
}
```

Esto es correcto y es lo primero que escribe todo el mundo. Fíjate en el orden: **primero la base de datos, después la caché**. Al revés hay una carrera con final feliz aparente y datos viejos reales, y aparece en la tabla de errores frecuentes.

El problema de verdad no es el orden, es lo que pasa seis meses después. Ese `producto:123` no es la única copia del precio en la caché:

```
producto:123                  → la ficha completa del producto
productos-destacados          → la lista de la portada, con precio incluido
categoria:ropa:productos      → el listado de la categoría, con precio incluido
```

Cambiar un precio debería invalidar **las tres**, así que al `Remove` de arriba le faltan dos líneas: `cache.Remove("productos-destacados")` y `cache.Remove($"categoria:{producto.Categoria}:productos")`. Pero `ActualizarPrecioAsync` solo sabe de la primera, porque cuando se escribió las otras dos no existían — y quien añadió la portada cacheada no tenía ninguna razón para ir a tocar el método que actualiza precios.

Ese es el acoplamiento: cada punto de escritura tiene que conocer el mapa completo de claves derivadas. Con dos vistas se aguanta; con cinco, siempre hay una que alguien olvidó, y suele ser la de la portada — la más visible. Las tres salidas a este problema ocupan las secciones siguientes: **TTL de seguridad** (asume el olvido y limita su duración), **versionado** (invalida el grupo entero sin enumerarlo) y **dependencias** (que la clave derivada declare de quién depende, en vez de que el escritor lo adivine).

## Combinar ambas: activa como mecanismo, TTL como red de seguridad

Es el consejo más práctico de toda la ficha. Invalida activamente en cada escritura que conozcas **y además** pon un TTL razonable a todo.

```csharp
cache.Remove($"producto:{id}");                                   // fino: frescura inmediata en lo conocido
cache.Set($"producto:{id}", producto, TimeSpan.FromMinutes(10));  // sucio: acota lo que no conocías
```

La lógica es de gestión de riesgo, no de rendimiento: la invalidación activa acierta en el 95 % de los casos y el TTL convierte el 5 % restante en "diez minutos de dato viejo" en lugar de "dato viejo hasta el próximo despliegue". Una caché sin TTL es una caché en la que un solo olvido dura para siempre. Corolario incómodo: **si te encuentras subiendo el TTL para ganar *hit ratio*, estás subiendo también la duración de tus futuros bugs.** Ese es el intercambio real que estás firmando.

## Versionado de claves: invalidar miles de entradas de golpe

Cambia el formato del objeto `Producto` cacheado, o se recarga el catálogo entero desde el ERP. Hay 40 000 claves `producto:*` que ya no valen. El primer impulso es este:

```bash
# ❌ nunca en producción
redis-cli --scan --pattern 'producto:*' | xargs redis-cli DEL
```

`KEYS producto:*` es peor todavía: recorre **todo** el espacio de claves y Redis atiende comandos en un solo hilo, así que durante ese recorrido el servidor no responde a nadie más. Con millones de claves eso son segundos de caché caída para toda la aplicación. `SCAN` no bloquea, pero sigue siendo un ir y venir de miles de peticiones y no es atómico: mientras recorres, alguien está escribiendo claves nuevas que no verás. La técnica que resuelve esto es el **prefijo generacional**, un contador que forma parte de la clave:

```csharp
private async Task<long> ObtenerGeneracionAsync(string grupo)
{
    // se cachea en local unos segundos para no pagar un viaje a Redis por lectura
    if (memoria.TryGetValue($"gen:{grupo}", out long gen)) return gen;
    gen = (long)await db.StringGetAsync($"gen:{grupo}");
    memoria.Set($"gen:{grupo}", gen, TimeSpan.FromSeconds(5));
    return gen;
}

public async Task<Producto?> ObtenerProductoAsync(int id)
{
    var gen = await ObtenerGeneracionAsync("productos");
    var clave = $"v{gen}:producto:{id}";   // → v7:producto:123
    // ... cache-aside normal contra esa clave
}

// invalidar las 40 000 claves de golpe:
public Task InvalidarCatalogoAsync() => db.StringIncrementAsync("gen:productos");
```

Un solo `INCR`, O(1), atómico. La generación pasa de 7 a 8, todas las lecturas empiezan a pedir `v8:producto:*` y las 40 000 claves `v7:*` quedan **huérfanas**: nadie las va a preguntar nunca más. De ahí sale el detalle que hace o rompe la técnica: **las claves viejas siguen ocupando memoria**, y solo desaparecen cuando caduca su TTL o cuando Redis las expulsa por presión de memoria. Con versionado, el TTL no es una red de seguridad opcional: es lo único que impide que la memoria crezca una generación tras otra. Y los cinco segundos de caché local sobre el contador también tienen su precio: durante esos cinco segundos una instancia puede seguir leyendo la generación vieja. Si eso no es aceptable, quita la caché local y paga el viaje extra a Redis.

## Invalidación por dependencias: cuando la entrada agrega varias entidades

`cliente:42:pedidos-activos` no depende de una fila, sino de todos los pedidos de ese cliente. Cuando se cancela el pedido **#4711**, el código que lo cancela no tiene por qué saber que existe una vista cacheada de pedidos activos — y no debería tener que saberlo.

Se invierte la dirección del conocimiento: **la clave derivada declara de quién depende** en el momento de cachearse.

```csharp
// al cachear la vista agregada, registramos las entidades de las que depende
await db.StringSetAsync("cliente:42:pedidos-activos", json, TimeSpan.FromMinutes(10));
foreach (var pedidoId in new[] { 4711, 4712, 4715 })
{
    await db.SetAddAsync($"dep:pedido:{pedidoId}", "cliente:42:pedidos-activos");
    await db.KeyExpireAsync($"dep:pedido:{pedidoId}", TimeSpan.FromHours(1));
}
```

Y al cancelar el pedido, el escritor solo anuncia *qué entidad* ha cambiado; la caché resuelve el resto:

```csharp
public async Task CancelarPedidoAsync(int pedidoId)   // pedidoId = 4711
{
    await repositorio.CancelarAsync(pedidoId);
    var dependientes = await db.SetMembersAsync($"dep:pedido:{pedidoId}");
    if (dependientes.Length > 0)
        await db.KeyDeleteAsync(dependientes.Select(d => (RedisKey)d.ToString()).ToArray());
    await db.KeyDeleteAsync($"dep:pedido:{pedidoId}");
}
```

Devuelve las claves que dependían del #4711 —aquí `cliente:42:pedidos-activos`— y las borra en un único `DEL` múltiple. El TTL de una hora en el conjunto `dep:*` evita que los registros de dependencias se acumulen para siempre.

Cuándo **no** montar esto: si tu grafo de dependencias son dos claves fijas, esta maquinaria es peor que escribir los dos `Remove` a mano; vale la pena solo cuando las dependencias son dinámicas y no se pueden enumerar en el código. Y si estás en .NET 9 o superior, `HybridCache` ya trae etiquetas y `RemoveByTagAsync`: no lo escribas tú.

## Varias instancias: tu `Remove` no llega a las otras cuatro

Aquí aparece el fallo que más desconcierta al pasar de una instancia a cinco. Tienes caché local (en memoria del proceso) y llamas a `cache.Remove("producto:123")`. Funciona: en *tu* proceso. Las otras cuatro instancias siguen sirviendo el precio viejo hasta que caduque su copia, y como el balanceador reparte, la misma persona ve un precio distinto según a qué instancia le toque: refresca y cambia. La comparación general entre caché local y distribuida está en [Caché distribuida vs. local](Cache-Distribuida-vs-Local.md); aquí interesa solo la solución a este problema concreto: **avisar a las demás por pub/sub**.

```csharp
// al arrancar: cada instancia escucha el canal y limpia su copia local
subscriber.Subscribe(RedisChannel.Literal("cache:invalidacion"), (_, mensaje) =>
    memoria.Remove(mensaje.ToString()));

// al invalidar: borra lo tuyo, borra lo compartido, y avisa
public async Task InvalidarAsync(string clave)
{
    memoria.Remove(clave);                    // mi copia local
    await db.KeyDeleteAsync(clave);           // la copia compartida en Redis
    await subscriber.PublishAsync(RedisChannel.Literal("cache:invalidacion"), clave);
}
```

Cada instancia recibe `producto:123` y lo borra de su memoria; la que publica también recibe su propio mensaje, y es inofensivo porque ya lo había borrado.

El detalle importante, y es el que se olvida: **el pub/sub de Redis es "a lo sumo una vez"**. No hay confirmación, no hay reintento, no hay historial. Una instancia que estuviera reiniciándose, con un corte de red de 200 ms o con el `ConnectionMultiplexer` reconectando, **no recibe el mensaje y no se enterará jamás**. Por eso el TTL sigue siendo obligatorio: es lo que acota el daño de un mensaje perdido a "unos minutos" en lugar de "hasta el próximo despliegue".

## Cache stampede: el número que lo hace tangible

Tienes `productos-destacados` con **2000 peticiones por segundo** y un recálculo que tarda **3 segundos** (varios `JOIN` y un agregado sobre `Pedidos`). La clave caduca. En esos 3 segundos que dura el recálculo llegan **6000 peticiones**, ninguna encuentra el dato, las 6000 detectan el *miss* y las 6000 lanzan la misma consulta de 3 segundos contra `TiendaDB` — justo en hora punta, que es exactamente cuando la base de datos menos margen tiene.

```csharp
// Sin protección: N peticiones simultáneas → N consultas idénticas al origen
if (!cache.TryGetValue(clave, out var valor))
{
    valor = await origenLento.CalcularAsync(); // se ejecuta 6000 veces a la vez
    cache.Set(clave, valor, ttl);
}
```

Y el fallo se realimenta: con 6000 consultas encima, el recálculo ya no tarda 3 segundos sino 20, así que en esos 20 segundos entran 40 000 peticiones más. La base de datos agota el pool de conexiones, todo empieza a dar *timeout*, y en el registro de errores no aparece nada sobre caché — aparece una base de datos saturada sin causa aparente. Un *miss* se convirtió en una caída.

## Mitigar el stampede, de menos a más

### 1. *Lock* local con doble comprobación

Que solo un hilo del proceso recalcule y los demás esperen su resultado.

```csharp
private static readonly SemaphoreSlim _lock = new(1, 1);

public async Task<Producto> ObtenerConProteccionAsync(int id)
{
    if (cache.TryGetValue($"producto:{id}", out Producto? producto))
        return producto!;

    await _lock.WaitAsync();
    try
    {
        if (cache.TryGetValue($"producto:{id}", out producto))
            return producto!; // otra petición ya lo recalculó mientras esperábamos

        producto = await repositorio.ObtenerPorIdAsync(id);
        cache.Set($"producto:{id}", producto, TimeSpan.FromMinutes(5));
        return producto;
    }
    finally
    {
        _lock.Release();
    }
}
```

La segunda comprobación **dentro** del *lock* es la clave: sin ella, los hilos que esperaban recalculan igual al entrar, uno detrás de otro, y no has arreglado nada. Pero tiene dos límites que hay que ver:

- **Solo protege dentro de un proceso.** Con cinco instancias, pasas de 6000 consultas a 5. Muchísimo mejor, pero no una.
- **Ese semáforo es `static` y único, así que serializa *todas* las claves.** Una petición de `producto:123` espera a que termine el recálculo de `productos-destacados`, con el que no tiene nada que ver. Lo correcto es **un semáforo por clave**, en un `ConcurrentDictionary<string, SemaphoreSlim>` — y con una política para retirar los que ya no se usan, o el diccionario se convierte en la fuga de memoria que sustituye al problema original.

### 2. *Lock* distribuido en Redis

Para que solo una instancia de las cinco recalcule, el *lock* tiene que vivir donde todas lo ven:

```redis
SET lock:productos-destacados <token-único> NX PX 5000
```

```csharp
var token = Guid.NewGuid().ToString();
bool loTengo = await db.StringSetAsync(
    "lock:productos-destacados", token,
    TimeSpan.FromSeconds(5),   // ← el PX
    When.NotExists);           // ← el NX

if (!loTengo)
{
    await Task.Delay(100);                       // otra instancia está recalculando
    return await ObtenerDeCacheOEsperarAsync();  // reintenta la lectura
}
```

- `NX` (*not exists*) es lo que hace que solo uno gane: es una comprobación y una escritura en una sola operación atómica.
- **`PX` no es opcional.** Es la caducidad del propio *lock*, y sin ella un proceso que muere sujetando la clave —un despliegue, un `OutOfMemory`, un contenedor que el orquestador mata— la deja puesta **para siempre**. Nadie puede volver a recalcular esa clave, y el síntoma es una entrada que se queda obsoleta sin explicación hasta que alguien encuentra un `lock:*` sin TTL en Redis. Un *lock* sin caducidad no es un *lock*, es un interbloqueo con retardo.
- El `token` único sirve para liberar **solo si sigues siendo el dueño** (comparar y borrar en un script Lua). Sin él, un proceso lento cuyo *lock* ya caducó borraría al liberar el *lock* de otro. Y si necesitas garantías serias sobre esto, usa una biblioteca como `RedLock.net` en vez de escribirlo a mano.

### 3. Recálculo anticipado

Las dos anteriores gestionan el *miss*; esta lo evita. Refrescar la entrada **antes** de que caduque de verdad, para que bajo carga nunca haya un hueco: se guardan dos tiempos, un TTL real de 10 minutos y un umbral de refresco a los 8, y cuando una lectura ve que quedan menos de 2 minutos devuelve el valor cacheado y **además** dispara el recálculo en segundo plano. Nadie espera y el dato nunca falta.

La versión elegante de esto es la **expiración anticipada probabilística** (*XFetch*): cada lectura tiene una probabilidad de refrescar que crece según se acerca la caducidad, ponderada por lo que costó el último recálculo. Reparte los refrescos en el tiempo sin ninguna coordinación entre instancias, y sin *locks*: por probabilidad, casi siempre refresca una sola petición.

### 4. `HybridCache` de .NET 9

Todo lo anterior es código que puedes escribir tú y mantener tú. Desde .NET 9 existe la abstracción oficial que ya lo trae hecho:

```csharp
builder.Services.AddHybridCache();

public async Task<Producto?> ObtenerProductoAsync(int id, CancellationToken ct) =>
    await cache.GetOrCreateAsync(
        $"producto:{id}",
        id,
        async (clave, token) => await repositorio.ObtenerPorIdAsync(clave, token),
        new HybridCacheEntryOptions {
            Expiration = TimeSpan.FromMinutes(10),           // en Redis
            LocalCacheExpiration = TimeSpan.FromMinutes(1) }, // en memoria del proceso
        tags: ["productos"],
        cancellationToken: ct);
```

Lo que te ahorra escribir: la protección de *stampede* (peticiones concurrentes a la misma clave se agrupan y solo una ejecuta la fábrica), los dos niveles local + distribuido con la coordinación entre ellos, la serialización, y la invalidación por etiquetas con `RemoveByTagAsync("productos")`. Sigue siendo tu responsabilidad decidir los TTL y llamar a la invalidación en los puntos correctos: `HybridCache` resuelve la mecánica, no el diseño.

## Jitter: no dejes que todo caduque a la misma vez

Al arrancar, una instancia calienta la caché con **5000 entradas** de producto, todas con el mismo TTL de 10 minutos. Diez minutos después caducan **las 5000 a la vez** y se produce un pico de 5000 recálculos simultáneos. Y como los recálculos también terminan casi a la vez, las nuevas entradas vuelven a compartir caducidad: el pico se repite cada diez minutos, indefinidamente. Es una base de datos que se cae en punto, con una regularidad que parece un `cron`. La solución cuesta una línea:

```csharp
private static TimeSpan ConJitter(TimeSpan ttl, int margenSegundos = 60)
{
    var desvio = Random.Shared.Next(-margenSegundos, margenSegundos + 1);
    return ttl + TimeSpan.FromSeconds(desvio);
}

cache.Set($"producto:{id}", producto, ConJitter(TimeSpan.FromMinutes(10)));
```

Con `10 min ± 60 s`, esas 5000 caducidades se reparten sobre una ventana de 120 segundos: unas 42 por segundo en lugar de 5000 de golpe. La misma cantidad de trabajo, repartida. Y como el desvío es aleatorio en cada refresco, los grupos no se vuelven a sincronizar. Regla práctica: **un margen del ±10 % del TTL** basta y no cuesta nada. El único caso donde el jitter molesta es cuando la caducidad debe ser exacta por contrato (un token que expira en un instante concreto); ahí desplaza siempre hacia abajo, nunca hacia arriba.

## Penetración de caché: lo que no existe nunca da *hit*

Toda la ficha ha hablado de datos que están y se quedan viejos. Este es el caso simétrico: datos que **no están porque no existen**. Un bot recorre `/api/productos/999991`, `999992`, `999993`… Ninguno existe en `Productos`. El código cache-aside típico consulta la caché (*miss*, obviamente), va a la base de datos, no encuentra nada y **no cachea nada** — porque no hay nada que cachear. Resultado: cada petición atraviesa la caché limpiamente y golpea `TiendaDB`. La caché está ahí, con un *hit ratio* magnífico en las métricas globales, y no protege absolutamente nada.

La mitigación es **cachear también la ausencia**, con un TTL corto:

```csharp
private static readonly object NoExiste = new();

public async Task<Producto?> ObtenerProductoAsync(int id)
{
    var clave = $"producto:{id}";
    if (cache.TryGetValue(clave, out object? entrada))
        return entrada == NoExiste ? null : (Producto?)entrada;

    var producto = await repositorio.ObtenerPorIdAsync(id);
    cache.Set(clave, (object?)producto ?? NoExiste,
        producto is null
            ? TimeSpan.FromSeconds(30)                  // ausencia: corto, por si se crea
            : ConJitter(TimeSpan.FromMinutes(10)));
    return producto;
}
```

El centinela `NoExiste` distingue "no está en caché" de "está cacheado como inexistente", que con un `null` a secas se confunde. Y el TTL corto es deliberado: si mañana se crea el producto 999991, no quieres 10 minutos de 404 falso.

Para casos extremos —un espacio de identificadores enorme y ataque sostenido— existe el **filtro de Bloom**: una estructura probabilística diminuta que responde "seguro que no existe" o "puede que exista", y descarta la mayoría de las peticiones inventadas sin tocar ni la caché ni la base de datos. Es raro necesitarlo: la caché negativa cubre el 99 % de los casos.

## Caché fría al arrancar

Una instancia nueva arranca con la caché local vacía. Si acabas de desplegar y las cinco instancias se reinician, **todo el tráfico va al origen a la vez**: es un *stampede* de todas las claves calientes simultáneamente, provocado por ti mismo, en el peor momento posible. Qué merece la pena precargar al arrancar (*cache warming*):

- ✅ **Lo pequeño, compartido y muy leído**: categorías, tipos de IVA, configuración, traducciones. Y **la lista corta de agregados caros** que ya sabes cuáles son (`productos-destacados` y compañía): una lista escrita a mano es más eficaz que cualquier heurística.
- ❌ **El catálogo entero**: 40 000 productos que retrasan el arranque, hinchan la memoria y de los que se leerán cien; además falsean el chequeo de salud y el orquestador puede matar el contenedor por tardar en responder. Y ❌ **los datos por usuario**: no sabes quién va a entrar, y son la mayor parte del volumen.

Dos medidas estructurales ayudan más que precargar: **usar caché distribuida para lo caro** (una instancia nueva encuentra Redis ya caliente y no arranca en frío en absoluto) y **desplegar de forma escalonada**, no reiniciando las cinco instancias a la vez.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| La ficha del producto muestra el precio nuevo, pero la portada sigue con el viejo | Se invalida `producto:123` y no `productos-destacados` ni `categoria:ropa:productos`; falta el mapa de claves derivadas o falta versionado |
| `TiendaDB` se satura en punto, a la misma hora todos los días, sin pico de tráfico | TTL idénticos sincronizados: miles de claves caducan a la vez. Falta *jitter* |
| Latencias altas y en aumento en claves que no tienen nada que ver entre sí | El `SemaphoreSlim` es `static` y único, así que serializa todas las claves. Hace falta uno por clave |
| Una lectura vuelve a cachear el valor viejo justo después de invalidar | Se invalidó **antes** de escribir en la base de datos: entre el `Remove` y el `COMMIT`, otra petición leyó el valor viejo y lo recacheó. Invalida siempre **después** |
| Una entrada se queda obsoleta indefinidamente y solo se arregla reiniciando | *Lock* distribuido sin `PX`: un proceso murió sujetándolo en el último despliegue y nadie puede recalcular esa clave |
| Refrescar la página alterna entre el dato nuevo y el viejo, y en local no pasaba | Cachés locales de varias instancias sin aviso por pub/sub: en local hay un proceso y en producción cinco, y el `Remove` no viaja |
| La memoria de Redis crece sin parar aunque el volumen de datos sea estable | Versionado de claves sin TTL: las generaciones viejas quedan huérfanas y nadie las borra |
| El *hit ratio* es alto y la base de datos sigue igual de cargada | Penetración: el tráfico va a identificadores inexistentes, que nunca dan *hit* |

## Buenas prácticas avanzadas

- **Invalida por evento, no por sondeo.** Engancha la invalidación al mismo punto de código —o al mismo evento de dominio— que produce el cambio, en lugar de un proceso que compruebe cada minuto si algo cambió. El sondeo es más caro y llega tarde por definición. Si la escritura y el cacheo viven en módulos distintos, publica el evento con un patrón *outbox* dentro de la misma transacción que la escritura: así no existe el estado "se escribió pero el aviso se perdió".
- **Invalida siempre después de escribir, y desconfía igualmente.** El orden correcto es `COMMIT` y luego `Remove`, porque al revés una lectura concurrente recachea el valor viejo dentro de la ventana. Pero incluso en el orden correcto queda una ventana estrecha —una lectura que empezó antes del `COMMIT` y escribe en caché después del `Remove`— y no se cierra sin transacciones distribuidas. La respuesta madura no es perseguirla: es el TTL, que la acota. Reconocer que la invalidación es *best effort* es lo que separa un diseño honesto de uno que se cree exacto.
- **Distingue invalidar de refrescar, y elige según la carga.** Invalidar (borrar) es simple pero deja un hueco de *miss* hasta la siguiente lectura, y ese hueco es precisamente donde nace el *stampede*. Refrescar (recalcular y reemplazar en el mismo paso) no deja hueco, a cambio de trabajo en el momento de la escritura. Para una clave con 2000 peticiones por segundo, refrescar; para una que se lee una vez cada minuto, invalidar y no complicarse.
- **Pon un TTL a todo, sin excepciones, incluidos los *locks* y los metadatos.** Cualquier clave sin caducidad es una fuga de memoria o un interbloqueo esperando su turno: la entrada que nadie invalidó, la generación huérfana del versionado, el `lock:*` de un proceso muerto, el conjunto `dep:*` de un pedido de hace un año. Repasa `TTL clave` en Redis sobre una muestra: si algo devuelve `-1`, tienes un problema latente.
- **Mide el retraso de invalidación, no solo el *hit ratio*.** El *hit ratio* te dice si la caché sirve; no te dice si miente. Instrumenta la antigüedad del dato servido (guarda la marca de tiempo del cálculo dentro del propio valor cacheado) y mira su percentil 99. Es la única métrica que detecta un punto de escritura que se olvidó de invalidar, porque el síntoma —un p99 que sube hasta tocar el TTL— aparece antes de que nadie de negocio se queje.
- **Prueba la invalidación con más de una instancia.** El error de invalidación más caro es el que no se reproduce en local, y en local hay una sola instancia. Levanta dos réplicas en tu entorno de pruebas y comprueba que un cambio se ve en las dos: es un test aburrido de escribir y es el que atrapa la clase entera de fallos de esta ficha.

## Recursos didácticos

- [Optimal Probabilistic Cache Stampede Prevention](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf) — el artículo original de *XFetch*, corto y legible. La fórmula ocupa una línea y explica por qué el refresco probabilístico funciona sin coordinación entre instancias.
- [Documentación de `HybridCache`](https://learn.microsoft.com/aspnet/core/performance/caching/hybrid) — la referencia de la API de .NET 9, con la configuración de los dos niveles, las etiquetas y la serialización. Empieza por aquí antes de escribir tu propia capa de invalidación.
- [Redis Keyspace Notifications](https://redis.io/docs/latest/develop/use-cases/keyspace-notifications/) — cómo Redis puede avisar por pub/sub de que una clave caducó o se borró, útil para construir la invalidación entre instancias sin publicar los mensajes tú.

---

*En resumen: cachear bien es fácil e invalidar bien es donde se demuestra si de verdad entiendes tu propio sistema — decide con cuántos segundos de dato viejo puedes vivir, invalida activamente lo que conozcas, y pon un TTL con jitter debajo de todo para que tus olvidos caduquen solos.*
