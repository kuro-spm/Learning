# JWT (JSON Web Token)

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Un JWT (*JSON Web Token*) es un formato estándar para representar un conjunto de afirmaciones (*claims*) sobre una identidad —quién es, qué puede hacer, hasta cuándo es válido— de forma compacta y verificable mediante una firma digital. Es el formato de token más usado hoy para autenticación y autorización en APIs.

## ¿Por qué existe?

Cuando adoptas un modelo de tokens (ver [Sesiones vs Tokens](Sesiones-vs-Tokens.md)) necesitas un formato concreto para ese token: algo que cualquier servicio pueda leer y, sobre todo, verificar que no ha sido manipulado, sin tener que preguntarle a un servidor central "¿este token es válido?" en cada request. JWT resuelve esto firmando digitalmente el contenido: cualquiera con la clave pública (o el secreto compartido) puede comprobar que los datos no se han tocado desde que se emitieron, sin necesidad de una base de datos de sesiones.

> Piensa en un JWT como un sobre de cristal lacrado: cualquiera puede leer lo que hay dentro sin abrirlo (no está cifrado), pero el lacre (la firma) demuestra que nadie lo ha abierto y vuelto a cerrar con contenido distinto.

## ¿Cuándo y para qué se usa?

- **APIs stateless**: una API REST que valida cada request comprobando la firma del token, sin consultar una tabla de sesiones.
- **Microservicios**: el servicio de pedidos de una tienda online recibe un JWT del API Gateway y confía en sus claims (id de usuaria, rol) sin tener que volver a autenticarla contra el servicio de identidad.
- **OAuth2 y OpenID Connect**: los *access tokens* y *id_tokens* que emiten estos protocolos suelen tener formato JWT (ver las fichas de [OAuth2](OAuth2.md) y [OpenID Connect](OpenID-Connect.md)).

## Lo mínimo que necesitas saber

**1. Las tres partes de un JWT**

Un JWT es tres bloques en Base64Url separados por puntos: `header.payload.signature`.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsIm5hbWUiOiJBbmEiLCJleHAiOjE3MjAwMDAwMDB9.dQw4w9WgXcQ...
└────────── header ─────────┘ └──────────────── payload ────────────────┘ └── signature ──┘
```

El *header* indica el algoritmo de firma; el *payload* contiene los claims; la *signature* se calcula sobre las dos partes anteriores usando una clave que solo el emisor conoce.

**2. Claims estándar del payload**

```json
{
  "iss": "https://miapp.identidad.com",  // issuer: quién lo emitió
  "sub": "42",                            // subject: a quién identifica
  "aud": "api-pedidos",                   // audience: para quién es
  "exp": 1720003600,                      // expiration: hasta cuándo vale (timestamp Unix)
  "iat": 1720000000                       // issued at: cuándo se emitió
}
```

**3. Firma simétrica (HS256) vs asimétrica (RS256)**

Con `HS256`, emisor y verificador comparten el mismo secreto: sencillo, pero cualquiera que pueda verificar también podría falsificar. Con `RS256`, el emisor firma con una clave privada y cualquiera verifica con la clave pública correspondiente, sin poder firmar tokens nuevos: es la opción habitual cuando el emisor (servidor de identidad) y los verificadores (varias APIs) son componentes distintos.

**4. Generar y validar en C#**

```csharp
// Generar
var claims = new[] { new Claim(JwtRegisteredClaimNames.Sub, "42") };
var creds = new SigningCredentials(clave, SecurityAlgorithms.HmacSha256);
var token = new JwtSecurityToken(issuer: "miapp", audience: "api-pedidos",
    claims: claims, expires: DateTime.UtcNow.AddMinutes(15), signingCredentials: creds);
string jwt = new JwtSecurityTokenHandler().WriteToken(token);

// Validar
var parametros = new TokenValidationParameters
{
    ValidIssuer = "miapp",
    ValidAudience = "api-pedidos",
    IssuerSigningKey = clave,
    ValidateLifetime = true
};
var principal = new JwtSecurityTokenHandler().ValidateToken(jwt, parametros, out _);
```

**5. Expiración y por qué no se "cierra sesión" de un JWT**

El claim `exp` marca cuándo deja de ser válido, y esa comprobación la hace cada verificador de forma local. Como el token no depende de un registro en el servidor, no hay un "borrar" que lo invalide antes de tiempo: solo dejará de aceptarse cuando expire o si implementas explícitamente una lista de revocación.

## Lo que NO hace

- **No cifra el contenido** — el payload es Base64Url, no cifrado: cualquiera puede decodificarlo y leerlo. La firma protege la integridad, no la confidencialidad. Nunca metas contraseñas o datos sensibles en los claims.
- **No es una sesión** — no hay estado en el servidor; es responsabilidad de quien lo verifica decidir en cada request si el token es válido y suficiente.
- **No se revoca por sí mismo** — mientras no expire y la firma sea correcta, un JWT sigue siendo válido aunque la cuenta se haya bloqueado mientras tanto, salvo que existan comprobaciones adicionales.

## Buenas prácticas avanzadas

- **Valida siempre el algoritmo esperado, nunca confíes en el `alg` del header** — el ataque clásico consiste en reescribir el header a `alg: none` o a un algoritmo débil y ver si el verificador lo acepta sin más. Configura la librería para exigir un algoritmo concreto (`ValidAlgorithms`), no el que el token diga traer.
- **Cuidado con la confusión de algoritmos entre RS256 y HS256** — si un verificador usa la clave pública RS256 como si fuera un secreto HMAC, un atacante que conozca esa clave pública (son públicas por diseño) puede forjar tokens HS256 válidos. Las librerías modernas mitigan esto, pero es un motivo más para fijar explícitamente el algoritmo permitido.
- **Usa tokens de vida corta y un `refresh token` aparte** — cuanto más dura un `access_token`, mayor la ventana de uso indebido si se filtra. Unos minutos u horas de vida, combinados con un refresh token de un solo uso y rotación, acotan el daño sin obligar a la usuaria a volver a loguearse constantemente.
- **Valida `iss` y `aud`, no solo la firma** — una firma válida solo demuestra quién lo emitió; sin comprobar la audiencia, un token válido para otra API distinta podría colarse como válido en la tuya si comparten emisor.
- **Prepara la rotación de claves con `kid` y un endpoint JWKS** — identificar la clave usada en cada token (el campo `kid` del header) permite rotar claves de firma sin invalidar de golpe todos los tokens en circulación ni requerir un despliegue coordinado.

## Recursos didácticos

[jwt.io](https://jwt.io) tiene un decodificador interactivo: pega cualquier JWT y verás sus tres partes separadas, decodificadas y coloreadas, además de poder verificar la firma en el propio navegador.

---

*En resumen: un JWT es un conjunto de claims firmado digitalmente que cualquiera puede leer pero nadie puede falsificar sin la clave — perfecto para verificar identidad sin depender de una sesión en el servidor.*
