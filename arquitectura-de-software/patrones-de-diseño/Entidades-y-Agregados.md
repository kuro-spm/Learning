# Entidades y Agregados

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Una **entidad** es un objeto del dominio que se distingue por su **identidad** (un id), no por sus atributos: aunque cambien sus datos, sigue siendo "la misma". Un **agregado** es un grupo de entidades y objetos de valor que se tratan como una sola unidad, con una entidad concreta —la **raíz del agregado**— como único punto de entrada desde fuera.

## ¿Por qué existe?

En la [Capa de Dominio](../clean-architecture/Capa-de-Dominio.md) ya viste que una entidad protege sus propias reglas (un `Pedido` no se puede confirmar vacío). Pero muchas reglas de negocio no dependen de un solo objeto, sino de **varios a la vez**: "la suma de las líneas de un pedido debe coincidir con su total", "no se puede añadir una línea a un pedido ya enviado". Si `Pedido` y `LineaPedido` se pudieran modificar por separado y sin control, nada garantizaría que esa regla conjunta se cumpla siempre.

El agregado resuelve esto marcando una frontera: fuera de ella, nadie toca directamente una `LineaPedido` suelta. Todo pasa por la raíz (`Pedido`), que es la única responsable de mantener el conjunto siempre en un estado válido.

> Piensa en un contrato con varios anexos. No puedes firmar o modificar un anexo por separado sin pasar por el contrato principal: el contrato es la puerta de entrada, y garantiza que el conjunto (contrato + anexos) siempre sea coherente.

## ¿Cuándo y para qué se usa?

Cada vez que un conjunto de objetos debe cambiar **junto**, de forma consistente: un `Pedido` con sus `LineaPedido`, un `Carrito` con sus `ItemCarrito`, una `Reserva` con sus `Ocupante`. La raíz del agregado (`Pedido`, `Carrito`, `Reserva`) es la entidad por la que se accede siempre desde fuera; las entidades internas no se exponen ni se modifican directamente.

## Lo mínimo que necesitas saber

**1. Identidad frente a valor**

Una entidad se compara por su id, no por sus atributos: un `Cliente` con id `42` sigue siendo el mismo cliente aunque le cambies el nombre. Esto la distingue de un [objeto de valor](Value-Objects.md), que se compara por contenido.

```csharp
public class Cliente
{
    public int Id { get; }
    public string Nombre { get; private set; }
    // Dos Cliente con el mismo Id son "el mismo", aunque el Nombre difiera en el tiempo
}
```

**2. La raíz del agregado es el único punto de entrada**

Las entidades internas del agregado no se exponen para modificarse desde fuera; solo la raíz ofrece métodos para operar sobre el conjunto.

```csharp
public class Pedido
{
    private readonly List<LineaPedido> _lineas = new();
    public IReadOnlyCollection<LineaPedido> Lineas => _lineas.AsReadOnly();

    public void AnadirLinea(int productoId, int cantidad, decimal precioUnitario)
    {
        if (Confirmado)
            throw new InvalidOperationException("No se puede modificar un pedido ya confirmado.");
        _lineas.Add(new LineaPedido(productoId, cantidad, precioUnitario));
    }

    public bool Confirmado { get; private set; }
}
```

Nadie hace `pedido.Lineas.Add(...)` directamente: la colección se expone solo de lectura, y añadir una línea pasa siempre por `AnadirLinea`, que puede vigilar sus reglas.

**3. El agregado es el límite de la transacción**

Cuando guardas un cambio, guardas el agregado **completo**, no una entidad interna suelta. Esto simplifica mucho las cosas: o el pedido con todas sus líneas queda guardado de forma consistente, o no se guarda nada.

**4. Entre agregados, referencias por id, no por objeto**

Un `Pedido` no debería mantener una referencia directa en memoria a todo el objeto `Cliente`; le basta con guardar `ClienteId`. Así cada agregado se puede cargar y guardar de forma independiente, sin arrastrar grafos de objetos enormes.

```csharp
public class Pedido
{
    public int ClienteId { get; }   // referencia por id, no el objeto Cliente completo
}
```

## Lo que NO hace

- **No todo objeto necesita ser una entidad** — si algo no tiene identidad ni ciclo de vida propio, es un [objeto de valor](Value-Objects.md), más simple de razonar.
- **No se cargan ni se guardan trozos sueltos de un agregado** — se trabaja con el agregado entero a través de su raíz.
- **Un agregado no tiene por qué coincidir con una tabla de base de datos** — puede mapearse a varias tablas relacionadas (`Pedidos` + `LineasPedido`), pero eso es un detalle de persistencia, no del modelo.

## Buenas prácticas avanzadas

- **Mantén los agregados pequeños** — un agregado con demasiadas entidades anidadas es lento de cargar completo y aumenta los bloqueos de concurrencia (dos personas editando "el mismo" agregado gigante aunque toquen partes distintas). Si dudas del tamaño, referencia el otro objeto por id en vez de anidarlo.
- **Una transacción, un agregado** — un caso de uso debería modificar un único agregado por transacción. Si una operación parece necesitar cambiar dos agregados a la vez de forma atómica, es una señal de que el límite está mal puesto o de que la coordinación debería resolverse con [eventos de dominio](Domain-Events.md) y consistencia eventual, no ampliando el agregado para que abarque a ambos.
- **Protege los invariantes solo desde la raíz** — si una entidad interna del agregado expone setters públicos o métodos que cualquiera puede llamar sin pasar por la raíz, el agregado deja de garantizar nada: las reglas conjuntas se pueden saltar por la puerta de atrás.
- **Usa concurrencia optimista en la raíz** — un campo de versión (`RowVersion`/`ConcurrencyToken`) en la raíz del agregado detecta cuando dos procesos intentan guardar cambios sobre la misma versión a la vez, evitando que uno pise silenciosamente el trabajo del otro.

---

*En resumen: una entidad se reconoce por su identidad, no por sus datos; un agregado agrupa varias bajo una raíz que es la única puerta de entrada, garantizando que las reglas que dependen de varios objetos a la vez nunca se rompan.*
