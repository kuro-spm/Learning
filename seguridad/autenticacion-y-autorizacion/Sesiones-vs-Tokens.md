# Sesiones vs Tokens

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Dos estrategias distintas para resolver el mismo problema: recordar que una usuaria ya se autenticó, request tras request. Las **sesiones** guardan el estado de login en el servidor y entregan al navegador solo un identificador opaco (normalmente en una cookie). Los **tokens** hacen lo contrario: empaquetan la información de la identidad en un paquete autocontenido que viaja con cada request y el servidor no necesita recordar nada.

## ¿Por qué existe?

HTTP no tiene memoria: cada request es independiente y el servidor, por defecto, no sabe si dos peticiones vienen de la misma persona. Alguna pieza tiene que "recordar" que ya hiciste login.

El modelo de **sesiones** resuelve esto guardando ese recuerdo en el servidor (en memoria, en base de datos o en Redis) y dándole al cliente solo una llave —el ID de sesión— para recuperarlo. Es el modelo clásico de las aplicaciones web con formularios y cookies.

El modelo de **tokens** nació con la necesidad de servir a clientes que no son un navegador con cookies: apps móviles, SPAs que hablan con una API, o servicios que llaman a otros servicios. En vez de una llave que apunta a un dato guardado en el servidor, el token *es* el dato, firmado para que no se pueda falsificar.

> Si te suena la diferencia entre `IMemoryCache` y una cadena firmada con HMAC que llevas tú misma en el bolsillo: una sesión es el caché en el servidor; un token es el dato viajando contigo, verificable sin preguntarle nada a nadie.

## ¿Cuándo y para qué se usa?

- **Sesiones**: encajan de forma natural en aplicaciones monolíticas renderizadas en servidor (un panel de administración con Razor Pages, un backoffice clásico) donde el navegador y el servidor mantienen una conversación continua con cookies.
- **Tokens**: encajan en arquitecturas donde el cliente y el servidor están más desacoplados: una SPA en React que consume una API REST, una app móvil de una tienda online, o comunicación entre microservicios (el servicio de pedidos llama al de facturación llevando la identidad de la usuaria original en un token).

## Lo mínimo que necesitas saber

**1. Sesión con cookie en ASP.NET Core**

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.ExpireTimeSpan = TimeSpan.FromMinutes(30);
        options.SlidingExpiration = true;
    });

// Al hacer login, el servidor crea la sesión y manda una cookie con el ID:
await HttpContext.SignInAsync(principal);
```

El servidor guarda internamente "el ID de sesión X pertenece a la usuaria Ana"; la cookie solo lleva ese ID.

**2. Token en el header `Authorization`**

```csharp
// El cliente envía el token en cada request, no una cookie:
// Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
        };
    });
```

El servidor no guarda nada: valida la firma y los datos que el propio token trae consigo (ver la ficha de [JWT](JWT.md)).

**3. Revocar el acceso antes de que expire**

Con sesiones, revocar es trivial: borras la fila o la entrada de caché en el servidor, y la próxima request con esa cookie falla al instante. Con tokens autocontenidos, el servidor no tiene ningún registro que borrar — el token sigue siendo válido hasta que expira por sí mismo, a menos que implementes una lista de revocación explícita.

**4. Escalar a varios servidores**

Una sesión en memoria de un servidor no es visible desde otro: en un despliegue con varias instancias necesitas *sticky sessions* (que la usuaria siempre vaya al mismo servidor) o un almacén compartido (Redis, base de datos). Un token autocontenido se valida igual en cualquier instancia sin coordinación adicional, porque toda la información necesaria viaja en él.

## Lo que NO hace

- **Un token no es automáticamente más seguro que una sesión** — ambos pueden robarse (XSS, red comprometida); la seguridad depende de cómo se transmiten y almacenan, no del modelo elegido.
- **Una sesión no es "anticuada"** — para aplicaciones monolíticas con renderizado en servidor sigue siendo la opción más simple y con revocación instantánea gratis.
- **Un token no es "stateless de verdad" en cuanto necesitas revocación real** — en el momento en que añades una lista de revocación o un refresh token guardado en base de datos, has vuelto a tener estado en el servidor; solo lo has movido de sitio.

## Buenas prácticas avanzadas

- **Configura las cookies de sesión con `HttpOnly`, `Secure` y `SameSite`** — `HttpOnly` impide que JavaScript lea la cookie (mitiga robo por XSS), `Secure` obliga a HTTPS y `SameSite=Lax` o `Strict` reduce el riesgo de CSRF. Omitir cualquiera de las tres en producción es un descuido común y grave.
- **Los tokens de acceso deben durar minutos, no días** — cuanto más corta la vida de un token que no puedes revocar, menor la ventana de exposición si se filtra. Combínalo con un *refresh token* de vida más larga, guardado de forma más protegida, para renovar el acceso sin pedir credenciales de nuevo.
- **No creas que JWT resuelve la revocación "gratis"** — si tu aplicación necesita poder cerrar sesión de forma inmediata (por ejemplo, tras detectar un robo de cuenta), necesitas una lista de revocación o tokens de vida muy corta; el diseño "puramente stateless" y la revocación instantánea son objetivos en tensión.
- **Rota el ID de sesión al autenticar** — genera un ID de sesión nuevo justo después del login, no reutilices el que tenía la usuaria como anónima. Evita la *session fixation*, donde un atacante planta un ID de sesión antes de que la víctima inicie sesión y luego lo reutiliza ya autenticado.
- **Decide el almacén de sesión pensando en el volumen, no por defecto** — sesiones en memoria del proceso se pierden en cada despliegue y no escalan a varias instancias; un almacén distribuido (Redis) añade una dependencia de red pero resuelve ambos problemas de raíz.

---

*En resumen: la sesión guarda el estado en el servidor y entrega solo una llave; el token lleva el propio estado firmado consigo — elige según si tu arquitectura necesita revocación instantánea (sesión) o desacoplamiento entre servicios (token).*
