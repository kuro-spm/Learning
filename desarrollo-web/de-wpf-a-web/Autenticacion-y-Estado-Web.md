# Autenticación y estado web

**¿Qué es?** Es la forma en que una aplicación web **sabe quién eres** (autenticación) y
**recuerda algo entre una petición y otra** (estado), a pesar de que [HTTP no tiene
memoria](HTTP.md). Como cada petición llega "en blanco", hay que llevar contigo, en cada
una, la prueba de quién eres.

---

## ¿Por qué existe?

En WPF esto era invisible: la app conocía a la usuaria de Windows que la había abierto, y
el estado vivía en memoria mientras la ventana estuviera abierta. En la web no hay nada
de eso: el servidor atiende a miles de clientes y olvida cada petición en cuanto
responde. Para que "iniciar sesión" signifique algo en las siguientes pantallas, hace
falta un mecanismo explícito que viaje en cada petición.

> Si vienes de WPF: olvídate del "usuario actual" como un dato global y permanente. En
> web, la identidad es algo que el cliente **adjunta a cada petición**, no un estado que
> el servidor mantiene abierto por ti.

---

## ¿Cuándo y para qué se usa?

En cualquier app con login: una tienda online que recuerda tu carrito y tus pedidos, un
panel de gestión donde solo entra el personal autorizado, una API que distingue qué
puede ver cada cliente. Aparece en cuanto algo deja de ser público.

---

## Lo mínimo que necesitas saber

**1. Autenticación ≠ autorización**

- **Autenticación**: comprobar *quién eres* (login).
- **Autorización**: comprobar *qué te está permitido* hacer una vez identificada.

ASP.NET Core las gestiona como dos pasos seguidos en el [middleware](Routing-y-Middleware.md):
primero `UseAuthentication`, luego `UseAuthorization`.

**2. Hay dos enfoques típicos según el tipo de app**

- **Cookies** (apps con páginas, como [Razor/MVC](Razor-Pages-y-MVC.md)): al hacer login,
  el servidor entrega una cookie que el navegador reenvía sola en cada petición.
- **Tokens** (APIs y SPAs, como una [Web API](Web-API-y-REST.md)): el cliente recibe un
  token (a menudo un *JWT*) y lo manda en la cabecera `Authorization` de cada petición.

**3. La identidad viaja en cada petición, no se "guarda" en el servidor**

Tanto la cookie como el token acompañan a *todas* las peticiones siguientes. El servidor
los lee, verifica que son válidos y así sabe quién pide cada cosa, sin recordar nada entre
medias.

```text
1. POST /login (usuario + contraseña)  → el servidor responde con una cookie o un token
2. GET /api/pedidos  (+ cookie/token)  → el servidor lee la prueba y sabe quién eres
3. GET /api/perfil   (+ cookie/token)  → otra vez: la identidad va adjunta de nuevo
```

**4. Proteger algo es declarar que requiere identidad**

Marcas qué partes exigen estar autenticada (o tener cierto rol) y el framework rechaza
con un `401`/`403` a quien no cumpla:

```csharp
[Authorize] // solo usuarias autenticadas
public class PedidosController : ControllerBase { /* ... */ }

[Authorize(Roles = "Admin")] // además, solo administradoras
public IActionResult Borrar(int id) { /* ... */ }
```

**5. Para estado pequeño y temporal existe la sesión; para lo importante, la base de datos**

Cosas efímeras (un carrito sin login) pueden guardarse en la *sesión* del servidor. Pero
lo que debe perdurar —pedidos, perfiles, configuración— va a la base de datos, no en
memoria del proceso, porque el servidor puede reiniciarse o haber varias copias atendiendo.

---

## Lo que NO hace

- **No mantiene una sesión "abierta" como en escritorio** — la identidad se reenvía en cada petición; no es un estado vivo en el servidor.
- **No protege nada que no declares** — si no marcas un endpoint, queda abierto a cualquiera.
- **No sustituye a HTTPS** — cookies y tokens viajan por la red; sin HTTPS, podrían ser interceptados.

---

## Buenas prácticas avanzadas

- **Un JWT está firmado, no cifrado: cualquiera puede leerlo** — el contenido de un JWT es solo texto codificado en Base64; la firma garantiza que nadie lo ha *modificado*, no que nadie lo pueda *leer*. Pégalo en cualquier decodificador y verás su interior. Por eso jamás se meten datos sensibles en él (contraseñas, datos personales de más): solo la identidad y los permisos mínimos.
- **Dónde guardas el token importa tanto como el token** — un JWT en `localStorage` es legible por cualquier JavaScript de la página: un solo script comprometido (un ataque XSS) y la identidad está robada. Una cookie con `HttpOnly` (invisible para JavaScript), `Secure` (solo por HTTPS) y `SameSite` (que frena el CSRF) es la opción más defendible para apps de navegador; el token "a mano" queda para clientes que no son un navegador.
- **Los JWT no se pueden "cerrar sesión": planifica la expiración** — una cookie de sesión se puede invalidar en el servidor; un JWT firmado es válido hasta que caduca, lo revoques o no. Si emites tokens de 30 días y hay que expulsar a alguien, no puedes. El patrón profesional: tokens de acceso cortos (minutos) más un *refresh token* de larga vida que sí se puede revocar en el servidor.

---

*En resumen: como HTTP no recuerda nada, en web la identidad viaja adjunta a cada petición —por cookie o por token— y el estado importante vive en la base de datos, no en la memoria del servidor como hacías en WPF.*
