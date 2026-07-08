# Atributos personalizados

## ¿Qué es?

Un atributo personalizado es una etiqueta de metadatos que defines tú, en vez de usar una de las que traen el lenguaje o un framework. Se declara como una clase normal que hereda de `System.Attribute`, y a partir de ahí se coloca sobre tu código con corchetes igual que `[Obsolete]` o `[Authorize]`.

## ¿Por qué existe?

Los atributos integrados cubren lo habitual, pero a veces quieres marcar tu código con información propia y que otra parte de tu programa la lea: "esta acción hay que auditarla", "esta propiedad se exporta a CSV", "este método requiere un permiso a medida". Definir un atributo propio te da una etiqueta reutilizable para expresar esa intención una vez y aplicarla en muchos sitios, sin repetir la lógica.

> Recuerda la idea base ([ver Atributos](Atributos.md)): el atributo es solo la etiqueta. Quien le da sentido es el código que lo lee. Crear el atributo es la mitad del trabajo; la otra mitad es el *procesador* que lo interpreta.

## ¿Cuándo y para qué se usa?

Cuando ninguna etiqueta existente expresa lo que quieres marcar. Ejemplos genéricos: un atributo `[Auditable]` sobre los métodos cuyas llamadas hay que registrar; un `[ColumnaCsv("Precio")]` sobre las propiedades que van a un export; un `[SoloEnHorarioLaboral]` que un filtro de ASP.NET Core comprueba antes de dejar pasar la petición.

## Lo mínimo que necesitas saber

**1. Definirlo: hereda de `Attribute`, con el sufijo en el nombre**

Por convención, la clase termina en `Attribute`, y al usarla omites ese sufijo:

```csharp
public sealed class AuditableAttribute : Attribute
{
    public string Motivo { get; }
    public AuditableAttribute(string motivo) => Motivo = motivo;
}

[Auditable("cambio sensible de datos")]     // se usa sin el sufijo "Attribute"
public void DeleteProduct(int id) { /* ... */ }
```

**2. Argumentos: por constructor (obligatorios) o por propiedad (opcionales)**

Los parámetros del constructor son **posicionales** (obligatorios); las propiedades públicas se asignan con **nombre** y sirven para lo opcional:

```csharp
[Auditable("borrado", Severidad = 3)]        // "borrado" es posicional; Severidad, con nombre
```

**3. Los argumentos deben ser constantes de compilación (y esto tiene una razón de fondo)**

Aquí está la limitación que sorprende a casi todo el mundo: **no puedes pasar una variable ni el resultado de una llamada a un atributo**. Esto no compila:

```csharp
string motivo = CargarMotivoDesdeBd();
[Auditable(motivo)]                          // ERROR CS0182: no es una expresión constante
public void DeleteProduct(int id) { }
```

**El porqué:** un atributo no se ejecuta cuando escribes el código, sino que se **graba en los metadatos del ensamblado** (el `.dll`) en tiempo de compilación. El compilador guarda ahí el tipo del atributo y los valores que le pasaste, como una "foto fija". Más tarde, cuando alguien lee el atributo por reflection, el CLR **reconstruye la instancia** a partir de esa foto. Como los valores quedan congelados en el binario al compilar, tienen que ser conocidos y codificables en ese momento: no hay ningún instante de ejecución en el que evaluar `CargarMotivoDesdeBd()`.

Lo que sí puedes pasar:

- Literales (`"texto"`, `42`, `true`), y constantes `const`.
- `typeof(MiClase)` (un `System.Type`).
- Valores de un `enum`.
- Arrays unidimensionales de todo lo anterior (`new[] { "a", "b" }`).

Y ojo con estos, que **no** valen aunque parezcan inofensivos: `decimal`, `DateTime`, objetos creados con `new` (salvo arrays), o cualquier cosa calculada en ejecución.

