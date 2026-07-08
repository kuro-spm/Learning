# Atributos

## ¿Qué es?

Un atributo es una etiqueta declarativa que colocas sobre un elemento de tu código —una clase, un método, una propiedad o un parámetro— usando corchetes: `[Obsolete]`, `[Serializable]`, `[Route("api/products")]`. No ejecuta nada por sí mismo: es **metadatos**, información adicional adjunta al código que otra pieza (el compilador, el runtime o un framework) lee para decidir cómo comportarse.

## ¿Por qué existe?

Sirve para separar el *qué* del *cómo*. En vez de escribir la misma comprobación una y otra vez dentro de cada método ("¿quien llama tiene permiso?", "¿este campo viene relleno?"), declaras la intención una vez encima del elemento y dejas que otro código actúe en consecuencia.

Es el mismo mecanismo que ya conoces de las restricciones de una tabla SQL: cuando declaras una columna como `NOT NULL` o `UNIQUE`, no escribes un `if` en cada `INSERT` para comprobarlo; declaras la regla en la columna y el motor la aplica solo. Un atributo hace eso mismo, pero sobre tu código C#.

> Si vienes de Java, un atributo es lo que allí se llama *annotation* (`@Override`, `@Entity`). En TypeScript/Angular, el concepto equivalente son los *decorators* (`@Component`).

## ¿Cuándo y para qué se usa?

Aparecen en cuanto usas casi cualquier framework de .NET. El propio lenguaje y la librería base traen unos cuantos —`[Obsolete("usa el método nuevo")]` para marcar código a extinguir, `[Serializable]` para permitir serializar un tipo, `[Flags]` en enums de banderas—. Pero donde más se ven es en frameworks: mapear una propiedad a una columna con Entity Framework, validar un modelo con Data Annotations o, en ASP.NET Core, restringir el acceso a un endpoint y asociarlo a una URL. Cada vez que veas corchetes encima de una clase o un método, estás viendo un atributo dándole instrucciones a algún motor.

## Lo mínimo que necesitas saber

**1. Sintaxis: corchetes encima del elemento**

```csharp
[Obsolete]                          // sin argumentos
[Route("api/products")]             // con un argumento posicional
[Obsolete("Usa CreateOrderV2", error: true)]   // posicional + propiedad con nombre
public class OrderService { }
```

**2. Se colocan sobre distintos elementos, y se pueden apilar**

```csharp
[ApiController]
[Route("api/[controller]")]         // varios atributos, uno por línea
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetById([FromRoute] int id) { /* ... */ }
    //                           ↑ un atributo también puede ir sobre un parámetro
}
```

**3. Cada atributo vive en un namespace: necesitas su `using`**

Un atributo es una clase normal, así que hay que importar el namespace donde está definida. Si escribes `[Authorize]` sin el `using` correcto, el proyecto no compila:

```csharp
using System;                                 // aquí viven [Obsolete], [Serializable]...
using Microsoft.AspNetCore.Authorization;     // aquí viven [Authorize] y [AllowAnonymous]
using Microsoft.AspNetCore.Mvc;               // aquí viven [ApiController], [HttpGet], [FromBody]...
```

**4. El nombre real de la clase termina en `Attribute`**

`[Serializable]` es en realidad la clase `SerializableAttribute`. C# te deja omitir el sufijo `Attribute` al usarlo, por eso escribes solo `[Serializable]`.

**5. No hacen nada por sí solos: alguien tiene que leerlos**

Un atributo es metadatos inertes. Es otro código quien, mediante *reflection* (la capacidad de .NET de examinar tipos en ejecución), inspecciona el elemento, encuentra el atributo y actúa según lo que dice. Sin ese lector, el atributo no tiene ningún efecto.

```csharp
// Así lee un atributo quien lo procesa:
var info = typeof(OrderService).GetCustomAttribute<ObsoleteAttribute>();
if (info is not null)
    Console.WriteLine($"Tipo obsoleto: {info.Message}");
```

## Lo que NO hace

- **No ejecuta lógica por su cuenta** — solo aporta información; el comportamiento lo pone quien lee el atributo.
- **No cambia el tipo ni el valor** del elemento que decora — un método sigue siendo el mismo método.
- **No se valida "a fondo" en compilación** — el compilador comprueba que el atributo existe y que sus argumentos encajan, pero no que tenga sentido dónde lo pusiste; un atributo colocado donde nadie lo lee simplemente se ignora en silencio.

## Buenas prácticas avanzadas

- **Los argumentos deben ser constantes de compilación** — solo puedes pasar valores conocidos al compilar: literales, `const`, `typeof(...)`, enums y arrays de esos. Por eso no existe `[Authorize(Roles = miVariable)]`. Cuando necesitas una regla dinámica (leída de base de datos, por ejemplo), el atributo no basta: hay que resolver la lógica en ejecución (en ASP.NET Core, con *policies* o filtros).
- **Puedes crear los tuyos, y lo natural es que "hagan algo" vía un procesador** — heredar de `Attribute` te da la etiqueta, pero por sí sola es inerte. Cobra sentido cuando otra pieza la busca por reflection: un *action filter* en ASP.NET Core, un serializador, un validador propio. La etiqueta declara la intención; el procesador la ejecuta.
- **Controla dónde se puede poner con `[AttributeUsage]`** — al declarar un atributo propio, `[AttributeUsage(AttributeTargets.Method)]` impide que alguien lo cuele sobre una clase por error, y `AllowMultiple = true/false` decide si puede repetirse. Es la diferencia entre una etiqueta que se usa mal en silencio y una que avisa en compilación.
- **La *reflection* no es gratis: cachea si la usas en caliente** — inspeccionar atributos en cada iteración de un bucle o en cada petición es un cuello de botella clásico y silencioso. Los frameworks maduros leen los metadatos una vez al arrancar y reutilizan el resultado; si escribes tú el código que hace `GetCustomAttributes(...)`, cachéalo igual.

## Recursos didácticos divertidos

La documentación oficial mantiene el catálogo de atributos integrados navegable en <https://learn.microsoft.com/dotnet/api/system.attribute>: útil para descubrir cuántos trae .NET de serie más allá de los cuatro de siempre.

---

*En resumen: un atributo es una etiqueta de metadatos que pones sobre tu código para que otra pieza, al leerla por reflection, sepa cómo tratarlo — declaras la regla una vez y dejas de repetirla. Para ver los atributos concretos de un framework web, consulta [Primeros pasos con ASP.NET Core](../../../desarrollo-web/asp-net-core/README.md).*
