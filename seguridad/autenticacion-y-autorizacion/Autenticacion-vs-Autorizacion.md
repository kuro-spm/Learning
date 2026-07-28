# Autenticación vs Autorización

## ¿Qué es?

Dos preguntas distintas que toda aplicación con usuarios tiene que responder por separado: la **autenticación** contesta *¿quién eres?* y la **autorización** contesta *¿qué puedes hacer?*. Se abrevian **authn** y **authz** justamente porque ni siquiera en inglés se distinguen bien al escribirlas.

## ¿Por qué existe?

Se confunden por tres motivos que se refuerzan entre sí: las dos palabras empiezan igual, las dos abreviaturas se diferencian en una sola letra, y en una petición real **ocurren seguidas**, con milisegundos de diferencia. El resultado es que mucha gente las trata como un único paso llamado "el login", y de ahí sale el razonamiento que abre casi todos los agujeros de control de acceso: *si está autenticado, puede hacerlo*.

> Piensa en un concierto. En la puerta te piden el documento de identidad para comprobar que eres quien dices ser: eso es autenticación, y ocurre una vez. Lo que puedes hacer dentro lo decide tu entrada: general, grada o *backstage*. Tener un documento de identidad impecable no te mete en el camerino, y el portero del camerino no vuelve a mirarte el DNI —ya sabe quién eres— sino la pulsera.

La analogía se sostiene en el detalle importante: el control de la puerta es **uno** y sirve para todo el recinto; los controles de dentro son **muchos**, uno por zona, y cada uno decide con criterios propios. En una API pasa exactamente lo mismo.

## ¿Cuándo y para qué se usa?

En cualquier sistema donde no todo el mundo pueda hacer todo. El ejemplo que recorre esta ficha es una **tienda online**: una API en .NET en `api.tienda.ejemplo.com` y un frontend en `tienda.ejemplo.com`, con tablas `Clientes`, `Pedidos` y `Sesiones`. Los roles son `Cliente`, `Empleado` y `Administrador`.

Sobre ese sistema, las dos preguntas se separan así:

- `GET /pedidos/4711` — authn: ¿hay alguien identificado detrás de esta petición? authz: ¿ese alguien es la clienta **42**, dueña del pedido #4711, o un `Empleado` que puede consultarlo?
- `POST /pedidos` — cualquier `Cliente` autenticado puede crear pedidos. Authz apenas hace falta más allá del rol.
- `DELETE /productos/17` — solo `Administrador`. Da igual cuántos clientes autenticados haya.
- `GET /admin/informes` — solo `Administrador`, y aquí un fallo de autorización expone las ventas de todas las clientas.

Los cuatro endpoints comparten **exactamente la misma** autenticación: el mismo token, el mismo *middleware*, la misma validación. Lo que cambia en cada uno es la autorización, y cambia tanto que en `DELETE /productos/17` basta mirar un rol mientras que en `GET /pedidos/4711` hay que ir a la base de datos a ver de quién es el pedido. Esa asimetría es la razón de que valga la pena separar los dos conceptos en la cabeza antes de escribir la primera línea.

---

## La tabla que las separa

|  | Autenticación (authn) | Autorización (authz) |
|---|---|---|
| Pregunta que responde | ¿Quién eres? | ¿Puedes hacer *esto* con *este* recurso? |
| Cuándo ocurre | Al presentar la credencial y en cada petición al validar el token o la cookie | Después, una vez conocida la identidad, en **cada** petición y a veces varias veces por petición |
| Qué se comprueba | Que la credencial es válida y no ha caducado | Rol, permisos, propiedad del recurso, estado del negocio |
| Alcance | Transversal: una vez para toda la API | Específico: depende del endpoint y del dato concreto |
| Qué falla si sale mal | Cualquiera puede hacerse pasar por la clienta 42 | La clienta 42 puede leer el pedido de otra persona |
| Código HTTP | `401 Unauthorized` | `403 Forbidden` |

Fíjate en la fila de alcance, porque es la que explica el resto de la ficha: la autenticación se resuelve **una vez**, en un sitio, y ya está; la autorización hay que escribirla **caso por caso**, y por eso es la que se olvida.

## 401 frente a 403, con precisión