**El corolario:** si tu regla depende de datos que solo se conocen en ejecución (un valor de base de datos, la hora actual, el importe de una factura), el atributo por sí solo no llega. El patrón es separar el *qué* del *cómo*: el atributo lleva un nombre o clave fija, y un **procesador** en ejecución resuelve la lógica. En ASP.NET Core eso son las *policies* y los filtros (ver [`[Authorize]`](../../../desarrollo-web/asp-net-core/Authorize.md) y [RBAC y Claims](../../../seguridad/autenticacion-y-autorizacion/RBAC-y-Claims.md)).

**4. `[AttributeUsage]`: controla dónde se puede poner**

Se pone sobre tu propia clase de atributo para limitar su uso; si alguien lo coloca donde no toca, falla en compilación:

```csharp
[AttributeUsage(AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
public sealed class AuditableAttribute : Attribute { /* ... */ }
```

- `AttributeTargets` — dónde es válido (`Method`, `Class`, `Property`, `Parameter`... o combinaciones con `|`).
- `AllowMultiple` — si se puede repetir sobre el mismo elemento.
- `Inherited` — si las clases derivadas lo heredan.

**5. Leerlo: reflection**

El atributo no hace nada hasta que alguien lo busca. Ese "alguien" usa reflection:

```csharp
var metodo = typeof(ProductService).GetMethod(nameof(ProductService.DeleteProduct));
var auditable = metodo.GetCustomAttribute<AuditableAttribute>();
if (auditable is not null)
    _auditor.Registrar(auditable.Motivo);      // aquí sí ocurre algo, en ejecución
```

## Lo que NO hace

- **No se ejecuta solo** — sin un procesador que lo lea por reflection, es metadato muerto.
- **No acepta datos dinámicos** — por la restricción de constantes de compilación; para eso está el patrón marcador + procesador.
- **No impone comportamiento** — describe una intención; hacerla cumplir es cosa del código que lo interpreta.

## Buenas prácticas avanzadas

- **Sella el atributo y declara siempre `[AttributeUsage]`** — un `sealed` evita jerarquías de atributos que casi nunca aportan, y un `[AttributeUsage]` explícito convierte un mal uso ("lo puse sobre una clase, pero solo valía para métodos") en un error de compilación en vez de en un fallo silencioso donde nadie lee la etiqueta.
- **Constructor para lo obligatorio, propiedades para lo opcional** — si un dato es imprescindible para que el atributo tenga sentido, exígelo en el constructor; así no se puede usar la etiqueta a medias. Deja las propiedades con nombre para los ajustes opcionales con valor por defecto razonable.
- **El atributo declara el *qué*; el procesador implementa el *cómo*** — resiste la tentación de meter lógica en el atributo. Un `[SoloEnHorarioLaboral]` no debería saber qué hora es: solo marca. Un filtro o handler que se ejecuta en cada petición es quien mira el reloj y decide. Esa separación es justo lo que hace ASP.NET Core con las *policies*, y es lo que te permite tener reglas dinámicas pese a la restricción de constantes.
- **Cachea la lectura por reflection si es en caliente** — `GetCustomAttribute` no es gratis. Un procesador que lo llama en cada iteración de un bucle o en cada request es un cuello de botella silencioso; lee los metadatos una vez (al arrancar, o la primera vez) y guarda el resultado en un diccionario.

## Recursos didácticos

En <https://sharplab.io> puedes pegar una clase con un atributo y ver el **IL y los metadatos** que genera el compilador: es la forma más directa de comprobar con tus ojos que los argumentos quedan "grabados a fuego" en el binario, y de entender por qué tienen que ser constantes.

---

*En resumen: un atributo personalizado es una clase que hereda de `Attribute` y que usas como etiqueta propia; sus argumentos son constantes de compilación porque se graban en los metadatos del ensamblado, así que para reglas dinámicas separas el atributo (el qué) de un procesador que lo lee por reflection en ejecución (el cómo).*
