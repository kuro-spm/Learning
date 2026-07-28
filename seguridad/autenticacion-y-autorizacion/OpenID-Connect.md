# OpenID Connect (OIDC)

## ¿Qué es?

OpenID Connect es una capa de **identidad** construida encima de [OAuth2](OAuth2.md). Añade lo que OAuth2 deliberadamente no resuelve: una forma estandarizada de decir «esta persona es quien dice ser», mediante un tipo de token nuevo, el `id_token`, que viaja junto al `access_token` habitual.

## ¿Por qué existe?

OAuth2 resuelve autorización delegada, pero muchísimas aplicaciones lo estaban usando para resolver login («iniciar sesión con Google»), y el estándar no dice nada de eso. El patrón que se improvisó fue siempre el mismo: pedir un `access_token`, llamar después a alguna API propietaria del proveedor —`/me`, `/user`, `/v1/profile`, cada uno con su nombre— y deducir de la respuesta quién había entrado.

Ese apaño tiene dos problemas. El primero es que no hay dos proveedores iguales, así que cada integración se escribía a mano. El segundo es más grave: un `access_token` no dice para quién se emitió ni quién lo emitió, así que una aplicación podía aceptar un token conseguido en otro sitio y darle sesión a la persona equivocada.

OpenID Connect estandariza ese caso de uso: define exactamente qué información de identidad se entrega, en qué formato (un [JWT](JWT.md) firmado) y en qué orden se verifica, para que «iniciar sesión con» funcione igual en cualquier proveedor que implemente el estándar.

> Si OAuth2 es la tarjeta de acceso limitado del hotel, OIDC es el documento de identidad que el hotel te devuelve validado y sellado en el momento del *check-in*: no solo te permite entrar a tu habitación, sino que certifica ante cualquiera que pregunte que efectivamente eres quien dices ser.

## ¿Cuándo y para qué se usa?

Detrás de cualquier botón de «Iniciar sesión con Google / Microsoft / GitHub», y detrás del *single sign-on* corporativo. En vez de que la aplicación gestione registro, contraseñas, recuperación por correo y segundo factor, delega la autenticación en un proveedor de identidad y confía en lo que ese proveedor certifica.

Esta ficha **empieza donde termina la de [OAuth2](OAuth2.md)**: se da por sabido el flujo Authorization Code con PKCE y `state`, y aquí solo se explica **lo que cambia** cuando añades el scope `openid`.

## El escenario de esta ficha

La tienda online `tienda.ejemplo.com`, con su API en `api.tienda.ejemplo.com`, quiere ofrecer «iniciar sesión con» en lugar de gestionar contraseñas propias. El proveedor de identidad es `accounts.proveedor.ejemplo.com` y el `client_id` registrado para la tienda es `tienda-online`.

Al terminar el login, la tienda tiene que acabar con una fila en su tabla `Clientes` —la clienta interna **42**— para poder mostrarle su pedido **#4711**. El proveedor no sabe nada de esa fila: es trabajo de la tienda.

## `id_token` frente a `access_token`: dos tokens, dos destinatarios

Es **la** distinción de la que depende todo lo demás, y casi todos los errores de una integración OIDC son una confusión entre estas dos columnas:

| | `id_token` | `access_token` |
|---|---|---|
| Para quién es | Para el **Client** (la tienda) | Para el **Resource Server** (la API del proveedor) |
| Qué certifica | Que esta persona se autenticó, con quién y cuándo | Que el portador tiene permiso para ciertas operaciones |
| Formato | Siempre un JWT firmado (lo garantiza el estándar) | **No garantizado**: puede ser un JWT o una cadena opaca |
| Quién lo valida | El Client, con la clave pública del proveedor | El Resource Server para el que se emitió |
| Dónde se usa | Se lee una vez, en el *callback* del login | Se manda en `Authorization: Bearer` a la API |
| Si lo confundes | Mandarlo a una API como credencial: cuela hoy y rompe el día que el proveedor cambie su contenido | Usarlo para identificar: no sabes ni quién lo emitió ni para quién |

El punto que se escapa: el `access_token` **no es tuyo**. Su formato y su contenido son decisión del proveedor y pueden cambiar sin avisar, porque el único consumidor legítimo es su propia API. Si lo abres para leer el `sub`, estás dependiendo de un detalle interno de otro sistema.