Aquí es donde casi todo el mundo se equivoca, y la culpa es en parte del estándar: `401` se llama `Unauthorized` pero significa **no autenticado**. El nombre es un error histórico que RFC 9110 arrastra y documenta: *"the request has not been applied because it lacks valid authentication credentials"*. Y `403 Forbidden` es el que significa *"sé quién eres y aun así no"*.

Un `401` **debe** llevar una cabecera `WWW-Authenticate` que diga cómo autenticarse. No es opcional ni cosmético: es lo que convierte el `401` en una invitación a presentar credenciales. Así responde la API de la tienda a `GET /pedidos/4711` sin token o con un token caducado:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token", error_description="The token expired at '2026-07-28T09:14:02Z'"
Content-Type: application/problem+json

{ "type": "about:blank", "title": "Unauthorized", "status": 401 }
```

El cliente lee ese `401`, entiende que su token ya no sirve y sabe qué hacer: renovarlo o mandar a la usuaria al login. Compara con la respuesta a la misma petición hecha por una clienta correctamente autenticada, pero que **no** es la dueña del pedido #4711:

```http
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{ "type": "about:blank", "title": "Forbidden", "status": 403 }
```

Sin `WWW-Authenticate`, y a propósito: no hay nada que reintentar. Volver a hacer login con las mismas credenciales daría exactamente el mismo `403`. La regla práctica:

| Situación | Código | ¿`WWW-Authenticate`? | Qué debe hacer el cliente |
|---|---|---|---|
| No llega token | `401` | Sí | Autenticarse |
| Token caducado o firma inválida | `401` | Sí | Renovar el token o volver al login |
| Token válido, rol insuficiente | `403` | No | Nada; el error es de permisos |
| Token válido, recurso de otra persona | `403` o `404` | No | Nada |

### Cuando el 403 filtra información

Esa última fila tiene truco. Si la clienta 42 pide `GET /pedidos/9002`, un pedido que existe pero es de otra persona, y la API responde `403`, acaba de aprender algo: **el pedido #9002 existe**. Un `404` habría significado "no hay tal pedido". Con esa diferencia, un script que recorra `/pedidos/1` a `/pedidos/99999` y anote qué devuelve `403` y qué devuelve `404` obtiene el número exacto de pedidos de la tienda y sus identificadores, sin leer ni uno.

Con identificadores secuenciales el cálculo es inmediato: si `/pedidos/1` a `/pedidos/4711` responden `403` y a partir de `4712` responden `404`, la tienda ha hecho 4711 pedidos. Eso es información comercial que nadie pretendía publicar, obtenida en unos segundos con una respuesta que parecía segura porque *denegaba* el acceso.

Por eso a veces se responde `404` a propósito para recursos ajenos: oculta la existencia, no solo el contenido. Cuándo elegir cada uno:

- **`403`** cuando el cliente ya sabe que el recurso existe, o cuando saberlo no aporta nada a un atacante. Un `Empleado` que intenta entrar en `/admin/informes` sabe perfectamente que ese panel existe; mentirle con un `404` solo genera tickets de soporte.
- **`404`** cuando la existencia del recurso es en sí misma un dato: pedidos, facturas, expedientes, perfiles privados. Es lo que hace GitHub con los repositorios privados, y por eso `404` es la respuesta habitual ahí.

Decídelo de forma consciente y aplícalo igual en todo el recurso. Lo peor de los tres mundos es una API que devuelve `404` cuando el pedido no existe y `403` cuando existe pero es ajeno: has escrito el oráculo y encima crees que estás protegido.

## El orden es obligatorio y no negociable

No se puede autorizar sin haber autenticado, y no es una recomendación de estilo: **autorizar es decidir sobre una identidad**, así que sin identidad verificada no hay nada sobre lo que decidir. Este código lo hace al revés:

```csharp
// ❌ Lee el rol del token ANTES de comprobar que el token es auténtico
var payload = LeerPayloadSinValidar(token);      // solo decodifica base64
if (payload.Rol != "Administrador")
    return Forbid();

