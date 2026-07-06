# Decorator

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

**¿Qué es?** Decorator es un patrón de diseño clásico (GoF) que añade comportamiento a un objeto envolviéndolo con otro objeto que implementa la misma interfaz, sin modificar la clase original ni recurrir a la herencia.

---

## ¿Por qué existe?

Añadir responsabilidades extra a una clase —registrar logs, cachear resultados, reintentar en caso de fallo— modificando directamente su código la mezcla con preocupaciones que no son su responsabilidad principal. Y crear una subclase por cada combinación posible de estas responsabilidades (una clase con caché, otra con logging, otra con ambas...) se dispara en cuanto aparece una tercera opción.

Decorator resuelve esto envolviendo: cada decorador implementa la misma interfaz que el objeto original, añade su parte y delega el resto al objeto que envuelve. Se pueden apilar tantos decoradores como haga falta, cada uno aportando una sola responsabilidad. En [Casos de Uso](../clean-architecture/Casos-de-Uso.md) ya se mencionaba esta idea para centralizar preocupaciones transversales; esta ficha desarrolla el patrón general.

---

## ¿Cuándo y para qué se usa?

Añadir caché a un repositorio sin tocar su implementación, añadir *logging* alrededor de un servicio, añadir reintentos a un cliente HTTP que llama a una API externa — siempre que quieras sumar una capacidad sin modificar (ni heredar de) la pieza original.

---

## Lo mínimo que necesitas saber

**1. Una interfaz común**

```csharp
public interface IRepositorioProductos
{
    Task<Producto?> ObtenerPorId(int id);
}
```

**2. La implementación original y un decorador que añade caché**

```csharp
public class RepositorioProductosCacheado(IRepositorioProductos interno, IMemoryCache cache)
    : IRepositorioProductos
{
    public async Task<Producto?> ObtenerPorId(int id)
    {
        if (cache.TryGetValue(id, out Producto? producto))
            return producto;

        producto = await interno.ObtenerPorId(id);
        cache.Set(id, producto, TimeSpan.FromMinutes(5));
        return producto;
    }
}
```

**3. Los decoradores se apilan, todos con la misma interfaz**

Caché, luego *logging*, luego reintentos: cada uno envuelve al anterior, y quien los usa no distingue si habla con el original o con toda la cadena.

```csharp
IRepositorioProductos repositorio =
    new RepositorioProductosConLog(
        new RepositorioProductosCacheado(
            new RepositorioProductosSql(contexto)));
```

**4. El consumidor no sabe cuántas capas hay**

Sea cual sea la combinación, el código que usa `IRepositorioProductos` sigue llamando a `ObtenerPorId` exactamente igual.

---

## Lo que NO hace

- **No cambia la interfaz pública** — un decorador siempre implementa el mismo contrato que decora; si necesitas una operación nueva, no es un decorador.
- **No es composición cualquiera** — la clave es que decorador y decorado comparten interfaz, de modo que el decorador puede sustituir al original en cualquier sitio donde se use.
- **No sustituye a un pipeline o middleware ya existente en el framework** — para peticiones HTTP en ASP.NET, por ejemplo, ya existe el mecanismo de *middleware*; reinventar decoradores a mano ahí sería duplicar una pieza que el framework ya resuelve.

---

## Buenas prácticas avanzadas

- **Registra la cadena en el contenedor de DI, no la construyas a mano en cada sitio** — librerías como Scrutor permiten decorar un registro existente (`services.Decorate<IRepositorioProductos, RepositorioProductosCacheado>()`), evitando que "quién envuelve a quién" quede disperso y duplicado por el código.
- **El orden de los decoradores importa, y no es obvio leyendo cada uno por separado** — cachear antes o después de registrar el log cambia si el log ve los aciertos de caché o no. Documenta el orden de la cadena en el punto donde se registra, porque no se deduce mirando cada decorador aislado.

---

*En resumen: Decorator añade capas de comportamiento envolviendo un objeto con otros que comparten su interfaz, permitiendo combinar responsabilidades (caché, logging, reintentos...) sin tocar la implementación original ni multiplicar subclases.*