```csharp
// ❌ Identificar a la clienta a partir del access_token
var claims = new JwtSecurityTokenHandler().ReadJwtToken(accessToken);
var clienteExterno = claims.Subject;   // hoy funciona; mañana es una cadena opaca
```

```csharp
// ✅ La identidad sale del id_token, que es para ti y está validado
var clienteExterno = principalDelIdToken.FindFirst("sub")!.Value;
```

## El scope `openid`: lo único que cambia en el flujo

Lo que convierte un flujo de OAuth2 en un flujo de OIDC es un scope. Sin `openid` no hay `id_token`; con él, el proveedor está obligado a devolverlo.

La petición de autorización es la de OAuth2 con dos añadidos, `scope=openid ...` y `nonce`:

```
GET https://accounts.proveedor.ejemplo.com/authorize
  ?response_type=code
  &client_id=tienda-online
  &redirect_uri=https%3A%2F%2Ftienda.ejemplo.com%2Fsignin-oidc
  &scope=openid%20email%20profile
  &state=9f2a41c8e7b04d15
  &nonce=b7c19e4a2f8d3061
  &code_challenge=Cc1iJyzKPO0ZsRzv2uAqPnJYzowFLIdWxhUJJEbH1D4
  &code_challenge_method=S256
```

Solo tres parámetros son nuevos respecto a la ficha de OAuth2:

- `openid` — obligatorio y siempre el primero de la lista. Es el interruptor del protocolo.
- `email profile` — scopes estándar de OIDC que piden los claims de perfil. No son inventados por el proveedor: el estándar fija qué claim trae cada uno.
- `nonce` — valor aleatorio que tiene que reaparecer **dentro** del `id_token`. Tiene sección propia más abajo.

El canje del código es idéntico al de OAuth2, pero la respuesta trae un campo más:

```json
{
  "token_type": "Bearer",
  "expires_in": 900,
  "access_token": "def50200a1b9c4e8f7d3a2...",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjA0Y2EifQ.eyJpc3MiOiJodHRwczov...",
  "scope": "openid email profile"
}
```

Fíjate en un detalle del ejemplo: el `access_token` es una cadena opaca sin puntos y el `id_token` tiene las tres partes de un JWT. Es lo normal y es exactamente lo que decía la tabla anterior.

## El `id_token` desglosado, claim a claim

El `id_token` es un JWT: tres partes separadas por puntos, firmadas. Su estructura, la codificación base64url y los ataques sobre la firma están en [JWT](JWT.md); aquí interesa el *payload*, que una vez decodificado se lee así:

```json
{
  "iss": "https://accounts.proveedor.ejemplo.com",
  "sub": "8f14e45fceea167a",
  "aud": "tienda-online",
  "exp": 1785226200,
  "iat": 1785225300,
  "auth_time": 1785225240,
  "nonce": "b7c19e4a2f8d3061",
  "email": "ana.garcia@correo.ejemplo.com",
  "email_verified": true,
  "name": "Ana García",
  "picture": "https://accounts.proveedor.ejemplo.com/fotos/8f14e45f.jpg"
}
```

Los cinco primeros son obligatorios y son los que se validan:

- `iss` (*issuer*) — quién emitió el token. Tiene que coincidir **exactamente** con el `issuer` del documento de descubrimiento, carácter por carácter, barra final incluida.
- `sub` (*subject*) — el identificador de la persona **dentro de ese proveedor**. Opaco, estable y sin significado: no intentes leerlo.
- `aud` (*audience*) — para quién es el token. Aquí es tu `client_id`, `tienda-online`. Es lo que impide que un `id_token` emitido para otra aplicación te sirva de login.
- `exp` / `iat` — caducidad y momento de emisión, en segundos desde 1970. En el ejemplo, 900 segundos de vida: quince minutos.
- `nonce` — el valor que mandaste en la petición de autorización, devuelto firmado.

Y dos que casi nadie usa y resuelven problemas reales:

- `auth_time` — cuándo se autenticó de verdad la persona, que **no** es lo mismo que `iat`. Si el proveedor ya tenía sesión abierta, el login es silencioso y `auth_time` puede ser de hace tres días. Para una operación sensible —cambiar la dirección de envío de un pedido— puedes exigir que `auth_time` sea reciente y, si no lo es, pedir una reautenticación con `prompt=login` y `max_age`.
- `email`, `email_verified`, `name`, `picture` — los claims de perfil, que llegan porque pediste los scopes `email` y `profile`. Son opcionales: un proveedor conforme puede no mandar ninguno.