return Ok(_informes.ObtenerResumen());
```

La comprobación del rol parece rigurosa y no vale nada. Un [JWT](JWT.md) no está cifrado: su payload es base64 legible y **editable**. Cualquiera coge su propio token de `Cliente`, cambia `"rol": "Cliente"` por `"rol": "Administrador"`, lo vuelve a codificar y entra en `/admin/informes`. La firma es lo único que impide eso, y este código no la mira.

```csharp
// ✅ Primero validar la credencial, después decidir sobre la identidad que contiene
var identidad = _validadorDeTokens.Validar(token);   // firma, caducidad, emisor
if (identidad is null)
    return Unauthorized();                            // 401: no sabemos quién eres

if (!identidad.EstaEnRol("Administrador"))
    return Forbid();                                  // 403: sabemos quién eres, y no

return Ok(_informes.ObtenerResumen());
```

La secuencia siempre es la misma: validar → extraer identidad → decidir. Cualquier lectura del token antes del primer paso es decoración.

## Dónde vive cada una en una API

La autenticación es transversal: se configura una vez en el *middleware* y a partir de ahí toda la aplicación tiene `HttpContext.User` rellenado. La autorización es específica del endpoint y muchas veces del dato concreto, así que se declara y se escribe *pieza por pieza*. En ASP.NET Core eso se traduce en dos líneas cuyo orden importa:

```csharp
var app = builder.Build();

app.UseRouting();
app.UseAuthentication();   // valida el token y rellena HttpContext.User
app.UseAuthorization();    // decide si ese User puede llegar al endpoint
app.MapControllers();
```

`UseAuthentication()` no rechaza nada: solo intenta identificar. Si no hay token, deja a `HttpContext.User` como anónimo y sigue. `UseAuthorization()` es quien mira el `[Authorize]` del endpoint y corta con `401` o `403`.

Si se invierten, el síntoma es inconfundible y desconcertante:

```csharp
// ❌ Autorizar antes de autenticar
app.UseAuthorization();
app.UseAuthentication();
```

**Todo endpoint con `[Authorize]` devuelve `401`, incluso con un token perfectamente válido y recién emitido.** El motivo es literal: cuando `UseAuthorization()` se ejecuta, `HttpContext.User` todavía está vacío porque `UseAuthentication()` no ha corrido, así que la autorización ve un usuario anónimo y responde lo que le corresponde a un anónimo. El token está bien; simplemente nadie lo ha leído aún. Es un error que se busca durante horas en la configuración de la firma del token, y está en dos líneas de `Program.cs`.

## La autorización a nivel de dato es la que se olvida

Este es el fallo más común del mundo real, y merece leerse despacio. `[Authorize(Roles = "Cliente")]` comprueba que quien llama es *una* clienta. No comprueba que sea **la** clienta correcta:

```csharp
// ❌ Cualquier Cliente autenticado lee cualquier pedido
[Authorize(Roles = "Cliente")]
[HttpGet("/pedidos/{id:int}")]
public async Task<IActionResult> ObtenerPedido(int id)
{
    var pedido = await _pedidos.BuscarAsync(id);
    return pedido is null ? NotFound() : Ok(pedido);
}
```

La clienta 42 se autentica con su cuenta real, cambia `4711` por `4712` en la barra del navegador y ve el pedido de otra persona: dirección de envío, teléfono, importe, artículos. El atributo hizo su trabajo —confirmó el rol— y el agujero está abierto igual. Esto se llama **IDOR** (*Insecure Direct Object Reference*): el identificador del recurso viaja en la petición y el servidor no comprueba que quien lo pide tenga relación con él.

La corrección es una comprobación de propiedad, y tiene que estar en el código del endpoint:

```csharp
// ✅ Se comprueba que el pedido pertenece a quien lo pide
[Authorize(Roles = "Cliente")]
[HttpGet("/pedidos/{id:int}")]
public async Task<IActionResult> ObtenerPedido(int id)
{
    var clienteId = int.Parse(User.FindFirst("clienteId")!.Value);   // de la identidad, nunca del cliente
    var pedido = await _pedidos.BuscarAsync(id);

    if (pedido is null || pedido.ClienteId != clienteId)
        return NotFound();     // 404 deliberado: no confirma que #4712 exista

    return Ok(pedido);
}
```

Dos detalles que hacen que funcione. Primero, `clienteId` sale de `User`, es decir, del token ya validado: si viniera de la query string o del cuerpo de la petición, el atacante lo controlaría y no habríamos arreglado nada. Segundo, la variante mejor todavía es no dar la oportunidad: `_pedidos.BuscarDeClienteAsync(id, clienteId)` mete la condición en el `WHERE` de la consulta, y así no existe la ruta de código en la que se olvida el `if`.

Y lo importante: **ningún atributo ni *middleware* puede resolver esto por ti**. `[Authorize]` se evalúa antes de que exista el pedido #4711 en memoria; no sabe qué recurso vas a leer ni de quién es. La relación entre la identidad y el dato solo la conoce tu código, después de consultar la base de datos. Por eso la autorización a nivel de dato es trabajo manual, endpoint por endpoint, y por eso se olvida. Los modelos de permisos que ordenan este trabajo —roles, claims, políticas, listas— los desarrollan [RBAC y Claims](RBAC-y-Claims.md) y [ACL](ACL.md).

## Cuándo NO aplica ninguna de las dos

Tan importante como saber dónde poner las comprobaciones es saber dónde **no** ponerlas, porque exigir identidad donde no hace falta tiene coste y, sobre todo, porque hay fallos que se parecen a un problema de permisos y no lo son.

**No exijas autenticación en lo que es público de verdad.** El catálogo de la tienda es el caso claro: `GET /productos` tiene que funcionar para quien acaba de llegar a `tienda.ejemplo.com` sin cuenta. Con una política de denegar por defecto activa, eso se declara explícitamente:

```csharp
[AllowAnonymous]                       // deliberado y visible en el código
[HttpGet("/productos")]
public async Task<IActionResult> ListarCatalogo()
    => Ok(await _productos.ListarPublicadosAsync());
