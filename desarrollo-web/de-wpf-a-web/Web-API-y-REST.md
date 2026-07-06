# Web API y REST

## ¿Qué es?

Una **Web API** es una aplicación de servidor que no devuelve páginas para mirar, sino
**datos** (normalmente en JSON) para que otro programa los consuma. **REST** es el estilo
más extendido para diseñar esas APIs: organiza todo en torno a "recursos" (productos,
pedidos, usuarios) a los que se accede con los verbos de HTTP.

## ¿Por qué existe?

Cuando la interfaz la pinta otra cosa —una página con mucho JavaScript, una app móvil,
otra aplicación— el servidor ya no necesita generar HTML; solo necesita entregar los
datos. La Web API es justo eso: el "cerebro" en el servidor que expone tus datos y reglas
de negocio de forma que cualquier cliente pueda pedirlos. Es la pieza que sustituye a las
llamadas a métodos en memoria que hacías en WPF, ahora a través de la red.

> Si vienes de WPF: una Web API es como tu capa de servicios, pero accesible por HTTP.
> Donde antes llamabas `servicio.ObtenerProductos()` en el mismo proceso, ahora el
> cliente hace `GET /api/productos` por la red y recibe la misma lista en JSON.

## ¿Cuándo y para qué se usa?

Cuando hay un cliente que no es el propio servidor: una aplicación de React o Angular, una
app móvil, un sistema externo que se integra con el tuyo. Por ejemplo, en una tienda
online moderna, la web (en el navegador) y la app del móvil pueden compartir **la misma**
Web API: ambas piden `GET /api/productos` y muestran los datos a su manera.

## Lo mínimo que necesitas saber

**1. Un controlador de API devuelve datos, no vistas**

Hereda de `ControllerBase` y sus métodos devuelven objetos que ASP.NET Core convierte a
JSON automáticamente:

```csharp
[ApiController]
[Route("api/productos")]
public class ProductosController : ControllerBase
{
    private readonly IProductoRepository _repo;
    public ProductosController(IProductoRepository repo) => _repo = repo;

    [HttpGet]
    public async Task<IActionResult> ObtenerTodos()
    {
        var productos = await _repo.ObtenerTodosAsync();
        return Ok(productos); // 200 + la lista en JSON
    }
}
```

**2. Los recursos se mapean a verbos HTTP (esto es lo "REST")**

Cada acción sobre un recurso usa el verbo que le corresponde, en vez de inventar nombres:

```text
GET    /api/productos       → listar productos
GET    /api/productos/42    → obtener el producto 42
POST   /api/productos       → crear un producto (datos en el cuerpo)
PUT    /api/productos/42    → actualizar el producto 42
DELETE /api/productos/42    → borrar el producto 42
```

**3. Los datos que llegan se enlazan a objetos C# automáticamente**

Cuando el cliente envía JSON en un `POST`, ASP.NET Core lo convierte a tu clase. Es como
deserializar, pero te lo da hecho:

```csharp
[HttpPost]
public async Task<IActionResult> Crear([FromBody] Producto nuevo)
{
    var creado = await _repo.CrearAsync(nuevo);
    return CreatedAtAction(nameof(ObtenerTodos), new { id = creado.Id }, creado); // 201
}
```

**4. Devuelves códigos de estado, no solo datos**

Parte del diseño de una API es responder con el código HTTP correcto: `200` si va bien,
`201` al crear, `404` si no existe, `400` si los datos son inválidos. El cliente decide
qué hacer según ese código.

**5. Las APIs suelen documentarse solas con OpenAPI/Swagger**

ASP.NET Core puede generar una página interactiva que lista todos los endpoints y permite
probarlos. Es el equivalente a tener un "manual de uso" siempre actualizado de tu API.

## Lo que NO hace

- **No genera interfaz** — devuelve datos en bruto; quien los muestra es el cliente (web o app).
- **No recuerda al cliente entre llamadas** — sigue siendo HTTP sin memoria; la identidad viaja en cada petición (ver [Autenticación y estado web](Autenticacion-y-Estado-Web.md)).
- **No valida tu negocio por ti** — comprobar reglas y permisos sigue siendo responsabilidad de tu código en el servidor.

## Buenas prácticas avanzadas

- **Expón DTOs, nunca tus entidades de base de datos** — devolver la entidad de EF Core directamente parece cómodo, pero acopla tu contrato público al esquema de la base de datos (renombras una columna y rompes la app móvil), filtra campos que no querías exponer (hashes, flags internos) y provoca ciclos de serialización con las relaciones. Define clases pequeñas de entrada/salida (`ProductoDto`) y mapea; el día que cambies la base de datos sin romper a nadie, se habrá pagado solo.
- **Ninguna colección sin paginar** — `GET /api/productos` que devuelve "todos" funciona con 50 productos de prueba y tumba el servidor (y al cliente) con 200.000 en producción. Pagina desde el primer día (`?page=2&pageSize=50`), con un tamaño máximo que el servidor impone aunque el cliente pida más. Es de los errores más caros de arreglar tarde, porque cambia el contrato.
- **Versiona la API antes de necesitarlo** — a diferencia de una web, los clientes de una API no se actualizan cuando tú despliegas: esa app móvil vieja seguirá llamando con el formato antiguo durante meses. Un simple `/api/v1/productos` desde el principio te da la puerta de escape para introducir `v2` con cambios incompatibles sin romper a nadie. Añadirlo después, con clientes ya en producción, es mucho más doloroso.
- **Errores con forma estándar: `ProblemDetails`** — cada endpoint inventándose su JSON de error (`{"error": ...}`, `{"mensaje": ...}`) obliga a cada cliente a adivinar. ASP.NET Core trae soporte del formato estándar *Problem Details* (RFC 7807): `return Problem(...)` o los `400` automáticos de `[ApiController]` ya lo usan. Un único formato de error para toda la API es de esas cosas que los consumidores agradecen sin saber por qué.
- **Haz seguros los reintentos en operaciones críticas** — si la app del móvil envía `POST /api/pedidos`, pierde la conexión y reintenta, ¿hay dos pedidos? El patrón profesional es aceptar una clave de idempotencia (una cabecera `Idempotency-Key` que genera el cliente): si llega dos veces la misma clave, el servidor devuelve el resultado original en vez de crear otro pedido. En pagos y pedidos no es opcional.

---

*En resumen: una Web API expone tus datos y reglas como recursos accesibles por HTTP en JSON, y REST es la convención de usar los verbos HTTP sobre esos recursos; es el backend cuando la interfaz la pone otra cosa.*