## `sub` es la clave estable; el correo no lo es

La clave con la que guardas a la persona en tu base de datos tiene que ser `sub`, nunca el correo. El correo es un dato de contacto que la persona puede cambiar en el proveedor cuando quiera, y cuando lo cambia pasa una de dos cosas, las dos malas.

Supón que la tabla `Clientes` guarda el correo como identificador externo. La clienta 42 entró la primera vez como `ana.garcia@correo.ejemplo.com` y el mes siguiente lo cambia en el proveedor a `ana.g@correo.ejemplo.com`:

| Momento | `sub` en el `id_token` | `email` en el `id_token` | Qué encuentra la tienda |
|---|---|---|---|
| Primer login | `8f14e45fceea167a` | `ana.garcia@…` | Crea la clienta 42 |
| Tras el cambio, buscando por correo | `8f14e45fceea167a` | `ana.g@…` | **Nada**: crea la clienta 87, vacía, sin el pedido #4711 |
| Tras el cambio, buscando por `sub` | `8f14e45fceea167a` | `ana.g@…` | La clienta 42, con su historial |

La fila del medio es el caso benigno: la clienta pierde su cuenta y abre una incidencia. El caso grave es el inverso. Si el proveedor libera el correo antiguo y otra persona lo registra, esa persona entra en la cuenta de la clienta 42 con su historial de pedidos y sus direcciones. Ese es el mecanismo real de la mayoría de los secuestros de cuenta por *social login*.

```csharp
// ❌ El correo como clave de identidad
var cliente = await db.Clientes.SingleOrDefaultAsync(c => c.Email == email);
```

```csharp
// ✅ El par (iss, sub) como clave de identidad; el correo es solo un dato
var cliente = await db.Clientes.SingleOrDefaultAsync(
    c => c.Issuer == iss && c.Subject == sub);
```

La regla completa es que la clave es el **par** (`iss`, `sub`), no `sub` a secas: dos proveedores distintos pueden emitir el mismo `sub` sin que ninguno haga nada mal.

## La trampa de `email_verified`

`email_verified: true` significa «este proveedor comprobó que la persona controla ese correo». Cuando es `false`, o cuando no viene, significa que el proveedor dejó escribir un correo cualquiera en un formulario.

El ataque tiene dos versiones y las dos acaban igual:

1. **Aceptar un correo sin verificar.** Un atacante se registra en `accounts.proveedor.ejemplo.com` poniendo `ana.garcia@correo.ejemplo.com` como su correo. El proveedor no lo verifica, pero emite un `id_token` con firma impecable, `iss` correcto y `aud` correcto. Todas tus validaciones pasan.
2. **Vincular cuentas por correo.** Es la funcionalidad simpática de «si ya tenías cuenta con contraseña y entras con el proveedor, te las unimos». Sin comprobar `email_verified`, esa vinculación entrega la cuenta con contraseña de la clienta 42 a quien haya puesto su correo en un proveedor descuidado.

```csharp
// ❌ Vinculación por correo sin comprobar la verificación
var cliente = await db.Clientes.SingleOrDefaultAsync(c => c.Email == email);
if (cliente is not null) VincularConProveedor(cliente, iss, sub);
```

```csharp
// ✅ Solo se vincula si el proveedor verificó el correo, y aun así se confirma
if (emailVerified is not true)
    return RedirectToAction("PedirCorreoPropio");   // no vinculamos a ciegas

var cliente = await db.Clientes.SingleOrDefaultAsync(c => c.Email == email);
if (cliente is not null) await EnviarConfirmacionDeVinculacion(cliente, iss, sub);
```

La regla en una línea: **la identidad es el par (`iss`, `sub`); el correo es un atributo**, y solo es utilizable como pista de vinculación si viene con `email_verified: true` de un proveedor en el que confías para eso. Y ni «confío en este proveedor» es una decisión global: un proveedor puede verificar los correos de su propio dominio y no los de fuera.

## El `nonce` y el ataque de repetición

El `nonce` es al `id_token` lo que el `state` es al flujo: un valor aleatorio de un solo uso que ata la respuesta a la petición que tú hiciste. La diferencia es dónde vuelve. El `state` vuelve como parámetro en la URL del *callback*; el `nonce` vuelve **dentro del token firmado**, que es lo que lo hace imposible de falsificar.