```

Poner `[Authorize]` aquí "por seguridad" no protege nada —los datos ya son públicos— y sí obliga a registrarse para mirar un escaparate. Lo que sí necesita ese endpoint es un límite de peticiones, que es otro problema.

**Y no confundas estas comprobaciones con la validación de datos.** Que la clienta 42 esté autenticada y autorizada a crear pedidos no dice nada sobre si el cuerpo que envía es coherente:

```csharp
// La autorización ya dijo "sí": esta clienta puede crear pedidos
// Y aun así esto hay que comprobarlo aparte
[Authorize(Roles = "Cliente")]
[HttpPost("/pedidos")]
public async Task<IActionResult> CrearPedido(NuevoPedido nuevo)
{
    if (nuevo.Unidades <= 0)
        return BadRequest("Las unidades deben ser positivas.");   // 400, no 403

    if (!await _stock.HayDisponibleAsync(nuevo.ProductoId, nuevo.Unidades))
        return Conflict("Sin stock suficiente.");                  // 409, no 403

    var pedido = await _pedidos.CrearAsync(nuevo, clienteId: 42);
    return Created($"/pedidos/{pedido.Id}", pedido);                // 201
}
```

`400` es "tu petición está mal formada", `409` es "tu petición es válida pero choca con el estado actual" y `403` es "no te dejo". Devolver `403` para un dato inválido manda al cliente a revisar permisos que están perfectos, y devolver `400` para una falta de permiso oculta un problema de seguridad detrás de un error de formulario.

## Identidad, autenticación y factores

Un **factor de autenticación** es una categoría de prueba, y hay tres: *algo que sabes* (una contraseña), *algo que tienes* (el móvil que recibe un código, una llave física) y *algo que eres* (huella, cara). Lo que define un factor no es lo trabajoso que sea, sino la categoría: pedir dos contraseñas es un solo factor repetido, porque quien te robó una probablemente te robó las dos por la misma vía.

La **autenticación multifactor** (MFA) exige pruebas de al menos dos categorías distintas, y lo que gana es concreto: una contraseña filtrada deja de ser suficiente. Como las contraseñas se filtran constantemente —por reutilización, por *phishing*, por brechas de otros sitios—, MFA es la mejora barata más grande que existe en authn. Lo que **no** arregla es nada de lo que cuenta esta ficha: la clienta 42 autenticada con dos factores sigue viendo el pedido #4712 si falta la comprobación de propiedad. MFA sube la calidad de la respuesta a *¿quién eres?*, y con eso no se decide nada sobre permisos. Cómo se almacenan las contraseñas que respaldan el primer factor lo cubre la colección de [algoritmos de hash](../algoritmos-de-hash/README.md).

## La cadena completa de una petición

Con el ejemplo de la tienda, de principio a fin:

```
1. La clienta 42 envía email + contraseña   →  POST /sesiones            [AUTHN]
2. La API los verifica y emite algo que la recuerda (cookie o token)     [AUTHN]
3. Cada petición posterior trae esa credencial:
      GET /pedidos/4711
      Authorization: Bearer eyJhbGciOi...
