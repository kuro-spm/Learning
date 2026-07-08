# WebApplicationFactory

## ¿Qué es?

`WebApplicationFactory<Program>` (del paquete `Microsoft.AspNetCore.Mvc.Testing`) arranca tu API ASP.NET Core **completa y en memoria** dentro del proceso de tests: la inyección de dependencias, el middleware y las rutas reales, pero sin abrir ningún puerto. Te da un `HttpClient` que habla directamente con ese host.

## ¿Por qué existe?

Probar una API "de verdad" exigía desplegarla en algún sitio, arrancarla, lanzar peticiones contra `localhost:5000` y confiar en que nadie más estuviera usando el puerto. `WebApplicationFactory` elimina todo eso: el test arranca la aplicación real en milisegundos, dentro del propio proceso, y las peticiones del `HttpClient` viajan directas al pipeline sin tocar la red. El resultado es un test end-to-end con la ergonomía de un test unitario.

> Si conoces el frontend, es la misma idea que jsdom para React: un entorno real *simulado dentro del proceso de test*, sin levantar la pieza pesada de verdad.

## ¿Cuándo y para qué se usa?

- **Tests de API**: comprobar que `POST /api/orders` devuelve 201 con el JSON correcto, que un endpoint protegido devuelve 401 sin token, o que el contrato de serialización es el esperado.
- **Smoke tests de arranque**: el simple hecho de que la factory arranque ya valida que la composición de DI es correcta; un `GET /healthz` que devuelve 200 detecta servicios sin registrar y configuración rota.
- **Flujos completos**: login → consultar el perfil → logout → verificar que el token ya no vale, todo con el mismo `HttpClient`.

## Lo mínimo que necesitas saber

**1. La factory mínima: heredar y fijar el entorno**

```csharp
public class TestWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
        => builder.UseEnvironment("Testing"); // no carga appsettings.Development.json
}
```

Para que el test "vea" la clase `Program` de una minimal API, añade al final de `Program.cs` la línea `public partial class Program { }` (o usa `InternalsVisibleTo`).

**2. Usarla como fixture y pedir un cliente**

```csharp
public class HealthTests : IClassFixture<TestWebApplicationFactory>
{
    private readonly TestWebApplicationFactory _factory;
    public HealthTests(TestWebApplicationFactory factory) => _factory = factory;

    [Fact]
    public async Task GET_Health_DevuelveOk()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/healthz");
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}
```

**3. Inyectar configuración antes de que arranque la app**

`UseSetting` sobreescribe la configuración *antes* de que `Program.cs` la lea — así apuntas la app a la base de datos del test (ver ficha de Testcontainers):

```csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.UseEnvironment("Testing");
    builder.UseSetting("ConnectionStrings:Default", _db.GetConnectionString());
}
```

**4. Peticiones y respuestas JSON tipadas**

Las extensiones de `System.Net.Http.Json` evitan serializar a mano:

```csharp
var response = await client.PostAsJsonAsync("/api/auth/login",
    new { email = "ana@example.com", password = "S3cret!" });

var result = await response.Content.ReadFromJsonAsync<LoginResultDto>();
Assert.NotNull(result?.Token);
```

**5. Añadir cabeceras (autenticación con Bearer token)**

```csharp
using var request = new HttpRequestMessage(HttpMethod.Get, "/api/auth/me");
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);

var response = await client.SendAsync(request);
Assert.Equal(HttpStatusCode.OK, response.StatusCode);
```

**6. Reemplazar servicios reales por dobles**

`ConfigureTestServices` se ejecuta *después* del registro normal, así que gana:

```csharp
builder.ConfigureTestServices(services =>
    services.AddSingleton<IEmailSender, FakeEmailSender>());
```

## Lo que NO hace

- **No abre un puerto real** — el `HttpClient` habla con el host en memoria; si necesitas probar con un navegador o con herramientas externas, esto no te sirve.
- **No te da una base de datos** — si la app necesita datos, tú decides: arrancarla sin base de datos (para endpoints que no la usan), o darle una real con Testcontainers.
- **No prueba el servidor web real** — Kestrel, HTTPS, certificados y proxies quedan fuera; lo que se prueba es el pipeline de la aplicación.
- **No aísla los tests entre sí automáticamente** — la factory compartida es la misma app para todos los tests de la clase; el estado que un test cree (en base de datos, en caché) lo ve el siguiente.

## Buenas prácticas avanzadas

- **Aserta el JSON crudo cuando pruebes el *contrato*** — deserializar al DTO y comparar propiedades no detecta que un enum se serializa como número en vez de como string (algo que rompe a los clientes). Comprueba el string literal para el contrato y deserializa para el resto:

```csharp
var json = await response.Content.ReadAsStringAsync();
Assert.Contains("\"role\":\"ADMIN\"", json);   // contrato: enum como STRING
var dto = JsonSerializer.Deserialize<LoginResultDto>(json, jsonOptions);
```

- **Replica las opciones de serialización de `Program.cs` en los tests** — si la app usa `JsonStringEnumConverter` y el test no, la deserialización falla o miente. Define unas `JsonSerializerOptions` estáticas en el test que copien las de la app (idealmente, extráelas a un sitio común).
- **Decide conscientemente cuándo falsear la autenticación** — para probar el *login en sí*, usa la autenticación real contra la base de datos. Para probar *otros* módulos protegidos, un `TestAuthHandler` que autentica siempre evita repetir el login en cada test. Son dos estrategias válidas; el error es no elegir.
- **Ten un test que verifique el "secure by default"** — un `GET` a un endpoint protegido *sin* token debe devolver 401. Este test vigila que nadie desactive por accidente la `FallbackPolicy` de autorización; es la alarma antirrobo de la API.
- **Dos factories mejor que una configurable** — una factory ligera sin base de datos (para smoke tests y endpoints sin datos) y otra con base de datos real. Los tests eligen la que necesitan y los baratos siguen siendo baratos.

## Recursos didácticos

- [http.cat](https://http.cat/) — cuando asertes códigos de estado, cada uno tiene su gato.

---

*En resumen: `WebApplicationFactory` arranca tu API real dentro del test — todo el pipeline HTTP, cero puertos, cero despliegues.*