El ataque que cierra, paso a paso:

1. El atacante inicia un login legítimo con su propia cuenta en el proveedor y captura el `id_token` que le devuelven, o consigue uno de otra sesión.
2. Provoca que el navegador de la clienta 42 llegue al *callback* de la tienda con ese token inyectado.
3. Sin `nonce`, la tienda valida firma, `iss`, `aud` y `exp` — todo correcto, el token es auténtico — y abre sesión con la identidad del atacante en el navegador de la clienta.

El resultado es que la clienta acaba navegando la tienda como otra persona, y todo lo que haga (añadir una tarjeta, cambiar una dirección) va a la cuenta del atacante. Con `nonce`, el paso 3 falla: el token reproducido lleva **el `nonce` de la sesión del atacante**, y la tienda espera el que guardó ella.

Los tres pasos que hay que implementar, y ninguno es opcional:

- Generar un valor aleatorio nuevo por intento de login (no reutilizar, no derivar del `state`).
- Guardarlo en un sitio que el atacante no controle: sesión de servidor o cookie firmada. En ASP.NET Core es la cookie de correlación, y por eso su pérdida rompe el login.
- Al recibir el token, comparar el claim `nonce` con el guardado y **abortar si no hay ninguno guardado**, no solo si no coincide.

## El orden de validación del `id_token`, que no es negociable

Hay un orden, y saltárselo convierte los claims en datos que envía cualquiera:

```
1. firma  →  2. iss  →  3. aud  →  4. exp / iat  →  5. nonce
```

El motivo del orden es que **cada paso solo tiene sentido si el anterior pasó**. Un `email_verified: true` es un dato precioso si el token está firmado por quien esperas y dirigido a ti; sin eso, es texto que ha escrito alguien.

```csharp
// ❌ Leer un claim antes de validar nada
var payload = new JwtSecurityTokenHandler().ReadJwtToken(idToken);
if (payload.Claims.First(c => c.Type == "email_verified").Value == "true")
    IniciarSesion(payload.Subject);      // cualquiera puede fabricar este token
```

```csharp
// ✅ Validar y solo después leer. Estos son los parámetros que importan
var parametros = new TokenValidationParameters
{
    ValidateIssuerSigningKey = true,                                  // 1
    IssuerSigningKeys        = clavesDelDocumentoDeDescubrimiento,    // 1 (por kid)
    ValidateIssuer           = true,
    ValidIssuer              = "https://accounts.proveedor.ejemplo.com", // 2
    ValidateAudience         = true,
    ValidAudience            = "tienda-online",                       // 3
    ValidateLifetime         = true,
    ClockSkew                = TimeSpan.FromMinutes(2)                // 4
};

var principal = handler.ValidateToken(idToken, parametros, out _);
// 5. el nonce se compara aparte, contra el valor guardado en la cookie de correlación
```

Con `AddOpenIdConnect` los cinco pasos los hace el *middleware*. Merece la pena saberlos porque son exactamente los que fallan con un mensaje `IDX...` cuando algo está mal configurado.

## El documento de descubrimiento

Todo proveedor OIDC serio publica su configuración en una URL fija derivada del `issuer`:

```http
GET /.well-known/openid-configuration HTTP/1.1
Host: accounts.proveedor.ejemplo.com
```

Recortado a los campos que se usan de verdad:

```json
{
  "issuer": "https://accounts.proveedor.ejemplo.com",
  "authorization_endpoint": "https://accounts.proveedor.ejemplo.com/authorize",
  "token_endpoint": "https://accounts.proveedor.ejemplo.com/token",
  "userinfo_endpoint": "https://accounts.proveedor.ejemplo.com/userinfo",
  "jwks_uri": "https://accounts.proveedor.ejemplo.com/.well-known/jwks.json",
  "end_session_endpoint": "https://accounts.proveedor.ejemplo.com/logout",
  "scopes_supported": ["openid", "email", "profile"],
  "response_types_supported": ["code"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "code_challenge_methods_supported": ["S256"]
}
```

Esto es lo que permite que una librería se configure sola: le das `Authority = "https://accounts.proveedor.ejemplo.com"` y de ahí saca las tres URLs del flujo, la de las claves públicas y la de logout. Es también dónde comprobar, antes de escribir código, si el proveedor soporta PKCE (`code_challenge_methods_supported`) o cierre de sesión (`end_session_endpoint`).

