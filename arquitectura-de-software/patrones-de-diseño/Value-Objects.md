# Value Objects

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Un objeto de valor (*value object*) es un objeto que se define **por sus atributos, no por una identidad**. Dos objetos de valor con los mismos datos son intercambiables y se consideran iguales, a diferencia de una entidad, que aunque cambie sus datos sigue siendo "la misma".

## ¿Por qué existe?

Es habitual modelar conceptos del negocio con tipos primitivos: un email como `string`, un precio como `decimal`, unas coordenadas como dos `double` sueltos. El problema es que un `string` acepta cualquier texto (incluido uno que no es un email válido) y un `decimal` no sabe si son euros o dólares. Esta "obsesión por los primitivos" traslada la validación y el significado del dato a cada sitio donde se usa, y es fácil de mezclar: nada impide pasar un `decimal` que representa un descuento donde se esperaba un precio.

Un objeto de valor envuelve esos primitivos en un tipo con nombre propio, que valida sus propias reglas al crearse y no se puede modificar después.

> Piensa en dos billetes de 10 €: no importa cuál de los dos tengas en la mano, valen lo mismo y son intercambiables. Nadie pregunta "¿cuál es la identidad de este billete?" — solo importa su valor. Eso es exactamente un objeto de valor.

## ¿Cuándo y para qué se usa?

Aparece en cualquier concepto del dominio que se defina solo por sus datos: un importe de dinero (`Dinero { Cantidad, Moneda }`), un rango de fechas, una dirección postal, un email, unas coordenadas geográficas. En una tienda online, el precio de un producto o la dirección de envío de un pedido son buenos candidatos a objeto de valor.

En la [Capa de Dominio](../clean-architecture/Capa-de-Dominio.md) ya viste un ejemplo mínimo con `Email`; esta ficha profundiza en el patrón.

## Lo mínimo que necesitas saber

**1. Igualdad por valor, no por referencia**

Dos objetos de valor son iguales si todos sus atributos coinciden, sin importar si son instancias distintas en memoria. En C#, los `record` dan esta igualdad gratis.

```csharp
public record Dinero(decimal Cantidad, string Moneda);

var a = new Dinero(10m, "EUR");
var b = new Dinero(10m, "EUR");
Console.WriteLine(a == b); // True — misma "forma", no la misma referencia
```

**2. Inmutabilidad**

Un objeto de valor no se modifica una vez creado: cualquier "cambio" produce una instancia nueva. Esto evita bugs de estado compartido (dos partes del código con la misma referencia, una la muta y la otra se entera sin querer).

```csharp
public record Dinero(decimal Cantidad, string Moneda)
{
    public Dinero Sumar(Dinero otro)
    {
        if (Moneda != otro.Moneda)
            throw new InvalidOperationException("No se pueden sumar monedas distintas.");
        return new Dinero(Cantidad + otro.Cantidad, Moneda);
    }
}
```

**3. Autovalidación en el constructor**

El objeto de valor no permite construirse en un estado inválido: la validación va en el constructor, no en un método aparte que alguien podría olvidar llamar.

```csharp
public record Email
{
    public string Valor { get; }

    public Email(string valor)
    {
        if (string.IsNullOrWhiteSpace(valor) || !valor.Contains('@'))
            throw new ArgumentException("Email no válido.");
        Valor = valor;
    }
}
```

**4. Composición de objetos de valor**

Los objetos de valor se pueden combinar entre sí para formar conceptos más ricos, como una `Direccion` compuesta por varios campos con sus propias reglas.

```csharp
public record Direccion(string Calle, string Ciudad, string CodigoPostal);
```

## Lo que NO hace

- **No tiene identidad propia** — no existe un "id de objeto de valor"; si necesitas rastrear algo a lo largo del tiempo, es una entidad, no un objeto de valor.
- **No suele tener su propia tabla en base de datos** — normalmente se guarda embebido dentro de la entidad que lo contiene (por ejemplo, como *owned type* en Entity Framework Core).
- **No reemplaza la validación de la interfaz de usuario** — protege el dominio de datos inválidos, pero un formulario bien diseñado sigue siendo necesario para dar feedback inmediato a quien lo rellena.

## Buenas prácticas avanzadas

- **Dale comportamiento, no lo dejes como una bolsa de datos con nombre** — un objeto de valor que solo expone propiedades y ninguna operación (`Sumar`, `AplicarDescuento`, `EstaEnRango`) desaprovecha el patrón: la lógica que opera sobre esos datos vuelve a dispersarse por fuera, exactamente el problema que el objeto de valor debía resolver.
- **Cuidado con la igualdad estructural en colecciones** — si un objeto de valor contiene una `List<T>` o un array, la igualdad automática de los `record` de C# no compara el contenido de esas colecciones elemento a elemento salvo que uses tipos inmutables pensados para ello (como `ImmutableArray<T>`); revisa esto antes de asumir que dos instancias "iguales" lo son de verdad.
- **En el ORM, persiste como tipo embebido, no como tabla propia** — mapear un objeto de valor como una entidad más con su propio `Id` en EF Core reintroduce la identidad que el patrón quería evitar; usa `OwnsOne`/*owned types* para que viva como columnas de la tabla dueña.
- **No conviertas un objeto de valor en agregado por acumulación de campos** — si un `Dinero` empieza a necesitar un histórico de cambios o un ciclo de vida propio, ha dejado de ser un objeto de valor: probablemente necesitas una entidad (ver [Entidades y Agregados](Entidades-y-Agregados.md)).

---

*En resumen: un objeto de valor es un dato inmutable definido por su contenido, no por quién es — dos objetos de valor iguales son literalmente intercambiables.*
