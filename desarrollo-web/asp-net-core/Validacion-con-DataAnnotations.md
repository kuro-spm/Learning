# Atributos de validación (Data Annotations)

## ¿Qué es?

Son atributos que colocas sobre las propiedades de un modelo para declarar **qué reglas debe cumplir**: obligatorio, longitud máxima, rango de valores, formato de email... Viven en el namespace `System.ComponentModel.DataAnnotations` y los más habituales son `[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]` y `[RegularExpression]`.

## ¿Por qué existe?

Los datos que llegan de fuera nunca son de fiar: un formulario a medio rellenar, un cliente con un bug, alguien probando a romper tu API. Sin validación declarativa, tendrías que escribir en cada acción una tanda de `if (string.IsNullOrEmpty(...))`. Las Data Annotations declaran la regla **una vez, en el modelo**, y el framework la comprueba por ti.

> Es exactamente la idea de los atributos como metadatos ([ver Atributos](../../lenguajes/csharp-dotnet/caracteristicas-del-lenguaje/Atributos.md)): declaras la restricción y otro código (el validador de MVC) la aplica. La analogía con las restricciones `NOT NULL` / `CHECK` de una columna SQL es casi literal.

## ¿Cuándo y para qué se usa?

En los modelos de entrada de cualquier API o formulario: el alta de una cuenta (email válido, contraseña con longitud mínima), la creación de un producto (nombre obligatorio, precio positivo), un formulario de contacto. Se ponen sobre el DTO que recibe la petición.

## Lo mínimo que necesitas saber

**1. Se declaran sobre las propiedades del modelo**

```csharp
public class CreateUserRequest
{
    [Required]
    [StringLength(50, MinimumLength = 2)]
    public string Name { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Range(18, 120)]
    public int Age { get; set; }
}
```

**2. Las reglas más habituales**

- `[Required]` — no puede faltar ni ser vacío.
- `[StringLength(max)]` / `[MaxLength]` / `[MinLength]` — longitud de texto.
- `[Range(min, max)]` — rango numérico.
- `[EmailAddress]`, `[Phone]`, `[Url]` — formatos comunes.
- `[RegularExpression("...")]` — un patrón a medida.
- `[Compare("OtraPropiedad")]` — dos campos deben coincidir (contraseña y su confirmación).

**3. Con `[ApiController]`, la validación es automática**

No tienes que comprobar nada: si el modelo no cumple las reglas, la acción no se ejecuta y se devuelve un `400` con los errores (ver [`[ApiController]`](ApiController.md)). Sin él, comprobarías `ModelState.IsValid` a mano.

**4. Personaliza el mensaje de error**

```csharp
[Required(ErrorMessage = "El nombre es obligatorio")]
public string Name { get; set; }
```

**5. Aplican también a modelos anidados**

Si un modelo contiene otro objeto con sus propias anotaciones, la validación desciende y comprueba también las propiedades internas.

## Lo que NO hace

- **No validan reglas de negocio complejas** — "este email no está ya registrado" requiere consultar la base de datos; eso no cabe en un atributo (ver la nota sobre FluentValidation más abajo).
- **No sanean ni transforman** — no recortan espacios ni normalizan el texto; solo dicen si es válido o no.
- **No se ejecutan solas fuera de MVC** — en un controlador MVC el framework las dispara; si validas un objeto por tu cuenta, tienes que invocar el `Validator` explícitamente.

## Buenas prácticas avanzadas

- **`[Required]` y los tipos por valor engañan** — un `int` o un `bool` no pueden ser "nulos", así que `[Required]` sobre ellos no detecta que el cliente "no los envió" (llegan como `0` o `false`). Si necesitas distinguir "no enviado" de "enviado con valor 0", declara la propiedad como anulable (`int?`) y entonces `[Required]` sí exige que venga.
- **Para reglas que dependen de datos o de otras propiedades, usa FluentValidation** — las Data Annotations brillan en reglas simples y estáticas. Cuando la regla necesita la base de datos ("email único") o condiciones cruzadas ("si el país es España, el CIF es obligatorio"), un `IValidatableObject` o, mejor, [FluentValidation](../de-wpf-a-web/FluentValidation.md) mantienen la lógica testeable y fuera del modelo.
- **Valida en el borde, no en todas partes** — la validación de entrada protege la frontera de tu API; una vez el dato entró y es válido, no lo revalides en cada capa interna. Duplicar la validación en dominio y en el modelo de entrada suele acabar en reglas que se contradicen con el tiempo.

---

*En resumen: las Data Annotations declaran las reglas de un modelo de entrada como atributos sobre sus propiedades, y con `[ApiController]` el framework rechaza solo lo que no cumple — validación declarativa en el borde, como los `NOT NULL` de una tabla pero en tu C#.*