**Cachéalo, y cachea las claves, pero respeta su expiración.** Descargarlo en cada login añade dos llamadas de red a cada inicio de sesión; no refrescarlo nunca rompe el día que el proveedor rote sus claves de firma. El comportamiento correcto es el que traen las librerías: cachear según las cabeceras de la respuesta y, cuando llegue un token con un `kid` desconocido, volver a bajar el `jwks_uri` una vez antes de rechazarlo. Si lo implementas a mano, implementa esa recarga por `kid` o el login se caerá entero en la siguiente rotación.

## El endpoint `/userinfo`: cuándo hace falta y cuándo estorba

El `id_token` no tiene por qué traer el perfil completo. Muchos proveedores mandan lo mínimo dentro del token —a menudo solo `sub`— para no engordarlo, y dejan el resto en `/userinfo`. Se llama con el `access_token`, no con el `id_token`:

```http
GET /userinfo HTTP/1.1
Host: accounts.proveedor.ejemplo.com
Authorization: Bearer def50200a1b9c4e8f7d3a2...
```

```json
{
  "sub": "8f14e45fceea167a",
  "email": "ana.garcia@correo.ejemplo.com",
  "email_verified": true,
  "name": "Ana García",
  "picture": "https://accounts.proveedor.ejemplo.com/fotos/8f14e45f.jpg"
}
```

El `sub` de esta respuesta **tiene que coincidir** con el del `id_token` que acabas de validar. Si no coincide, el `access_token` pertenece a otra persona y hay que abortar; el estándar lo exige explícitamente.

Y el error de diseño que se ve mucho: **llamar a `/userinfo` en cada petición de tu API**. Es una llamada de red a un tercero, con su latencia y su cuota, metida en el camino crítico de cada `GET /pedidos/4711`. Si el proveedor tiene un mal día, tu tienda también. Se llama **una vez, en el login**, y lo que necesites de ahí lo guardas en tu fila de `Clientes` o en tu propia sesión.

## Configuración en ASP.NET Core

```csharp
builder.Services.AddAuthentication(options =>
{
    // Quién resuelve "¿quién es esta persona?" en cada petición: la cookie.
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    // Quién actúa cuando hace falta autenticar: el redirect al proveedor.
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    options.Authority = "https://accounts.proveedor.ejemplo.com";  // descubrimiento
    options.ClientId = "tienda-online";
    options.ClientSecret = builder.Configuration["Oidc:ClientSecret"];
    options.ResponseType = "code";          // Authorization Code, nunca implicit
    options.UsePkce = true;                 // por defecto en .NET, déjalo explícito
    options.CallbackPath = "/signin-oidc";  // tiene que estar registrada tal cual

    options.Scope.Clear();
    options.Scope.Add("openid");            // sin esto no hay id_token
    options.Scope.Add("email");
    options.Scope.Add("profile");

    options.GetClaimsFromUserInfoEndpoint = true;  // una llamada, en el login
    options.SaveTokens = true;              // guarda id_token y access_token en la cookie

    // Del claim del proveedor al claim que usa tu código
    options.TokenValidationParameters.NameClaimType = "name";
    options.ClaimActions.MapJsonKey("email_verified", "email_verified");
});
```

Cuatro decisiones que conviene entender antes de copiar esto:

- **`DefaultScheme` es la cookie, no OIDC**, y eso es correcto: OIDC solo interviene en el instante del login. Si pones OIDC como `DefaultScheme`, cada petición intenta rearrancar un flujo de autorización y acabas con redirecciones en bucle o con `401` en llamadas AJAX.
- **`SaveTokens = true` guarda los tokens dentro de la cookie de sesión**, lo que la engorda unos cientos de bytes y mete el `access_token` en el navegador. Actívalo solo si de verdad vas a llamar a la API del proveedor después; si no, no lo necesitas.
- **`Scope.Clear()` antes de añadir** porque `AddOpenIdConnect` ya trae `openid` y `profile` por defecto, y duplicarlos hace que algunos proveedores rechacen la petición.
- **`CallbackPath` es parte de la `redirect_uri`** que se registra en el proveedor, y la comparación es exacta. Es el origen de la mitad de los `redirect_uri_mismatch`.

## Después del login: la sesión es tuya

