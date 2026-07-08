# Atributos de model binding ([From...])

## ¿Qué es?

Son los atributos que le dicen a ASP.NET Core **de dónde sacar el valor** de cada parámetro de tu método: del cuerpo de la petición, de la URL, de la query string, de una cabecera o del contenedor de servicios. Todos empiezan por `From`: `[FromBody]`, `[FromRoute]`, `[FromQuery]`, `[FromForm]`, `[FromHeader]` y `[FromServices]`.

El proceso de rellenar tus parámetros con datos de la petición se llama *model binding*; estos atributos son la forma de dirigirlo cuando la deducción automática no basta.

## ¿Por qué existe?

Una petición HTTP trae datos en muchos sitios a la vez: parte en la URL (`/api/products/42`), parte en la query (`?page=2`), parte en el cuerpo (un JSON), parte en las cabeceras. El framework necesita saber qué trozo corresponde a cada parámetro de tu método. Estos atributos lo hacen explícito.

> Con [`[ApiController]`](ApiController.md), muchos de estos atributos son opcionales porque el framework infiere el origen. Se ponen cuando la inferencia no acierta o cuando quieres dejarlo claro.

## ¿Cuándo y para qué se usa?

Aparecen en la firma de las acciones: leer el `id` de la URL, el número de página de la query, el objeto a crear del cuerpo JSON, un token de una cabecera, o pedir un servicio directamente en el método sin pasarlo por el constructor.

## Lo mínimo que necesitas saber

**1. `[FromRoute]` — de la plantilla de ruta**

```csharp
[HttpGet("{id}")]                                   // GET /api/products/42
public IActionResult GetById([FromRoute] int id) { }
```

**2. `[FromQuery]` — de la query string**

```csharp
[HttpGet]                                           // GET /api/products?page=2&size=20
public IActionResult GetAll([FromQuery] int page, [FromQuery] int size) { }
```

**3. `[FromBody]` — del cuerpo de la petición (normalmente JSON)**

```csharp
[HttpPost]                                          // el JSON del body se deserializa a CreateProductRequest
public IActionResult Create([FromBody] CreateProductRequest request) { }
```

Solo puede haber **un** `[FromBody]` por acción: el cuerpo se lee una sola vez.

**4. `[FromHeader]` — de una cabecera HTTP**

```csharp
public IActionResult Get([FromHeader(Name = "X-Tenant-Id")] string tenantId) { }
```

**5. `[FromForm]` — de un formulario (`multipart/form-data`, típico al subir archivos)**

```csharp
public IActionResult Upload([FromForm] IFormFile file) { }
```

**6. `[FromServices]` — del contenedor de dependencias**

Inyecta un servicio directamente en la acción en lugar de por el constructor. Útil para dependencias que solo usa un método:

```csharp
public IActionResult Report([FromServices] IReportGenerator generator) { }
```

## Lo que NO hace

- **No validan** — solo rellenan el parámetro; comprobar que el valor es correcto es cosa de las [Data Annotations](Validacion-con-DataAnnotations.md).
- **No convierten tipos imposibles** — si la query trae `page=abc` y el parámetro es `int`, el binding falla y (con `[ApiController]`) devuelve `400`.
- **No leen el cuerpo dos veces** — de ahí la regla de un único `[FromBody]`.

## Buenas prácticas avanzadas

- **No expongas tus entidades como modelo de entrada** — vincular directamente `[FromBody] Product` (tu entidad de base de datos) abre el *over-posting*: quien llama podría rellenar campos que no debería (`IsAdmin`, `Price`...). Usa un DTO específico de entrada (`CreateProductRequest`) con solo los campos que aceptas; es la defensa más simple y efectiva.
- **`[FromServices]` conviene para dependencias puntuales, no para todo** — mete en el constructor lo que usa casi todo el controlador, y deja `[FromServices]` para el servicio caro o raro que solo necesita una acción. Así no cargas en cada petición dependencias que ese endpoint concreto no toca.
- **La cultura afecta al binding de números y fechas** — un `decimal` o un `DateTime` que llega por query o formulario se parsea según la cultura configurada; un cliente que manda `1.5` y un servidor con cultura española (donde la coma es el decimal) es una fuente clásica de valores mal interpretados. Para APIs, fija un formato invariante (por ejemplo, ISO 8601 en las fechas) y no dependas de la cultura del servidor.

---

*En resumen: los atributos `[From...]` le dicen al model binder de qué parte de la petición —ruta, query, cuerpo, cabecera o servicios— sacar cada parámetro; con `[ApiController]` muchos se infieren, pero saber cuál es cuál te salva de bindings que llegan vacíos "sin motivo".*
