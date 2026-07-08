# Modelo JWT + Refresh Token

## ¿Qué es?

Es el patrón más extendido para mantener viva una sesión basada en tokens sin obligar a la usuaria a volver a loguearse cada pocos minutos. Combina dos piezas con papeles distintos: un **access token** de vida corta (normalmente un [JWT](JWT.md)) que autoriza cada request, y un **refresh token** de vida larga cuyo único trabajo es conseguir access tokens nuevos cuando el anterior caduca.

## ¿Por qué existe?

Un JWT tiene una debilidad de raíz: una vez emitido, es válido hasta que expira, y el servidor no tiene forma sencilla de "apagarlo" antes (ver [JWT](JWT.md) → *Lo que NO hace*). La defensa natural es darle una vida muy corta —minutos— para que, si se filtra, la ventana de uso indebido sea pequeña. Pero un token que caduca cada 15 minutos significaría pedir usuario y contraseña cada 15 minutos: inaceptable.

El refresh token resuelve esa tensión repartiendo el trabajo. El access token es de usar y tirar, viaja en cada request y caduca enseguida. El refresh token se usa rara vez (solo para renovar), se guarda de forma más protegida y sí se registra en el servidor, así que **sí se puede revocar**. Con eso obtienes lo mejor de dos mundos: requests rápidas y sin estado, pero con un punto de control para cortar el acceso.

> Piensa en el access token como la tarjeta magnética de un hotel, que caduca cada pocas horas, y en el refresh token como tu DNI en recepción: no lo enseñas para abrir cada puerta, solo para que te regeneren la tarjeta cuando expira, y recepción puede negarse si te han dado de baja.

## ¿Cuándo y para qué se usa?

- **SPAs que consumen una API**: una aplicación React que guarda el access token en memoria y renueva en segundo plano sin que la usuaria note nada.
- **Apps móviles**: una app de una tienda online que mantiene la sesión durante semanas sin volver a pedir contraseña, apoyándose en un refresh token guardado en el almacén seguro del dispositivo.
- **OAuth2**: el flujo *Authorization Code* devuelve exactamente este par (access + refresh); es el modelo de "iniciar sesión con..." que verás en la ficha de [OAuth2](OAuth2.md).

## Lo mínimo que necesitas saber

**1. Dos tokens, dos papeles**

| | Access token | Refresh token |
|---|---|---|
| Formato | JWT autocontenido | Cadena opaca (aleatoria) |
| Vida | Minutos | Días o semanas |
| Viaja en | Cada request (`Authorization`) | Solo al renovar |
| ¿Revocable? | No (hasta que expira) | Sí (registrado en el servidor) |

**2. El ciclo de vida**

Al hacer login, el servidor entrega ambos. A partir de ahí, el cliente usa el access token hasta que una request devuelve `401` porque ha caducado; entonces llama al endpoint de refresh, obtiene un access token nuevo y reintenta.

```ts
// Interceptor de cliente: renueva de forma transparente ante un 401
async function fetchConAuth(url: string) {
  let res = await fetch(url, { headers: { Authorization: `Bearer ${accessToken}` } });
  if (res.status === 401) {
    accessToken = await renovarAccessToken();      // llama a /auth/refresh
    res = await fetch(url, { headers: { Authorization: `Bearer ${accessToken}` } });
  }
  return res;
}
```

**3. El endpoint de renovación**

```csharp
[HttpPost("/auth/refresh")]
public async Task<IActionResult> Refresh(string refreshToken)
{
    var guardado = await _store.BuscarAsync(Hash(refreshToken));
    if (guardado is null || guardado.Expirado || guardado.Revocado)
        return Unauthorized();

    // Rotación: invalida el refresh usado y emite uno nuevo
    await _store.RevocarAsync(guardado.Id);
    var (nuevoAccess, nuevoRefresh) = EmitirParDeTokens(guardado.UsuarioId);
    return Ok(new { nuevoAccess, nuevoRefresh });
}
```

**4. Rotación y detección de reuso**

La práctica moderna es que cada refresh token sea **de un solo uso**: al canjearlo, se invalida y se entrega uno nuevo (*rotación*). Si alguien intenta volver a usar un refresh token ya canjeado, es señal de robo: se revoca toda la *familia* de tokens de esa sesión y se fuerza un login nuevo.

**5. Dónde guardar cada token**

El refresh token es el activo valioso (dura mucho y renueva el acceso), así que va donde JavaScript no pueda leerlo: una cookie `HttpOnly` + `Secure` + `SameSite`, o el almacén seguro del sistema en móvil. El access token, al ser efímero, suele vivir solo en memoria del cliente. **Guardar el refresh token en `localStorage` es un error común**: queda expuesto a cualquier XSS.

## Lo que NO hace

- **No convierte los JWT en revocables** — el access token sigue siendo válido hasta que expira; lo único revocable es el refresh. Si necesitas cortar el acceso al instante, la vida corta del access token es tu única garantía real.
- **No elimina el estado del servidor** — en cuanto guardas refresh tokens para poder revocarlos, ya tienes estado que mantener; el modelo "puramente stateless" desaparece (ver [Sesiones vs Tokens](Sesiones-vs-Tokens.md)).
- **No protege por sí solo frente a XSS** — si un atacante ejecuta JavaScript en tu página, puede usar el access token en memoria mientras la usuaria esté activa; el modelo acota el daño, no lo impide.

## Buenas prácticas avanzadas

- **Rotación con detección de reuso mediante familias de tokens** — asocia todos los refresh tokens de una misma sesión a un identificador de familia. Cuando detectes que se reusa uno ya canjeado, revoca la familia entera, no solo ese token: es la señal de que alguien tiene una copia robada, y así expulsas también al ladrón.
- **Combina expiración deslizante con una absoluta** — que el refresh se renueve con el uso está bien, pero fija además un tope absoluto (p. ej. 30 días desde el login) tras el cual toca reautenticarse sí o sí. Sin ese tope, una sesión activa se perpetúa indefinidamente y nunca fuerza una revalidación de credenciales.
- **Guarda solo el hash del refresh token, no el token en claro** — trátalo como una contraseña: si te roban la base de datos de sesiones, un hash no es utilizable, el token en claro sí. Basta un hash rápido (SHA-256) porque el token ya tiene entropía alta; no necesita bcrypt.
- **Vincula el refresh a un contexto y valida `iss`/`aud` del access** — atar el refresh a la sesión (device, IP aproximada) permite detectar canjes desde ubicaciones imposibles. Y no olvides validar audiencia y emisor del access token, no solo la firma (ver [JWT](JWT.md) → *Buenas prácticas*).
- **No pongas el refresh token en el cuerpo si puedes evitarlo** — entregarlo en una cookie `HttpOnly` con path restringido al endpoint de refresh reduce su superficie de exposición frente a devolverlo en JSON que el JavaScript tenga que manejar y guardar.

## Recursos didácticos divertidos

El [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics) del IETF dedica una sección entera a la rotación de refresh tokens y la detección de reuso; y en [jwt.io](https://jwt.io) puedes decodificar el access token para ver su `exp` corto en acción.

---

*En resumen: el modelo JWT + Refresh reparte el trabajo entre un access token efímero e irrevocable y un refresh token duradero y revocable, logrando sesiones cómodas sin renunciar del todo al control sobre el acceso.*