Este es el paso que casi nadie explica y el que más se hace mal. El `id_token` **sirve para el instante del login, no para la sesión**. Es una fotografía de un hecho ocurrido a las 07:55, con quince minutos de validez, y no está pensado para viajar en cada petición.

Lo que hace una integración correcta:

1. Recibe el `id_token` en el *callback* y lo valida en el orden de arriba.
2. Resuelve o crea la fila de `Clientes` a partir de (`iss`, `sub`).
3. Emite **su propia** sesión: una cookie de la tienda, o un token propio si la API es sin estado.
4. A partir de ahí, el `id_token` no vuelve a viajar. Ni a `api.tienda.ejemplo.com`, ni al frontend, ni a ningún sitio.

```csharp
// ❌ Reenviar el id_token del proveedor a tu propia API
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", idToken);
```

```csharp
// ✅ La tienda emite su propia credencial con sus propios claims
var identidad = new ClaimsIdentity(new[]
{
    new Claim(ClaimTypes.NameIdentifier, "42"),   // el id interno, no el sub del proveedor
    new Claim("email", "ana.garcia@correo.ejemplo.com"),
    new Claim(ClaimTypes.Role, "Cliente")
}, CookieAuthenticationDefaults.AuthenticationScheme);

await HttpContext.SignInAsync(new ClaimsPrincipal(identidad));
```

La razón de fondo es que el `id_token` no tiene los claims que tu API necesita —el identificador interno 42, los roles— y su caducidad la decide el proveedor, no tú. Qué forma dar a esa sesión propia (cookie o token) lo desarrolla [Sesiones vs Tokens](Sesiones-vs-Tokens.md), y cómo renovarla sin volver al proveedor, [Modelo JWT + Refresh Token](JWT-Refresh.md).

## Aprovisionamiento de la cuenta local

El proveedor dice **quién** es; qué puede hacer en la tienda lo dices tú. La clienta 42 necesita su fila en `Clientes` para que el pedido #4711 tenga a quién pertenecer, y esa fila la crea la tienda en el primer login:

```sql
-- La tabla de vinculación: una persona puede entrar por varios proveedores
CREATE TABLE IdentidadesExternas (
    Issuer     varchar(255) NOT NULL,   -- https://accounts.proveedor.ejemplo.com
    Subject    varchar(255) NOT NULL,   -- 8f14e45fceea167a
    ClienteId  int          NOT NULL,   -- 42
    PRIMARY KEY (Issuer, Subject),
    FOREIGN KEY (ClienteId) REFERENCES Clientes(Id)
);
```

Con esta tabla, la misma clienta 42 puede tener dos identidades externas (dos proveedores distintos) y sigue siendo una sola cuenta con un solo historial de pedidos. Sin ella, cada proveedor genera un cliente nuevo.

Los roles `Cliente`, `Empleado` y `Administrador` **no vienen del proveedor**, y esto no es una limitación técnica: son conceptos de tu negocio de los que el proveedor de identidad no tiene ni puede tener noticia. Aunque el proveedor mande un claim `roles`, no es la autoridad sobre quién administra tu tienda. Los roles se leen de tu base de datos al crear la sesión, como se explica en [RBAC y Claims](RBAC-y-Claims.md).

## El cierre de sesión, que es más difícil de lo que parece

Hay **dos sesiones** independientes: la de la tienda y la del proveedor. Cerrar una no cierra la otra, y de ahí sale la queja clásica: «he pulsado cerrar sesión y al volver a entrar se mete solo sin pedirme nada». Lo que ha pasado es que la cookie de la tienda se borró, pero el proveedor sigue con su sesión abierta, así que el nuevo flujo se resuelve en silencio.

Si quieres cerrar las dos, hay que pedirlo con RP-Initiated Logout: redirigir al `end_session_endpoint` que salía en el documento de descubrimiento.

```
GET https://accounts.proveedor.ejemplo.com/logout
  ?id_token_hint=eyJhbGciOiJSUzI1NiIsImtpZCI6IjA0Y2EifQ...
  &post_logout_redirect_uri=https%3A%2F%2Ftienda.ejemplo.com%2Fadios
  &client_id=tienda-online
```

- `id_token_hint` — el `id_token` de la sesión que se cierra. Le dice al proveedor **qué** sesión cerrar y le evita preguntar «¿seguro que quieres salir?». Requiere `SaveTokens = true`, que es la razón principal para activarlo.
- `post_logout_redirect_uri` — a dónde volver después. Como la `redirect_uri` del login, hay que registrarla de antemano y la comparación es exacta.

