# Atributos de enrutado y verbos HTTP

## ¿Qué es?

Son los atributos que asocian un método de tu controlador a una **URL** y a un **verbo HTTP**: `[Route]`, `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]` y `[HttpPatch]`. Le dicen al framework "cuando llegue una petición a *esta* dirección con *este* verbo, ejecuta *este* método".

## ¿Por qué existe?

Una petición web no es un clic sobre un botón: es una URL con un verbo, como `GET /api/products/42`. Algo tiene que traducir esa dirección en "llama al método `GetById` con `id = 42`". El enrutado por atributos hace exactamente eso, y lo declara **al lado del método**, de modo que al leer el código ves de un vistazo qué URL atiende cada acción.

> Recuerda que estos atributos son metadatos que el motor de routing lee al arrancar ([ver Atributos](../../lenguajes/csharp-dotnet/caracteristicas-del-lenguaje/Atributos.md)); no "hacen" nada por sí mismos hasta que el framework construye la tabla de rutas.

## ¿Cuándo y para qué se usa?

En cuanto tu API tiene más de un endpoint: listar productos (`GET /api/products`), ver uno (`GET /api/products/42`), crear (`POST /api/products`), actualizar (`PUT /api/products/42`) o borrar (`DELETE /api/products/42`). Cada operación es un método con su atributo de verbo y su plantilla de ruta.

## Lo mínimo que necesitas saber

**1. `[Route]` en la clase define el prefijo común**

El token `[controller]` se sustituye por el nombre del controlador sin el sufijo `Controller` (aquí, `products`):

```csharp
[ApiController]
[Route("api/[controller]")]      // prefijo: /api/products
public class ProductsController : ControllerBase { }
```

**2. Los atributos de verbo mapean cada método, y completan la ruta**

```csharp
[HttpGet]                        // GET /api/products
public IActionResult GetAll() { }

[HttpGet("{id}")]                // GET /api/products/42
public IActionResult GetById(int id) { }

[HttpPost]                       // POST /api/products
public IActionResult Create(CreateProductRequest request) { }

[HttpPut("{id}")]                // PUT /api/products/42
public IActionResult Update(int id, UpdateProductRequest request) { }

[HttpDelete("{id}")]             // DELETE /api/products/42
public IActionResult Delete(int id) { }
```

**3. Las partes entre llaves son parámetros de ruta**

En `"{id}"`, `{id}` es un hueco que se rellena con lo que venga en la URL y se pasa al parámetro del método con el mismo nombre. El verbo importa: `GET /api/products/42` y `DELETE /api/products/42` comparten URL pero van a métodos distintos.

**4. Puedes restringir el tipo del parámetro de ruta**

`[HttpGet("{id:int}")]` hace que una petición a `/api/products/abc` devuelva `404` directamente, sin llegar a tu método. Hay restricciones para `:int`, `:guid`, `:minlength(3)`, etc.

**5. El verbo describe la intención**

`GET` lee (no modifica), `POST` crea, `PUT` reemplaza, `PATCH` modifica parcialmente y `DELETE` borra. Elegir el verbo correcto no es un capricho: es lo que hace tu API predecible y compatible con las convenciones REST.

## Lo que NO hace

- **No validan los datos** — solo deciden qué método se ejecuta; validar el contenido es cosa de las [Data Annotations](Validacion-con-DataAnnotations.md).
- **No autorizan** — que una URL exista no significa que cualquiera pueda llamarla; eso es [`[Authorize]`](Authorize.md).
- **No leen el cuerpo de la petición** — de asociar los datos entrantes a tus parámetros se encarga el [model binding](Model-Binding.md).

## Buenas prácticas avanzadas

- **No metas verbos en la URL** — la acción la expresa el método HTTP, no la ruta. `POST /api/products` está bien; `POST /api/products/create` es redundante y delata una API que no piensa en REST. La misma URL cambia de significado según el verbo, y eso es deseable.
- **Pon las rutas más específicas antes que las genéricas** — si tienes `[HttpGet("{id}")]` y `[HttpGet("featured")]`, cuida que `featured` no se interprete como un `id`. Las restricciones de tipo (`{id:int}`) resuelven la mayoría de estas ambigüedades porque descartan la ruta que no encaja.
- **Nombra las rutas para generar URLs sin hardcodearlas** — `[HttpGet("{id}", Name = "GetProductById")]` te permite construir la URL de un recurso con `CreatedAtRoute(...)` al crearlo (para la cabecera `Location` de un `201`), en vez de concatenar strings a mano y arriesgarte a que se desincronicen de la ruta real.

---

*En resumen: `[Route]` y los atributos de verbo (`[HttpGet]`, `[HttpPost]`...) declaran, al lado de cada método, qué URL y qué verbo lo activan — son el mapa que traduce cada petición HTTP en una llamada a tu código.*
