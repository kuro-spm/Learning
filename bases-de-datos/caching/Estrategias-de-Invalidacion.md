# Estrategias de Invalidación de Caché

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

La invalidación de caché es el conjunto de técnicas para decidir cuándo un dato cacheado deja de ser válido y debe eliminarse o refrescarse, en vez de seguir sirviéndose desactualizado.

## ¿Por qué existe?

Cachear es fácil; el problema aparece cuando el dato de origen cambia y la copia en caché no se entera. Hay una frase muy citada en programación: "solo hay dos cosas difíciles en informática: invalidación de caché y poner nombres a las cosas". Si el dato en caché no se invalida a tiempo, la aplicación muestra información obsoleta (o incluso incorrecta) sin que nada avise del error.

## ¿Cuándo y para qué se usa?

Cualquier sistema con caché necesita una estrategia de invalidación, aunque sea implícita. Aparece, por ejemplo, cuando el precio de un producto cambia y la caché de la ficha de producto debe reflejarlo cuanto antes, o cuando se cancela un pedido y cualquier contador cacheado de "pedidos activos" debe actualizarse.

## Lo mínimo que necesitas saber

**1. TTL: dejar que caduque sola**

La forma más simple: la entrada tiene una fecha de caducidad y, pasado ese tiempo, se considera inválida sin que nadie tenga que borrarla explícitamente.

```csharp
cache.Set($"producto:{id}", producto, TimeSpan.FromMinutes(5));
```

Ventaja: no requiere coordinar quién invalida qué. Desventaja: durante esa ventana de tiempo, un cambio real en el origen no se refleja en la caché.

**2. Invalidación activa: borrar en el momento del cambio**

Cuando el código que modifica el dato también borra (o actualiza) su entrada de caché correspondiente.

```csharp
public async Task ActualizarPrecioAsync(int id, decimal nuevoPrecio)
{
    await repositorio.ActualizarPrecioAsync(id, nuevoPrecio);
    cache.Remove($"producto:{id}"); // se recalculará en la próxima lectura
}
```

Ventaja: la caché nunca queda desactualizada más allá de un instante. Desventaja: exige disciplina — cada punto del código que modifica el dato debe acordarse de invalidar la caché, y es fácil olvidar alguno.

**3. Combinar ambas: TTL como red de seguridad**

El patrón más robusto en la práctica: invalidar activamente en cada escritura conocida, y además poner un TTL razonable como salvaguarda por si algún punto de escritura se olvidó de invalidar.

**4. El problema del *cache stampede***

Cuando una entrada muy solicitada caduca y llegan muchas peticiones simultáneas, todas detectan el *miss* a la vez y recalculan el mismo dato en paralelo, multiplicando la carga sobre el origen justo cuando menos conviene.

```csharp
// Sin protección: N peticiones simultáneas → N consultas idénticas al origen
if (!cache.TryGetValue(clave, out var valor))
{
    valor = await origenLento.CalcularAsync(); // se ejecuta N veces a la vez
    cache.Set(clave, valor, ttl);
}
```

**5. Mitigar el *stampede*: bloqueo o recálculo anticipado**

Una solución común es que solo una petición recalcule el valor mientras las demás esperan (con un `SemaphoreSlim` o un *lock* distribuido), o refrescar la entrada un poco antes de que caduque de verdad para que nunca llegue a haber un *miss* real bajo carga.

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

## Lo que NO hace

- **No elimina la necesidad de decidir un TTL** — incluso con invalidación activa, un TTL de seguridad sigue siendo recomendable.
- **No es gratis** — invalidar activamente añade acoplamiento: el código que escribe el dato necesita saber qué claves de caché dependen de él.
- **No resuelve la coherencia entre varios cachés distintos** — si el mismo dato vive cacheado en dos sitios distintos, invalidar uno no invalida el otro automáticamente.

## Buenas prácticas avanzadas

- **Invalida por evento, no por sondeo** — engancha la invalidación al mismo punto del código (o al mismo evento de dominio) que produce el cambio, en vez de tener un proceso que compruebe periódicamente si algo cambió; es más preciso y más barato.
- **Usa TTL con "jitter" (variación aleatoria)** — si miles de claves se guardaron a la vez con el mismo TTL exacto, todas caducan a la vez y generan un pico de recálculo simultáneo. Añadir unos segundos aleatorios al TTL de cada entrada dispersa esos picos.
- **Diferencia invalidar de refrescar** — invalidar (borrar) es más simple, pero deja un hueco de *miss* hasta la siguiente lectura; refrescar (recalcular y reemplazar en el mismo paso) evita ese hueco a cambio de más trabajo en el momento de la escritura.
- **Ten en cuenta la invalidación en cascada** — si un dato agregado (como "total de pedidos de un cliente") depende de varias entidades, cambiar cualquiera de ellas debe invalidar también el agregado; es un punto habitual donde se cuelan datos obsoletos.

---

*En resumen: cachear bien es fácil; invalidar bien es donde se demuestra si de verdad entiendes tu propio sistema — decide con qué margen de datos obsoletos puedes vivir y diseña la invalidación en consecuencia.*