Al revés no funciona automáticamente: si la persona cierra sesión en el proveedor, tu cookie sigue viva hasta que caduque. Para eso existe el ***back-channel logout***, en el que el proveedor llama de servidor a servidor a un endpoint tuyo para avisar de que invalides la sesión — imprescindible si compartes sesión entre varias aplicaciones, y solo implementable si tus sesiones se pueden revocar del lado del servidor.

## Cuándo NO usar OIDC

- **Para el login propio con tus credenciales.** Si la clienta se autentica con la cuenta de la tienda y no hay proveedor externo, no hay identidad que delegar: montar OIDC contra ti mismo añade un Authorization Server, un documento de descubrimiento y un flujo de redirecciones para hacer lo que hace una cookie. Ve a [Sesiones vs Tokens](Sesiones-vs-Tokens.md).
- **Para autorizar llamadas entre tus propios servicios.** No hay nadie delante que se autentique, así que no hay `id_token` que emitir. Eso es Client Credentials de [OAuth2](OAuth2.md), y punto.
- **Cuando la dependencia no es asumible.** Delegar la identidad significa que si `accounts.proveedor.ejemplo.com` está caído, **nadie entra en tu tienda**, ni la clienta 42 ni tus empleados. Y si el proveedor cierra una cuenta, tú pierdes a ese cliente. Si eso no es tolerable, o mantienes una vía de acceso propia en paralelo, o no delegas.

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| `IDX10311: RequireNonce is 'true' (default) but validationContext.Nonce is null` | El `id_token` llegó sin `nonce`, o la cookie de correlación con el valor guardado se perdió. Nunca se arregla poniendo `RequireNonce = false`: eso desactiva la defensa del ataque de repetición |
| `IDX10214: Audience validation failed. Audiences: 'otra-app'. Did not match: validationParameters.ValidAudience: 'tienda-online'` | El `aud` del token no es tu `client_id`. Token de otra aplicación, o `ClientId` mal configurado |
| `IDX10205: Issuer validation failed` | El `iss` no coincide exactamente con el `Authority`: sobra o falta la barra final, o el proveedor emite con un *tenant* distinto del configurado |
| `Correlation failed. Unknown transaction.` | **El más frecuente y el peor documentado.** La cookie `.AspNetCore.Correlation.*` no llegó de vuelta: `SameSite=Lax` con un `POST` de retorno, sitio sin HTTPS, o varias instancias detrás de un balanceador sin claves de *Data Protection* compartidas (cada instancia cifra la cookie con su propia clave) |
| El login funciona en local y falla tras un proxy inverso | La `redirect_uri` se construye con `http://` porque el proxy no reenvía `X-Forwarded-Proto`, o falta `UseForwardedHeaders` antes de `UseAuthentication`. El proveedor la compara con la registrada en `https` y no coincide |
| Una clienta cambió su correo y perdió sus pedidos | Se guardó el correo como identificador en lugar del par (`iss`, `sub`) |
| Alguien entró en la cuenta de otra persona tras «vincular cuentas» | Se vinculó por correo sin exigir `email_verified: true` |
| «Ya sé quién es, tengo su `access_token`» | Se usó un token de autorización como prueba de identidad. La identidad está en el `id_token` |
| Cerrar sesión no cierra nada: al pulsar entrar, entra solo | Se borró la cookie de la tienda pero no se llamó al `end_session_endpoint` del proveedor |

## Buenas prácticas avanzadas