4. El middleware la valida (firma, caducidad, emisor)                    [AUTHN]
5. Extrae la identidad → HttpContext.User = { clienteId: 42, rol: Cliente }
   ─────────── aquí acaba authn y solo aquí puede empezar authz ───────────
6. ¿Puede el rol Cliente entrar en este endpoint?                        [AUTHZ]
7. ¿El pedido #4711 pertenece al cliente 42?                             [AUTHZ]
8. 200 OK con el pedido  ·  401 si falla 4  ·  403/404 si falla 6 o 7
```

Los pasos 3 y 4 son los que más variantes tienen, y cada uno tiene su ficha:

- **Cómo se recuerda el login entre peticiones** —sesión guardada en el servidor frente a estado dentro del propio token— en [Sesiones vs Tokens](Sesiones-vs-Tokens.md).
- **El formato del token, su firma y sus claims** en [JWT](JWT.md).
- **Delegar el acceso a otra aplicación sin darle la contraseña** en [OAuth2](OAuth2.md), y **usar una identidad de un proveedor externo** en [OpenID Connect](OpenID-Connect.md).

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| Se devuelve `401` a quien está autenticado pero le falta permiso | Se usa `401` como "acceso denegado" genérico; el cliente reintenta el login en bucle porque cree que su credencial no sirve |
| Se devuelve `403` esperando que el cliente vuelva a autenticarse | Falta la cabecera `WWW-Authenticate` y falta el código correcto: un token caducado es `401`, no `403` |
| **Todo** devuelve `401` con un token válido y recién emitido | `UseAuthorization()` colocado antes de `UseAuthentication()`: `HttpContext.User` está vacío cuando se evalúa el permiso |
| Una clienta ve el pedido de otra cambiando el id de la URL | IDOR: falta la comprobación de propiedad; `[Authorize(Roles = "Cliente")]` no la hace |
| El endpoint de administración es accesible aunque el botón no aparezca | El rol se comprueba en el frontend y no en la API; el frontend decide qué se *muestra*, nunca qué se *permite* |
| Un script deduce cuántos pedidos hay en la tienda | El `403` de un recurso ajeno confirma su existencia; hace falta `404` deliberado |
| Un endpoint nuevo queda abierto sin que nadie lo note | No hay política de denegar por defecto: el `[Authorize]` se añade a mano y se olvida |

## Buenas prácticas avanzadas

- **Deniega por defecto en la configuración, no en la revisión de código.** Un `FallbackPolicy` que exija usuario autenticado hace que cada endpoint nuevo nazca cerrado y haya que abrirlo explícitamente con `[AllowAnonymous]`. Olvidar un `[Authorize]` es un error de omisión, invisible en el *diff*; olvidar un `[AllowAnonymous]` rompe en la primera prueba y se arregla en un minuto. Cambia la clase de fallo por otra que se detecte sola.
- **Devuelve la misma respuesta cuando no existe y cuando no es tuyo.** Es la diferencia entre proteger el contenido y proteger también la existencia. Escribe un test que pida un recurso ajeno y otro inexistente y compare cuerpo y código: si difieren en algo, has construido un oráculo. Y ojo con el canal que nadie mira: si el `404` legítimo tarda 4 ms y el "ajeno" tarda 30 porque consultó la base de datos, la diferencia sigue ahí.
- **Autoriza en la consulta, no después de la consulta.** `BuscarDeClienteAsync(4711, 42)` es estructuralmente más seguro que `BuscarAsync(4711)` seguido de un `if`, porque no existe el camino en el que alguien olvide el `if`. Cuando la única forma de leer un pedido en toda la base de código exige el identificador de quien lo pide, el IDOR deja de ser posible por construcción en lugar de por disciplina.
- **Comprueba los permisos con datos del servidor, no solo con los claims del token.** Si a un `Empleado` se le retira el rol hoy, el token que emitiste esta mañana sigue diciendo `rol: Empleado` y sigue autenticando perfectamente hasta que caduque. Para acciones destructivas o sensibles, contrasta contra la base de datos; para el resto, tokens de vida corta. Es un compromiso consciente entre latencia y ventana de revocación, no un descuido.
- **Registra las denegaciones de autorización, no solo los fallos de autenticación.** Un `401` suele ser una sesión caducada y es ruido. Un `403` es alguien identificado intentando algo que no le toca: o tu interfaz le está ofreciendo un botón que no debería ver, o está probando. Un pico de `403` de un mismo `clienteId` sobre identificadores consecutivos es la firma de un IDOR siendo explorado, y es de las pocas alertas de seguridad con casi cero falsos positivos.
- **Exige reautenticación para las acciones irreversibles.** Que una sesión lleve seis horas abierta acredita quién inició el login, no quién está delante del teclado ahora. Cambiar el email, cambiar la contraseña o borrar la cuenta deberían pedir la contraseña otra vez (*step-up authentication*). Es autenticación puesta al servicio de una decisión de autorización, y limita el daño de un dispositivo desatendido.

## Documentación oficial

- [RFC 9110 §15.5.2 (401 Unauthorized)](https://www.rfc-editor.org/rfc/rfc9110.html#name-401-unauthorized) y [§15.5.4 (403 Forbidden)](https://www.rfc-editor.org/rfc/rfc9110.html#name-403-forbidden) — la fuente que zanja cualquier discusión de equipo sobre los dos códigos. Léelas cuando alguien defienda un `403` para un token caducado: el texto dice explícitamente que el `401` es por credenciales ausentes o inválidas y que **debe** ir con `WWW-Authenticate`.
- [RFC 6750 §3 (WWW-Authenticate Response Header Field)](https://www.rfc-editor.org/rfc/rfc6750.html#section-3) — define el contenido exacto de la cabecera para tokens Bearer, incluidos los valores de `error` (`invalid_request`, `invalid_token`, `insufficient_scope`). Es la referencia para saber qué debe emitir tu API y qué puede esperar un cliente.
- [Autorización en ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authorization/introduction) — la puerta de entrada al modelo de .NET: atributos, políticas y su relación con el *middleware*. Empieza aquí antes de tocar `[Authorize]` con parámetros.
- [Autorización basada en recursos en ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authorization/resourcebased) — precisamente el caso del pedido #4711: cómo autorizar cuando la decisión depende del dato y no se puede tomar con un atributo. Ve aquí cuando el `if` de propiedad empiece a repetirse en varios endpoints.

## Recursos didácticos

- [OWASP Top 10 — A01:2021 Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/) — el control de acceso roto es la categoría número uno de la lista, y el IDOR de esta ficha es su ejemplo más citado. Trae casos reales cortos y una lista de comprobación que se puede aplicar endpoint por endpoint.
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) — la versión práctica del punto anterior: denegar por defecto, autorizar en cada capa y verificar la propiedad del recurso, con el razonamiento de por qué cada regla existe.
- [httpbin.org/basic-auth/cliente/secreto](https://httpbin.org/basic-auth/cliente/secreto) — ábrelo en el navegador y verás un `401` de verdad: el navegador reacciona a la cabecera `WWW-Authenticate` mostrando su propio diálogo de credenciales. Es la forma más rápida de entender que esa cabecera *hace* algo en lugar de solo informar.
- [http.cat](https://http.cat) — cada código de estado ilustrado con un gato. Suena a broma y funciona sorprendentemente bien para que `401` y `403` dejen de intercambiarse en la cabeza.

---

*En resumen: la autenticación dice quién eres y se resuelve una vez en el middleware; la autorización dice qué puedes hacer con este dato concreto y hay que escribirla en cada endpoint — `401` es no autenticado y lleva `WWW-Authenticate`, `403` es autenticado y sin permiso, y el pedido de otra persona no se protege con un atributo sino comprobando de quién es.*
