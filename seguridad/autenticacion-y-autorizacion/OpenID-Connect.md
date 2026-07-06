# OpenID Connect (OIDC)

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

OpenID Connect es una capa de **identidad** construida encima de [OAuth2](OAuth2.md). Añade lo que OAuth2 deliberadamente no resuelve: una forma estandarizada de decir "esta usuaria es quien dice ser", mediante un tipo de token nuevo, el `id_token`, que viaja junto al `access_token` habitual de OAuth2.

## ¿Por qué existe?

OAuth2 resuelve autorización delegada, pero muchísimas aplicaciones lo estaban usando, de forma no estándar, para resolver login ("iniciar sesión con Google"): pedían un access_token y luego hacían una llamada extra a alguna API del proveedor para averiguar quién era la usuaria, cada una a su manera y con sus propios riesgos si esa llamada no se validaba bien.

OpenID Connect estandariza ese caso de uso: define exactamente qué información de identidad se entrega, en qué formato (un JWT, ver esa ficha) y cómo se verifica, para que "iniciar sesión con" funcione igual en cualquier proveedor que implemente el estándar.

> Si OAuth2 es la tarjeta de acceso limitado del hotel, OIDC es el documento de identidad que el hotel te devuelve validado y sellado en el momento del check-in: no solo te permite entrar a tu habitación, sino que certifica ante cualquiera que preguntes que efectivamente eres quien dices ser.

## ¿Cuándo y para qué se usa?

Aparece en cualquier botón de "Iniciar sesión con Google / Microsoft / GitHub" que veas en una tienda online, un SaaS o una app de tareas: en vez de que la aplicación gestione su propio registro y contraseñas, delega la autenticación en un proveedor de identidad externo y confía en la identidad que ese proveedor certifica.

## Lo mínimo que necesitas saber

**1. `id_token` vs `access_token`: propósitos distintos**

El `access_token` (heredado de OAuth2) sirve para llamar a APIs en nombre de la usuaria. El `id_token`, nuevo en OIDC, es un JWT que certifica la identidad de la usuaria en el momento del login: quién es, con qué proveedor se autenticó y cuándo. No se usa para llamar a APIs, solo para que el Client sepa quién acaba de entrar.

**2. Claims estándar del `id_token`**

```json
{
  "iss": "https://accounts.proveedor.com",
  "sub": "1234567890",
  "aud": "mi-tienda-online",
  "exp": 1720003600,
  "email": "ana@example.com",
  "email_verified": true,
  "name": "Ana García"
}
```

`sub` (subject) es el identificador único y estable de la usuaria en ese proveedor — es lo que deberías guardar como clave, no el email (que puede cambiar).

**3. El endpoint UserInfo**

Cuando el `id_token` no trae todos los datos que necesitas, el Client puede llamar al endpoint `/userinfo` del proveedor con el `access_token` para pedir más detalles del perfil, siguiendo los scopes concedidos (`profile`, `email`...).

**4. El documento de descubrimiento**

Cualquier proveedor OIDC serio publica su configuración en una URL fija:

```
GET https://accounts.proveedor.com/.well-known/openid-configuration
```

Ahí se encuentran las URLs de autorización, de token, del endpoint UserInfo y la ubicación de las claves públicas para verificar firmas — así una librería cliente puede configurarse automáticamente solo con la URL base del proveedor.

**5. Configuración típica en ASP.NET Core**

```csharp
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    options.Authority = "https://accounts.proveedor.com";
    options.ClientId = "mi-tienda-online";
    options.ResponseType = "code";
    options.Scope.Add("email");
});
```

## Lo que NO hace

- **No sustituye tu propio sistema de permisos** — OIDC te dice quién acaba de entrar, pero qué puede hacer esa usuaria dentro de tu aplicación sigue siendo cosa de tu propio [RBAC](RBAC-y-Claims.md); el proveedor de identidad no conoce tus roles de negocio.
- **No es un protocolo alternativo a OAuth2** — es una extensión: todo flujo de OIDC es, por debajo, un flujo de OAuth2 con un scope especial (`openid`) y un token adicional.
- **No elimina la necesidad de gestionar tu propia sesión** — tras validar el `id_token` en el login, la mayoría de aplicaciones emiten su propia cookie o token de sesión interno; no reenvían el `id_token` original en cada request posterior.

## Buenas prácticas avanzadas

- **No uses el `access_token` para identificar a la usuaria en tu backend** — su formato y contenido son responsabilidad del Resource Server para el que se emitió, no están garantizados como legibles ni estables; usa siempre el `id_token` para la identidad, y trátalo solo como evidencia del login, no como credencial para llamadas posteriores.
- **Valida el `nonce`** — igual que `state` protege el flujo de OAuth2 frente a CSRF, el `nonce` (un valor aleatorio que el Client genera antes de redirigir y que debe reaparecer dentro del `id_token`) protege el login frente a ataques de repetición con tokens robados de otra sesión.
- **No confíes en ningún claim sin validar firma, `iss` y `aud` primero** — un `id_token` con `email_verified: true` solo es fiable si antes comprobaste que la firma es válida, que lo emitió el proveedor que esperabas y que iba dirigido a tu aplicación y no a otra.
- **Separa el momento del login del resto de la sesión** — usa el `id_token` únicamente para establecer quién es la usuaria en el instante del login; no lo reenvíes a tus APIs internas más adelante, donde corresponde tu propio token de sesión o access_token con los scopes de tu dominio.
- **Cachea el documento de descubrimiento y las claves públicas, pero respeta su expiración** — la mayoría de librerías OIDC lo hacen automáticamente; si implementas la validación a mano, refresca las claves cuando el proveedor las rote (identificadas por `kid`) en vez de descargarlas en cada request.

---

*En resumen: OpenID Connect añade a OAuth2 la pieza que le faltaba —un `id_token` firmado que certifica la identidad de la usuaria— convirtiendo un protocolo de autorización en la base estándar del "iniciar sesión con" de internet.*