- **No uses nunca el `access_token` para identificar, ni el `id_token` como credencial de API.** Son los dos lados del mismo error. El formato del `access_token` es un detalle interno del Resource Server que puede cambiar sin aviso y sin romper a nadie más que a ti; el `id_token` es evidencia de un login ocurrido, no un permiso, y si lo aceptas en tu API estás aceptando una audiencia que no es la tuya. La regla operativa: en tu API, un `id_token` en la cabecera `Authorization` debería ser un `401`.
- **Valida el `nonce` contra la sesión, y aborta cuando no haya nada guardado.** El fallo sutil no es olvidar el parámetro, es guardarlo donde el atacante también llega —`localStorage`, una cookie sin `HttpOnly`— y «validar» comparando dos valores que vienen de la misma petición. Y el flujo tiene que abortar también cuando no hay valor guardado, no solo cuando no coincide: si no, un `nonce` ausente pasa la comprobación por defecto y el ataque de repetición vuelve a estar abierto.
- **No leas ningún claim antes de validar firma, `iss` y `aud`, en ese orden.** Un `email_verified: true` es un dato de un tercero de confianza solo después de esos tres pasos; antes es una cadena que ha escrito cualquiera. Es la diferencia entre `ValidateToken` y `ReadJwtToken`, dos métodos con nombres parecidos y consecuencias opuestas, y el segundo aparece en los ejemplos de internet porque «así se ve el contenido».
- **Separa el instante del login del resto de la sesión, y ponle tu propia caducidad.** El `id_token` establece quién entró; a partir de ese punto manda tu cookie o tu token, con tus claims (la clienta 42, el rol `Cliente`) y tu política de expiración. Quien mezcla las dos cosas acaba con sesiones que caducan a los quince minutos porque heredó el `exp` del proveedor, o con `id_token` reenviados a APIs internas que los aceptan porque nadie valida `aud`.
- **Cachea el descubrimiento y las claves, pero implementa la recarga por `kid`.** Casi todas las librerías lo hacen; el problema es quien valida a mano o cachea las claves «para siempre» en un `static`. La rotación de claves del proveedor es un evento normal y sin aviso: el día que ocurre, un `id_token` llega con un `kid` que no tienes, y la diferencia entre una recarga y una caída total del login es el bloque de código que baja el `jwks_uri` **una vez** antes de rechazar el token.

## Documentación oficial

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) — la especificación base. Ve a la sección 2 para la lista normativa de claims del `id_token` con cuáles son obligatorios, y a la **3.1.3.7** para los siete pasos de validación en su orden exacto: es el texto que zanja cualquier discusión sobre qué se comprueba antes de qué. La sección 5.3 fija el contrato de `/userinfo`, incluida la obligación de comparar el `sub`.
- [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html) — define `/.well-known/openid-configuration` y el significado de cada campo. Consúltala cuando necesites saber si algo que te ofrece un proveedor es estándar o una extensión suya, y para la regla de cómo se deriva la URL del `issuer`.
- [OpenID Connect RP-Initiated Logout 1.0](https://openid.net/specs/openid-connect-rpinitiated-1_0.html) — el cierre de sesión, que en Core no está. Corta y de lectura obligada antes de implementar logout: explica qué hace exactamente `id_token_hint` y por qué sin él el proveedor puede pedir confirmación a la persona.
- [Autenticación OIDC en ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authentication/configure-oidc-web-authentication) — la referencia de `AddOpenIdConnect` con todas las opciones y sus valores por defecto. Ve ahí para comprobar qué hace el *middleware* sin que se lo pidas, y a los eventos (`OnTokenValidated`, `OnRemoteFailure`) cuando necesites enganchar el aprovisionamiento de la cuenta local o dar un error decente.

## Recursos didácticos

- [OpenID Connect Playground](https://openidconnect.net/) — ejecuta el flujo completo contra un proveedor de pruebas o contra el tuyo, mostrando cada petición, la respuesta del `POST /token` y el `id_token` decodificado con sus claims. Es la forma más rápida de ver lo de esta ficha ocurriendo de verdad, y de comprobar qué contesta tu proveedor cuando quitas el scope `openid`.
- [oidcdebugger.com](https://oidcdebugger.com/) — construye la petición de autorización parámetro a parámetro contra tu propio proveedor y te devuelve la respuesta cruda. Es la herramienta para aislar si el problema está en tu URL o en el registro del cliente, sin tocar el código de la aplicación.
- [jwt.ms](https://jwt.ms) — decodifica un `id_token` y, a diferencia de un visor genérico, **describe qué significa cada claim** en una segunda pestaña. Útil para leer el token de un proveedor nuevo y entender qué te está mandando de verdad.
- [OpenID Connect explicado, de Connect2id](https://connect2id.com/learn/openid-connect) — el recorrido en prosa más claro que hay sobre cómo OIDC se apoya en OAuth2, con la separación entre los dos tokens bien explicada. Buena lectura previa a la especificación.

---

*En resumen: OpenID Connect añade a OAuth2 la pieza que le faltaba —un `id_token` firmado que certifica la identidad de la usuaria— convirtiendo un protocolo de autorización en la base estándar del «iniciar sesión con» de internet.*
