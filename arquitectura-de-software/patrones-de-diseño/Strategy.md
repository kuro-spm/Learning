# Strategy

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

**¿Qué es?** Strategy es un patrón de diseño clásico (GoF) que extrae una familia de algoritmos intercambiables a clases separadas que comparten una misma interfaz, para que el código que los usa pueda cambiar de algoritmo sin cambiar de forma.

---

## ¿Por qué existe?

Cuando hay varias formas de hacer lo mismo —calcular el coste de envío, aplicar un descuento, ordenar una lista— la solución más directa es un gran `if`/`switch` que decide qué hacer según el caso. Funciona al principio, pero cada nueva variante obliga a tocar ese mismo bloque de código, que crece sin parar y mezcla toda la lógica de todas las variantes en un solo sitio.

Strategy separa cada variante en su propia clase, todas con la misma interfaz. El código que las usa recibe "una estrategia" sin saber cuál es exactamente, y añadir una variante nueva significa crear una clase más, no tocar las que ya existían.

---

## ¿Cuándo y para qué se usa?

Cálculo del coste de envío según el transportista, cálculo de un descuento según el tipo de promoción, criterios de ordenación distintos para un listado, reglas de validación que cambian según el plan contratado por el cliente. Cualquier sitio donde el "cómo" varía pero el "qué se necesita" es siempre lo mismo.

---

## Lo mínimo que necesitas saber

**1. Una interfaz común para todas las variantes**

```csharp
public interface ICalculadoraEnvio
{
    decimal Calcular(Pedido pedido);
}
```

**2. Cada estrategia es una clase independiente**

```csharp
public class EnvioEstandar : ICalculadoraEnvio
{
    public decimal Calcular(Pedido pedido) => 5.99m;
}

public class EnvioExpress : ICalculadoraEnvio
{
    public decimal Calcular(Pedido pedido) => 14.99m;
}
```

**3. El contexto recibe la estrategia, no decide con un switch interno**

```csharp
public class ProcesadorPedido(ICalculadoraEnvio calculadoraEnvio)
{
    public decimal CalcularTotal(Pedido pedido) => pedido.Subtotal + calculadoraEnvio.Calcular(pedido);
}
```

**4. La elección de qué estrategia usar se mueve a un único lugar**

En vez de repetir el `switch` allá donde se necesite la estrategia, se resuelve una sola vez —normalmente al registrar los servicios o en una pequeña fábrica— y desde ahí se inyecta ya decidida.

---

## Lo que NO hace

- **No es lo mismo que Template Method** — Template Method fija el esqueleto de un algoritmo y deja que las subclases varíen solo algunos pasos; Strategy sustituye el algoritmo completo por otro, sin esqueleto compartido.
- **No elimina la necesidad de decidir qué estrategia usar** — solo mueve esa decisión (el `if`/`switch`) a un único lugar, en vez de repetirla en cada punto donde se necesita el comportamiento.

---

## Buenas prácticas avanzadas

- **Resuelve la estrategia inyectando la colección completa** — en .NET puedes pedir `IEnumerable<ICalculadoraEnvio>` al contenedor y elegir la que aplica por una propiedad (`Codigo`), en vez de mantener un `switch` centralizado que hay que ampliar cada vez que se añade una estrategia nueva.
- **Cuidado con el estado compartido en estrategias `Singleton`** — si una implementación guarda campos mutables y el contenedor la registra como instancia única para toda la aplicación, todas las llamadas concurrentes comparten ese estado sin darse cuenta. Las estrategias deberían ser sin estado, o registrarse con un ciclo de vida (`Scoped`/`Transient`) acorde a lo que realmente necesitan.

---

*En resumen: Strategy convierte un `switch` de comportamientos en un conjunto de clases intercambiables detrás de una misma interfaz, para poder añadir variantes sin tocar el código que ya funciona.*
