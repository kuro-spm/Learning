# [ApiController]

## ¿Qué es?

`[ApiController]` es un atributo que se pone sobre la clase de un controlador para decirle a ASP.NET Core: "esto es una API REST". Al ponerlo, el framework activa un paquete de comportamientos pensados para construir APIs que devuelven datos (JSON), ahorrándote código repetitivo.

## ¿Por qué existe?

Sin `[ApiController]`, en cada acción de tu API tendrías que repetir siempre lo mismo: comprobar a mano si los datos de entrada son válidos y devolver un `400` si no, indicar explícitamente de dónde sale cada parámetro, dar formato a los errores... El atributo asume que estás haciendo una API y aplica todas esas convenciones por ti.

> Si conoces los atributos como concepto general del lenguaje ([ver ficha de Atributos](../../lenguajes/csharp-dotnet/caracteristicas-del-lenguaje/Atributos.md)), este es un caso claro: una etiqueta inerte que el framework MVC lee al arrancar para cambiar cómo trata el controlador.

## ¿Cuándo y para qué se usa?

En cualquier controlador que sirva una API: los endpoints de una tienda online que devuelven el catálogo en JSON, la API que consume una app móvil, un servicio interno. Se pone junto con el atributo de ruta, sobre una clase que hereda de `ControllerBase`.

## Lo mínimo que necesitas saber

**1. Se declara sobre la clase, junto a la ruta**

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // ...
}
```

**2. Valida el modelo automáticamente y devuelve 400**

Si los datos de entrada no cumplen sus reglas de validación (ver [Validación con Data Annotations](Validacion-con-DataAnnotations.md)), la acción **ni siquiera se ejecuta**: el framework responde un `400 Bad Request` con el detalle de los errores. Sin `[ApiController]`, tendrías que escribir `if (!ModelState.IsValid) return BadRequest(ModelState);` en cada acción.

```csharp
[HttpPost]
public IActionResult Create(CreateProductRequest request)
{
    // Si 'request' no es válido, aquí ya no se llega: se devolvió 400 solo
    return Ok();
}
```

**3. Infiere de dónde sale cada parámetro**

No necesitas anotar los parámetros: el framework deduce que los tipos complejos vienen del cuerpo (`[FromBody]`), y los simples de la ruta o la query (ver [Model binding](Model-Binding.md)). Menos ruido en la firma.

**4. Requiere routing por atributos**

Con `[ApiController]` es obligatorio definir la ruta con atributos (`[Route]`, `[HttpGet("...")]`). No funciona con el enrutado convencional por convención.

**5. Da formato estándar a los errores (ProblemDetails)**

Las respuestas de error siguen el formato estándar `ProblemDetails` (un JSON con `type`, `title`, `status`, `errors`...), de modo que todos los clientes reciben los errores de forma predecible.

## Lo que NO hace

- **No autoriza ni autentica** — para eso está [`[Authorize]`](Authorize.md); `[ApiController]` no protege nada.
- **No sirve para vistas Razor/MVC** — es para APIs que devuelven datos, no para controladores que renderizan HTML.
- **No define las rutas por ti** — activa el routing por atributos, pero las rutas concretas las escribes tú.
- **No serializa a JSON** — de eso se encarga el formateador de salida; el atributo solo activa las convenciones.

## Buenas prácticas avanzadas

- **Declara los códigos de respuesta con `[ProducesResponseType]`** — `[ApiController]` da buenos defaults, pero documentar explícitamente cada resultado posible (`[ProducesResponseType(StatusCodes.Status404NotFound)]`) hace que Swagger/OpenAPI genere un contrato exacto y que quien consume la API sepa qué esperar sin leer tu código.
- **Personaliza el 400 automático si lo necesitas** — la validación automática es cómoda, pero a veces quieres otro formato de error. `builder.Services.Configure<ApiBehaviorOptions>(o => o.InvalidModelStateResponseFactory = ...)` te deja moldear esa respuesta en un único sitio, en vez de desactivar la validación automática y volver a los `if` manuales.
- **Cuidado con la inferencia de `[FromBody]` y los tipos simples** — la inferencia asume que un tipo complejo viene del cuerpo, pero solo puede haber **un** parámetro `[FromBody]` por acción (el cuerpo se lee una vez). Si intentas recibir dos objetos del cuerpo, tendrás un error en ejecución poco obvio: agrúpalos en un único modelo.

---

*En resumen: `[ApiController]` convierte un controlador en una API REST "con pilas incluidas" —validación automática, inferencia de binding y errores estándar— para que no repitas el mismo andamiaje en cada acción.*
