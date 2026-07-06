# Factory

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

**¿Qué es?** Factory es un patrón de diseño clásico (GoF) que encapsula la lógica de creación de objetos en un método o clase dedicada, en lugar de esparcir `new MiClase(...)` por todo el código. Existen dos variantes: *Factory Method* (un método que crea un tipo de objeto) y *Abstract Factory* (una interfaz que agrupa varios Factory Method relacionados).

---

## ¿Por qué existe?

Crear un objeto a veces implica validar datos, decidir entre varias implementaciones posibles o ensamblar piezas — no es un simple `new`. Si esa lógica se repite en cada sitio donde se necesita el objeto, cualquier cambio en las reglas de creación obliga a tocar muchos puntos del código, y es fácil que alguno se quede con la versión antigua.

En la [Capa de Dominio](../clean-architecture/Capa-de-Dominio.md) ya viste una versión mínima de esta idea: una factoría estática (`Pedido.Crear(...)`) que centraliza las reglas de creación de una entidad. Esta ficha generaliza el patrón GoF.

---

## ¿Cuándo y para qué se usa?

- **Factory Method**: cuando crear el objeto tiene reglas o validaciones propias que no deben repetirse ni saltarse (una entidad de dominio, una configuración compleja).
- **Abstract Factory**: cuando hay que elegir, en tiempo de ejecución, entre varias familias de implementaciones relacionadas — por ejemplo, exportar un informe a PDF o a Excel según lo que pida quien usa la aplicación.

---

## Lo mínimo que necesitas saber

**1. Factory Method: un método sustituye al `new` directo**

```csharp
public class Pedido
{
    private Pedido(int clienteId, IReadOnlyCollection<LineaPedido> lineas) { /* ... */ }

    public static Pedido Crear(int clienteId, IEnumerable<LineaPedido> lineas)
    {
        var listaLineas = lineas.ToList();
        if (listaLineas.Count == 0)
            throw new ArgumentException("Un pedido necesita al menos una línea.");

        return new Pedido(clienteId, listaLineas);
    }
}
```

**2. Abstract Factory: una interfaz agrupa varias fábricas relacionadas**

```csharp
public interface IExportador
{
    byte[] Exportar(Pedido pedido);
}

public interface IExportadorFactory
{
    IExportador CrearExportador(string formato); // "pdf", "excel"...
}
```

**3. El código cliente depende de la fábrica, no de las clases concretas**

```csharp
public class DescargarInforme(IExportadorFactory fabrica)
{
    public byte[] Ejecutar(Pedido pedido, string formato) =>
        fabrica.CrearExportador(formato).Exportar(pedido);
}
```

`DescargarInforme` nunca menciona `ExportadorPdf` ni `ExportadorExcel` directamente.

---

## Lo que NO hace

- **No es un contenedor de inyección de dependencias** — el contenedor de DI resuelve qué implementación usar de forma fija al arrancar la aplicación; una fábrica decide qué instancia crear con datos que solo se conocen en el momento de la llamada (el formato que pidió la usuaria en esa petición concreta).
- **No sustituye al constructor cuando no hay lógica de creación** — si el objeto no tiene reglas ni validaciones especiales, un `new` directo es más simple y una fábrica sería ceremonia sin beneficio.

---

## Buenas prácticas avanzadas

- **Usa una fábrica cuando el dato que decide el tipo solo se conoce en ejecución** — si la implementación a usar es fija al arrancar (por ejemplo, siempre la misma pasarela de pago configurada), eso lo resuelve el contenedor de DI; si depende de un dato que llega en cada llamada (el formato que pidió quien hace la petición), necesitas una fábrica invocada en ese momento.
- **No dejes abierto el constructor que la fábrica debería proteger** — si además de `Pedido.Crear(...)` el constructor sigue siendo público, cualquiera puede saltarse las validaciones llamándolo directamente. Márcalo `private` (o `internal` si el ORM lo necesita) para que la fábrica sea el único camino real.

---

*En resumen: una fábrica centraliza cómo se crea un objeto —con sus validaciones o su elección de implementación— para que el resto del código dependa de "qué necesito", no de "cómo se construye".*
