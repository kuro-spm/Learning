# Tipos de retorno de una acción (IActionResult)

## ¿Qué es?

Es lo que un método de acción de un controlador **devuelve** para que el framework construya la respuesta HTTP. No escribes la respuesta a mano: devuelves un objeto que la describe (los datos, el código de estado, las cabeceras) y ASP.NET Core lo traduce a una respuesta real. Los tres tipos de retorno que verás son **`IActionResult`**, **`ActionResult<T>`** y **devolver el dato directamente**.

## ¿Por qué existe?

Una respuesta HTTP no es solo unos datos: es un **código de estado** (`200 OK`, `404 Not Found`, `201 Created`...), a veces unas cabeceras y a veces un cuerpo. Pero un método de C# normal solo puede devolver "una cosa" de un tipo fijo. ¿Cómo haces que un mismo método devuelva un `200` con un producto en unos casos y un `404` vacío en otros?

Para eso existe `IActionResult`: es una **interfaz** (un contrato) que representa "algo que sabe convertirse en una respuesta HTTP". Como es una interfaz, tu método puede devolver cualquier implementación concreta según lo que ocurra, y todas encajan en la misma firma.

> Si te ancla a C#: `IActionResult` es un contrato como cualquier `interface`. Métodos como `Ok(...)` o `NotFound()` son fábricas que devuelven distintas clases que implementan ese contrato (`OkObjectResult`, `NotFoundResult`...). El método declara que devuelve el contrato; en tiempo de ejecución devuelve la implementación que toque.

## ¿Cuándo y para qué se usa?

En **toda** acción de un controlador. Un `GET` de un producto que puede existir o no devuelve `200` con el dato o `404` si no está. Un `POST` que crea un recurso devuelve `201`. Un `DELETE` correcto devuelve `204`. El tipo de retorno es lo que te permite expresar cada uno de esos desenlaces desde el mismo método.

## Lo mínimo que necesitas saber

**1. `IActionResult`: el contrato de "un resultado de acción"**

Devuelves algo que sabe convertirse en respuesta HTTP, no los datos crudos. Los métodos *helper* de `ControllerBase` producen esas implementaciones:

```csharp
[HttpGet("{id:int}")]
public IActionResult GetById(int id)
{
    var product = _repo.Find(id);
    if (product is null) return NotFound();   // 404, sin cuerpo
    return Ok(product);                        // 200 + product serializado a JSON
}
```

**2. Los *helpers* cubren cada tipo de respuesta**

Cada uno empaqueta un código de estado (y, si procede, un cuerpo):

- `Ok(obj)` → `200` con cuerpo
- `NotFound()` → `404`
- `BadRequest(error)` → `400`
- `NoContent()` → `204` (correcto pero sin nada que devolver, típico de un `DELETE`)
- `CreatedAtRoute(...)` / `CreatedAtAction(...)` → `201` + cabecera `Location` apuntando al recurso nuevo
- `Unauthorized()` / `Forbid()` → `401` / `403`

**3. `ActionResult<T>`: tipado y flexibilidad a la vez**

Permite devolver **o** el dato `T` directamente **o** un *helper*. Así el framework conoce el tipo de datos (útil para generar la documentación OpenAPI/Swagger) sin perder la libertad de devolver un `404`:

```csharp
[HttpGet("{id:int}")]
public ActionResult<Product> GetById(int id)
{
    var product = _repo.Find(id);
    if (product is null) return NotFound();
    return product;   // se envuelve solo en un 200 + JSON
}
```

**4. Devolver el dato directamente**

Si la acción no tiene desenlaces alternativos, puedes declarar el tipo concreto: ASP.NET lo serializa a JSON con un `200`.

```csharp
[HttpGet]
public IEnumerable<Product> GetAll() => _repo.GetAll();   // 200 + array JSON
```

**5. La serialización a JSON es automática**

El objeto que pasas a `Ok(product)` (o que devuelves directamente) lo convierte a JSON el formateador de salida. No llamas a ningún serializador tú.

**6. En métodos asíncronos, se envuelve en `Task<>`**

```csharp
public async Task<ActionResult<Product>> GetById(int id)
{
    var product = await _repo.FindAsync(id);
    return product is null ? NotFound() : product;
}
```

## Lo que NO hace

- **No construyes la respuesta a mano** — no tocas `Response.StatusCode` ni escribes el JSON; devuelves el resultado y el framework lo materializa.
- **`IActionResult` "pelado" no lleva el tipo de los datos** — por eso Swagger no sabe qué devuelve una acción `IActionResult` salvo que uses `ActionResult<T>` o lo anotes con `[ProducesResponseType]`.
- **El *helper* no elige el código por ti** — eres tú quien decide `Ok` vs `NotFound`; el *helper* solo empaqueta el código y el cuerpo que le indicas.
- **No valida la entrada** — de eso se encargan las [Data Annotations](Validacion-con-DataAnnotations.md) y [`[ApiController]`](ApiController.md).

## Buenas prácticas avanzadas

- **Prefiere `ActionResult<T>` a `IActionResult` en APIs** — combina el tipado (Swagger/OpenAPI genera el esquema del cuerpo automáticamente) con la libertad de devolver `404` o `400`. Con `IActionResult` a secas el contrato queda opaco y tienes que documentar el tipo a mano con `[ProducesResponseType(typeof(Product), 200)]`.
- **Devuelve el código de estado correcto, no siempre `200`** — crear → `201` con `CreatedAtRoute` (y su cabecera `Location`), borrar → `204`, no encontrado → `404`. Responder `200` con un `{ "error": ... }` en el cuerpo para todo es el error clásico que vuelve la API impredecible: el cliente no puede fiarse del código de estado.
- **No devuelvas entidades del dominio directamente, expón DTOs** — serializar la entidad de EF Core arrastra sus propiedades de navegación (ciclos infinitos, *over-fetching*, campos sensibles como un hash de contraseña). Mapea a un DTO y devuelve `Ok(dto)`.
- **`return null;` en un `ActionResult<T>` no da `404`** — da `204 No Content`, que casi nunca es lo que quieres. Si el recurso no existe, sé explícito con `return NotFound();`. Es un despiste sutil que pasa la revisión de código con facilidad.

## Recursos didácticos

La guía oficial «Controller action return types in ASP.NET Core web API» compara los tres enfoques con ejemplos: <https://learn.microsoft.com/aspnet/core/web-api/action-return-types>. Y como todo esto gira en torno a elegir el código de estado adecuado, <https://http.cat/> te los explica con gatos: útil (y divertido) para no confundir cuándo toca un `204`, un `201` o un `404`.

---

*En resumen: una acción no escribe la respuesta HTTP, devuelve un objeto que la describe — `IActionResult` para cualquier resultado, `ActionResult<T>` cuando quieres además tipado, o el dato a secas para un `200` simple; el framework se encarga del código de estado y de serializar a JSON.*
