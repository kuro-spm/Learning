# Caché Distribuida vs. Local

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

**¿Qué es?** Son dos formas distintas de ubicar la caché: **local** (o *in-process*) vive dentro de la memoria del propio proceso de la aplicación; **distribuida** vive en un servicio aparte (como Redis), compartido por todas las instancias de la aplicación.

---

## ¿Por qué existe?

Cuando una aplicación corre en una sola instancia, una caché local (un simple diccionario en memoria) es la opción más simple y rápida: no hay red de por medio. Pero en cuanto la aplicación se ejecuta en varias instancias a la vez (por escalado horizontal, detrás de un balanceador de carga), cada instancia tendría su propia copia de la caché, potencialmente desincronizada entre sí. La caché distribuida resuelve eso: todas las instancias leen y escriben el mismo almacén compartido.

---

## ¿Cuándo y para qué se usa?

- **Local**: una aplicación con una sola instancia, o datos que no necesitan estar sincronizados entre instancias (por ejemplo, una tabla de traducciones que se carga una vez al arrancar y apenas cambia).
- **Distribuida**: una API desplegada en varias instancias detrás de un balanceador, donde el usuario puede ser atendido por cualquiera de ellas en cada petición y necesita ver siempre el mismo dato cacheado (por ejemplo, su sesión o su carrito de la compra).

Un patrón habitual es combinar ambas en niveles: una caché local muy rápida como primera parada, y una distribuida como segunda, más lenta que la local pero compartida entre instancias.

---

## Lo mínimo que necesitas saber

**1. Local, con `IMemoryCache` en .NET**

```csharp
services.AddMemoryCache();

// en el servicio:
cache.Set("productos-destacados", productos, TimeSpan.FromMinutes(5));
```

**2. Distribuida, con `IDistributedCache` respaldada por Redis**

```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
});

// en el servicio, la interfaz es la misma independientemente del backend:
await cache.SetStringAsync("productos-destacados", json, opciones);
```

La ventaja de programar contra `IDistributedCache` es que cambiar el backend (de Redis a otro proveedor) no toca el código de negocio.

---

## Lo que NO hace

- **La local no resuelve la coherencia entre instancias** — cada instancia puede tener una versión distinta del mismo dato durante un rato.
- **La distribuida no es tan rápida como la local** — siempre hay una llamada de red de por medio, aunque sea a un servicio muy rápido como Redis.

---

*En resumen: la caché local es más rápida pero solo la ve una instancia; la distribuida es compartida pero paga el precio de la red — la eliges según si tu aplicación corre en una instancia o en varias.*
