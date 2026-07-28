# OAuth2

## ¿Qué es?

OAuth2 es un estándar de **autorización delegada**: permite que una aplicación acceda a datos de una persona que están alojados en otro servicio —sus pedidos, sus fotos, su calendario— sin que esa persona tenga que entregarle su usuario y contraseña de ese otro servicio.

## ¿Por qué existe?

Antes de OAuth2 había una única forma de que una aplicación externa leyera tus datos de otro sitio: que le dieras tu usuario y tu contraseña de ese otro sitio, y que ella los guardara. El resultado era malo por tres motivos a la vez. La aplicación externa obtenía acceso **total** a la cuenta, no solo a la parte que necesitaba. Ese acceso era **indefinido**, porque una contraseña no caduca. Y era **irrevocable en la práctica**: para cortarle el paso a una sola aplicación había que cambiar la contraseña, lo que rompía todas las demás a la vez.

OAuth2 rompe ese acoplamiento. La persona nunca escribe su contraseña en la aplicación externa: se autentica directamente en el servicio dueño de los datos, y es ese servicio el que entrega a la aplicación un **token** con permisos limitados, con caducidad y revocable de forma individual.

> Piensa en OAuth2 como la tarjeta magnética de un hotel: no le das al botones la llave de tu casa; el hotel le da una tarjeta que abre solo tu habitación, solo durante tu estancia, y que pueden desactivar en cualquier momento sin que tú cambies nada.

## ¿Cuándo y para qué se usa?

- **Conceder acceso a un servicio de terceros.** Es el caso original y el que recorre esta ficha: una aplicación externa quiere leer datos que están en tu API, en nombre de la persona dueña de esos datos.
- **Comunicación máquina a máquina.** Un servicio que llama a la API de otro con credenciales propias, sin que haya nadie delante en ese momento. Es el flujo *Client Credentials*, el más habitual en backend.
- **«Iniciar sesión con Google/GitHub/Microsoft».** Aquí OAuth2 solo pone la mitad de abajo: el protocolo que de verdad transporta la identidad es [OpenID Connect](OpenID-Connect.md), construido encima. Volveremos a esto, porque confundir las dos cosas es el error de diseño más común.

---

## El escenario de esta ficha

Toda la ficha usa el mismo caso, y conviene fijarlo antes de ver una sola petición.

Una tienda online tiene su API en `api.tienda.ejemplo.com` y su frontend en `tienda.ejemplo.com`. Una aplicación externa de gestión de envíos, `envios.ejemplo.com`, quiere **leer los pedidos** de una clienta concreta —la clienta **42**, con el pedido **#4711** entre ellos— para calcular etiquetas de envío. La clienta quiere que pueda hacerlo, pero no quiere darle su contraseña de la tienda.

Con eso, los cuatro roles del protocolo dejan de ser abstractos.

## Los cuatro roles

| Rol | Quién es aquí | Qué guarda | Qué nunca ve |
|---|---|---|---|
| **Resource Owner** | La clienta 42, dueña de sus pedidos | Su contraseña de la tienda | El `client_secret` de la app de envíos |
| **Client** | `envios.ejemplo.com`, la app externa | Su `client_id`, su `client_secret` y los tokens que recibe | **La contraseña de la clienta** |
| **Authorization Server** | El servidor de identidad de la tienda, `auth.tienda.ejemplo.com` | Las credenciales de la clienta y el registro de clientes | Los datos de negocio (no necesita leer pedidos) |
| **Resource Server** | `api.tienda.ejemplo.com`, la API de la tienda | Los pedidos, incluido el #4711 | La contraseña de la clienta y el `client_secret` |

La fila que hay que memorizar es la segunda: **la contraseña de la clienta solo se escribe en el Authorization Server, y el Client nunca la ve**. Todo el diseño del protocolo existe para sostener esa frase. Si en algún momento tu implementación hace que el Client reciba la contraseña —aunque sea «solo para reenviarla»—, has salido de OAuth2 y has vuelto al problema que OAuth2 resuelve.

En un montaje pequeño, el Authorization Server y el Resource Server pueden ser el mismo servicio. Siguen siendo dos roles distintos: uno emite tokens, el otro los acepta y los valida.

## El flujo Authorization Code, petición a petición

Este es el flujo estándar cuando hay una persona delante. Son cuatro intercambios y merece la pena verlos literales, porque los detalles importantes están en los parámetros.

### Paso 1 — El Client redirige al Authorization Server

La app de envíos no llama a ninguna API todavía: manda el navegador de la clienta al Authorization Server con una URL construida por ella.

```http
GET https://auth.tienda.ejemplo.com/authorize
  ?response_type=code
  &client_id=envios-ejemplo-com
  &redirect_uri=https%3A%2F%2Fenvios.ejemplo.com%2Fcallback
  &scope=pedidos.read
  &state=9f2a41c8e7b04d15
  &code_challenge=Cc1iJyzKPO0ZsRzv2uAqPnJYzowFLIdWxhUJJEbH1D4
  &code_challenge_method=S256
```

Parámetro a parámetro:

- `response_type=code` — «devuélveme un código de autorización», no un token. Es lo que selecciona este flujo.
- `client_id` — identifica públicamente al Client. No es un secreto; viaja en una URL que la clienta puede leer.
- `redirect_uri` — a dónde volver con el resultado. Tiene que estar registrada de antemano en el Authorization Server, y la comparación es exacta (sección propia más abajo).
- `scope=pedidos.read` — el permiso que se pide. Uno solo, el mínimo necesario.
- `state` — valor aleatorio de un solo uso que el Client guarda en la sesión y comprobará al volver. Sección propia más abajo.
- `code_challenge` y `code_challenge_method` — PKCE. Sección propia más abajo.

La clienta ve la pantalla de login **de la tienda**, en el dominio de la tienda, escribe ahí su contraseña y después una pantalla de consentimiento del estilo «*envios.ejemplo.com quiere leer tus pedidos*» con Permitir / Denegar.

### Paso 2 — La vuelta al Client con el código

Si acepta, el Authorization Server redirige el navegador a la `redirect_uri` registrada:

```http
HTTP/1.1 302 Found
Location: https://envios.ejemplo.com/callback
  ?code=aZk3Q1p8Rn7XsT2v
  &state=9f2a41c8e7b04d15
```

El `code` es de **un solo uso** y de vida muy corta (el estándar recomienda 10 minutos o menos; en la práctica suelen ser 30-60 segundos). Todavía no sirve para llamar a ninguna API. Lo primero que hace el Client al recibir esto es comprobar que el `state` es exactamente el que guardó.

### Paso 3 — El canje del código por el token

Ahora el backend de la app de envíos, **desde su servidor y no desde el navegador**, canjea el código:

```http
POST /token HTTP/1.1
Host: auth.tienda.ejemplo.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=aZk3Q1p8Rn7XsT2v
&redirect_uri=https%3A%2F%2Fenvios.ejemplo.com%2Fcallback
&client_id=envios-ejemplo-com
&client_secret=s3cr3t-de-la-app-de-envios
&code_verifier=M9k2vQ7pLxT4nR8sJ1yB6cF0hW3dZaE5gU2iO7qXlKt
```

Y la respuesta, si todo cuadra:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjA0Y2E...",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "def50200a1b9c4e8f7d3...",
  "scope": "pedidos.read"
}
```

Cuatro lecturas de esa respuesta: el `access_token` es la credencial para llamar a la API; `token_type: Bearer` significa «quien lo lleve, lo usa» (de ahí que haya que protegerlo); `expires_in: 900` son 15 segundos por cada minuto de vida, es decir 15 minutos; y el `refresh_token` sirve para conseguir un `access_token` nuevo cuando el primero caduque.

Ese `access_token` **puede** ser un JWT o puede ser una cadena opaca sin significado. OAuth2 no lo especifica: es una decisión del Authorization Server, y quien la consume es el Resource Server. El formato, la firma y la validación los cubre [JWT](JWT.md).

### Paso 4 — La llamada al Resource Server

```http
GET /pedidos/4711 HTTP/1.1
Host: api.tienda.ejemplo.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjA0Y2E...
```

La API valida el token, comprueba que trae `scope=pedidos.read`, comprueba que **el pedido #4711 pertenece a la clienta 42** y devuelve el pedido. Ese segundo control no lo hace OAuth2; lo hace tu código. Es lo bastante importante para tener su propia sección.

## ¿Por qué un código intermedio y no el token directamente?

Es la pregunta que se hace todo el mundo la primera vez: si el Authorization Server ya sabe que la clienta ha aceptado, ¿por qué no devuelve el `access_token` en la redirección del paso 2 y nos ahorramos el paso 3?

Porque **la redirección viaja por el navegador**, y todo lo que viaja por el navegador deja rastro:

| Dónde acaba | El `code` (un solo uso, 60 s, inútil sin PKCE) | El `access_token` (15 min, sirve para todo) |
|---|---|---|
| Historial del navegador | Sí | Sí |
| Log de acceso del servidor que recibe la redirección | Sí | Sí |
| Cabecera `Referer` hacia el siguiente sitio visitado | Sí | Sí |
| Extensiones del navegador, proxies corporativos | Sí | Sí |

Las filas son idénticas: la diferencia no está en el canal, está en **lo que se filtra**. Un `code` filtrado es casi inservible (ya se usó, o caduca en un minuto, o falta el `code_verifier`). Un `access_token` filtrado es acceso funcional a los pedidos durante los siguientes 15 minutos.

El paso 3 mueve la credencial de verdad a un canal donde el navegador no participa: una petición de servidor a servidor, sobre TLS, sin URL que quede en ningún historial.

Esto es exactamente lo que hacía mal el flujo **Implicit** (`response_type=token`): devolvía el `access_token` en el fragmento de la URL de redirección. Funcionaba, y su motivo original era razonable —en 2012 una SPA no podía hacer peticiones a otro dominio, porque CORS aún no estaba desplegado en todas partes—. Pero el motivo desapareció y el problema se quedó, así que OAuth 2.1 lo retira.

## PKCE: el código interceptado que ya no sirve

*Proof Key for Code Exchange* (se pronuncia «pixy») cierra el hueco que queda: **si alguien intercepta el `code` del paso 2, ¿puede canjearlo?**

Sin PKCE, en un cliente público —una SPA, una app móvil— la respuesta es sí. Ese tipo de cliente no puede guardar un `client_secret` de verdad (está en el código que se descarga o en el binario de la app), así que el paso 3 no exige nada que el atacante no pueda tener. Con el `code` en la mano, lo canjea antes que el cliente legítimo y se lleva el token.

PKCE arregla eso con un secreto de un solo uso que **se inventa el propio cliente en cada flujo**:

1. Antes del paso 1, el Client genera una cadena aleatoria de 43 a 128 caracteres, el **`code_verifier`**, y la guarda solo en su lado:

```text
code_verifier = M9k2vQ7pLxT4nR8sJ1yB6cF0hW3dZaE5gU2iO7qXlKt
```

2. Calcula su SHA-256 y lo codifica en **base64url sin relleno**. El resultado es el **`code_challenge`**:

```text
code_challenge = base64url( SHA-256( code_verifier ) )
               = Cc1iJyzKPO0ZsRzv2uAqPnJYzowFLIdWxhUJJEbH1D4
```

3. Manda el `code_challenge` (no el verifier) en la URL de autorización, con `code_challenge_method=S256`.
4. En el paso 3, manda el **`code_verifier`** en el `POST /token`.

El Authorization Server recalcula el SHA-256 del `code_verifier` recibido y lo compara con el `code_challenge` que guardó al emitir el código. Si no coincide, rechaza el canje:

```json
{
  "error": "invalid_grant",
  "error_description": "PKCE verification failed"
}
```

La clave está en la dirección de la función hash. El atacante que intercepta la redirección ve el `code` y, si le llega la URL de autorización, el `code_challenge`; pero de un SHA-256 no se vuelve atrás. Sin el `code_verifier` original, el canje falla. **El `code` deja de ser suficiente: hace falta el `code` y la prueba de haber iniciado ese mismo flujo.**

Un detalle que se pasa por alto: `code_challenge_method` admite también `plain`, que manda el verifier tal cual. No lo uses — anula el mecanismo entero, porque quien intercepta la primera petición ya tiene el secreto. Usa siempre `S256`.

**¿Y si el cliente sí puede guardar un secreto, como el backend de la app de envíos?** OAuth 2.1 recomienda PKCE también ahí, y el motivo no es el `client_secret` sino la **inyección de código de autorización**: un atacante consigue un `code` legítimo de su propia cuenta y lo inyecta en el callback de la víctima, de modo que la sesión de la víctima en la app de envíos queda ligada a la cuenta del atacante. El `client_secret` no impide eso, porque el canje lo hace el cliente legítimo con sus credenciales correctas. PKCE sí lo impide: el `code_verifier` que la app de envíos tiene guardado es el de *su* flujo, no el del flujo del atacante, y la verificación falla. Coste de activarlo: dos campos y una llamada a SHA-256.

## Qué grant type elegir

| Grant type | ¿Persona presente? | Dónde corre el Client | Cuándo elegirlo |
|---|---|---|---|
| **Authorization Code + PKCE** | Sí | Web con backend, SPA, app móvil | Por defecto siempre que haya alguien que deba consentir. Es el flujo de la app de envíos |
| **Client Credentials** | No | Servidor propio | Máquina a máquina: un servicio de facturación que lee pedidos con credenciales suyas, sin actuar en nombre de nadie |
| **Device Code** | Sí, pero en otro dispositivo | Aparato sin teclado o sin navegador (TV, impresora, CLI) | Cuando no se puede mostrar un formulario de login: el aparato muestra un código y la persona lo introduce en su móvil |
| **Refresh Token** | No en ese momento | Igual que el flujo que lo emitió | Para renovar un `access_token` caducado sin volver a molestar a la clienta. No es un flujo inicial |

Y los dos que la especificación original incluía y hoy **no se usan**:

| Grant type retirado | Por qué se retiró |
|---|---|
| **Implicit** (`response_type=token`) | Devolvía el `access_token` en la URL de redirección, con todo el rastro que eso deja (historial, logs, `Referer`), y sin forma de entregar un refresh token con seguridad. Su motivo original —la falta de CORS en las SPA— ya no existe |
| **Resource Owner Password Credentials** | El Client pide directamente el usuario y la contraseña de la clienta y los reenvía al Authorization Server. Es exactamente el problema que OAuth2 existe para evitar: rompe la única regla del protocolo. Además impide el consentimiento, el segundo factor y el login federado |

Ambos están eliminados en OAuth 2.1. Si una integración te pide implementar uno de los dos, eso es la señal para preguntar por qué, no para implementarlo.

## Client Credentials: el flujo sin nadie delante

Es el flujo que más se usa en backend y el que la mayoría de las guías cuenta de pasada, porque no tiene pantallas. Aquí no hay Resource Owner: el Client actúa **en su propio nombre**, con sus credenciales, para hablar con otra API.

Supón un servicio interno de facturación que necesita leer pedidos de forma programada. No hay clienta que consienta, así que no hay redirección ni código: se va directo al endpoint de token.

```http
POST /token HTTP/1.1
Host: auth.tienda.ejemplo.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=facturacion-interna
&client_secret=s3cr3t-de-facturacion
&scope=pedidos.read
```

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjA0Y2E...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "pedidos.read"
}
```

Tres cosas que cambian respecto al flujo anterior:

- **No hay `refresh_token`, y es correcto que no lo haya.** El Client tiene sus propias credenciales guardadas: cuando el token caduque, vuelve a pedir otro. Un refresh token aquí solo añadiría un secreto más que proteger.
- **El token no representa a ninguna persona.** Si tu API deduce «el dueño de este token es la clienta 42» a partir de un token de client credentials, la deducción es falsa. Este token dice qué *servicio* llama, no quién.
- **`client_secret` real, rotable y por entorno.** Es la única credencial del flujo. Cómo enviarla sin filtrarla lo cubre [Credenciales en llamadas salientes](../secretos-en-llamadas-salientes/Credenciales-en-Llamadas-Salientes.md).

## `state`: el CSRF sobre el propio login

El parámetro `state` parece burocracia hasta que se ve el ataque concreto que evita. Es un CSRF, pero no contra una operación de la aplicación: **contra el propio proceso de vinculación de cuentas**.

Sin `state`, así funciona:

1. El atacante inicia un flujo normal en `envios.ejemplo.com` y autoriza el acceso **a su propia cuenta** de la tienda.
2. En el paso 2, en lugar de dejar que su navegador siga la redirección, **se queda con el `code`** y no lo canjea.
3. Consigue que la víctima abra `https://envios.ejemplo.com/callback?code=<el-code-del-atacante>` — un enlace en un correo, una imagen, cualquier cosa que dispare esa petición desde el navegador de la víctima.
4. La app de envíos recibe un `code` válido en la sesión de la víctima, lo canjea, obtiene un token legítimo... **de la cuenta del atacante**, y lo vincula a la cuenta de la víctima en la app de envíos.

El resultado depende de la aplicación, y en ninguno de los casos es bueno: la víctima acaba trabajando contra los datos del atacante, o —si la app usa la vinculación al revés— el atacante gana acceso a lo que la víctima suba a partir de ese momento.

`state` lo corta con dos líneas:

```http
# ❌ Sin state: cualquier code que llegue al callback se procesa
GET /authorize?response_type=code&client_id=envios-ejemplo-com&scope=pedidos.read

# ✅ Con state: valor aleatorio guardado en la sesión antes de redirigir
GET /authorize?response_type=code&client_id=envios-ejemplo-com&scope=pedidos.read
    &state=9f2a41c8e7b04d15
```

El Client genera un valor aleatorio impredecible, lo guarda **en la sesión** (no en una variable global ni en la URL), y al recibir el callback comprueba que el `state` que vuelve es idéntico al guardado. Si no lo es, o si no había ninguno guardado, aborta sin canjear nada. El `code` del atacante llega sin `state`, o con un `state` que la sesión de la víctima no reconoce, y se descarta.

PKCE cubre parte del mismo terreno —de hecho, en OAuth 2.1 el `code_verifier` hace también este trabajo—, pero `state` sigue siendo útil por un segundo motivo muy práctico: es el sitio natural para llevar «a dónde iba la persona antes de que la mandáramos a autenticarse».

## Scopes: lo que piden y lo que NO garantizan

Un **scope** es una etiqueta de permiso que el Client solicita y la persona aprueba. Se piden en el parámetro `scope`, separados por espacios:

```text
scope=pedidos.read
scope=pedidos.read pedidos.write direcciones.read
```

La clienta 42 los ve traducidos en la pantalla de consentimiento («*leer tus pedidos*»), y el token que se emite recuerda cuáles se concedieron. El Authorization Server puede conceder **menos** de lo pedido; por eso la respuesta del token incluye su propio campo `scope`, y hay que leerlo en lugar de asumir que se obtuvo todo.

Pide el mínimo, y no «por si acaso más adelante». Cada scope extra es superficie: si el token se filtra, el daño es exactamente lo que el token permitía. Una app que solo calcula etiquetas de envío no necesita `pedidos.write`.

Y ahora el aviso más importante de esta ficha:

> **Un scope no es un permiso de tu dominio.**

Que un token traiga `pedidos.read` significa «este Client puede usar los endpoints de lectura de pedidos». **No dice qué pedidos.** El pedido #4711 es de la clienta 42; con ese mismo token, una petición a `/pedidos/4712` —el pedido de otra persona— llega al Resource Server con un scope perfectamente válido.

```http
# ✅ Autorizado: el token tiene pedidos.read Y el #4711 es de la clienta 42
GET /pedidos/4711
Authorization: Bearer eyJhbGciOi...

# ❌ Scope válido, acceso indebido: el #4712 es de otra clienta
GET /pedidos/4712
Authorization: Bearer eyJhbGciOi...   → debe responder 403, no 200
```

La comprobación «este recurso pertenece a quien pregunta» es lógica de negocio del Resource Server y **nadie la hace por ti**. Este fallo tiene nombre propio en la lista de OWASP —*Broken Object Level Authorization*— y es la vulnerabilidad número uno de APIs precisamente porque el token parece suficiente. La distinción entre «quién eres» y «qué puedes hacer con este objeto concreto» la desarrolla [Autenticación vs Autorización](Autenticacion-vs-Autorizacion.md), y el modelado de esos permisos, [RBAC y Claims](RBAC-y-Claims.md).

## `redirect_uri`: la comparación tiene que ser exacta

La `redirect_uri` es a dónde el Authorization Server manda el `code`. Si esa dirección se puede manipular, el `code` acaba en el servidor de un atacante — y ahí se acabó el flujo.

La regla del estándar es **comparación exacta de cadena** contra las URIs registradas de antemano: mismo esquema, mismo host, mismo puerto, mismo path, sin comodines y sin coincidencias por prefijo.

```text
# ❌ Registro permisivo — cada línea es una vulnerabilidad
https://envios.ejemplo.com/*                 comodín en el path
https://*.ejemplo.com/callback               comodín en el subdominio
https://envios.ejemplo.com                   coincidencia por prefijo

# ✅ Registro exacto, una entrada por cada callback real
https://envios.ejemplo.com/callback
http://localhost:5173/callback               solo en el cliente de desarrollo
```

Por qué falla cada uno de los ❌:

- Con comodín en el path, `https://envios.ejemplo.com/redirigir?a=https://atacante.example` casa con el patrón. Si la app de envíos tiene en cualquier parte una redirección abierta —y las hay más de lo que parece—, el `code` sale rebotado hacia el atacante con el `Referer` puesto.
- Con comodín en el subdominio, basta con controlar **un** subdominio (un `blog.ejemplo.com` en un CMS de terceros, un bucket de estáticos con dominio propio) para recibir códigos de todas las clientas.
- Con coincidencia por prefijo, `https://envios.ejemplo.com.atacante.example/callback` empieza por la cadena registrada y pasa el filtro. Es el mismo error que validar un host con `StartsWith`.

Dos consecuencias prácticas: registra los callbacks de desarrollo como entradas propias y distintas (`http://localhost:5173/callback`), en un cliente de OAuth separado del de producción; y manda la `redirect_uri` también en el `POST /token`, donde el Authorization Server vuelve a comprobar que es la misma que se usó al pedir el código.

## Access token y refresh token

El **access token** es la credencial de uso diario: se manda en cada petición al Resource Server y dura poco a propósito —minutos, no días—, porque no hay forma barata de retirarlo antes de tiempo si es autocontenido. El **refresh token** es la credencial para conseguir access tokens nuevos: vive mucho más (días o semanas), no se manda nunca al Resource Server y se guarda con bastante más cuidado, porque quien lo roba puede fabricar access tokens hasta que alguien lo revoque. La renovación es otro `POST /token`, esta vez con `grant_type=refresh_token`. El patrón completo —dónde guardar cada uno, cuánto durar y cómo rotar el refresh en cada uso para detectar robos— lo cubre [Modelo JWT + Refresh Token](JWT-Refresh.md).

## Lo que OAuth2 deliberadamente no hace

- **No es un protocolo de autenticación.** OAuth2 nunca le dice al Client quién es la persona: le entrega un token con ciertos permisos. Confundir «conseguí un `access_token`» con «sé quién es esta persona» es el error de diseño más común con OAuth2, y no es un matiz académico: un access token de otra aplicación, obtenido legítimamente por un atacante, sirve igual para «autenticarse» en una API que razona así. Para identidad existe [OpenID Connect](OpenID-Connect.md), que añade un `id_token` pensado y validado para eso.
- **No cifra ni firma nada por sí mismo.** La confidencialidad la pone TLS en todas las comunicaciones, sin excepción, y la integridad del token depende de su formato — ver [JWT](JWT.md).
- **No sustituye la gestión de permisos de tu aplicación.** Es la sección de scopes: el token dice qué endpoints, tu código dice qué objetos.
- **No define el formato del token.** El Authorization Server elige, y el Resource Server valida en consecuencia. Las dos opciones y sus consecuencias operativas están en [JWT-Refresh vs Tokens Opacos](JWT-Refresh-vs-Tokens-Opacos.md).

## Cuándo NO usar OAuth2

- **Para el login propio de tu aplicación con tus propias credenciales.** Si el frontend `tienda.ejemplo.com` y la API `api.tienda.ejemplo.com` son tuyos y la clienta se autentica con la cuenta de la tienda, no hay acceso delegado: no hay ningún tercero a quien no quieras dar la contraseña. Una sesión con cookie o un token propio hacen el trabajo con una fracción de las piezas móviles. Las alternativas están en [Sesiones vs Tokens](Sesiones-vs-Tokens.md).
- **Cuando no hay un tercero de por medio.** Client Credentials entre dos servicios tuyos tiene sentido si ya tienes un Authorization Server; montarlo *solo* para eso, cuando una credencial de servicio bien gestionada bastaría, es complejidad sin beneficio.
- **Cuando lo que necesitas es identidad y no acceso.** «Quiero saber quién es esta persona y mostrar su nombre» no es OAuth2: es [OpenID Connect](OpenID-Connect.md). Implementar Authorization Code y deducir la identidad del token es reinventar OIDC mal.

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| `{"error":"invalid_grant"}` al canjear el código | El `code` ya se usó (un reintento, o el navegador repitió el callback), caducó (más de un minuto), o la `redirect_uri` del `POST /token` no es idéntica a la del paso 1 |
| `{"error":"invalid_grant","error_description":"PKCE verification failed"}` | El `code_verifier` enviado no corresponde al `code_challenge` del flujo: la sesión se perdió entre los dos pasos, o hay varias instancias del Client sin estado compartido |
| `{"error":"redirect_uri_mismatch"}` en el paso 1 | La URI no está registrada exactamente: sobra o falta la barra final, es `http` frente a `https`, o el puerto no coincide |
| `{"error":"invalid_client"}` | `client_id` o `client_secret` incorrectos, secreto rotado y no desplegado, o el Client se autentica de una forma que el Authorization Server no espera (cuerpo frente a `Authorization: Basic`) |
| `{"error":"invalid_scope"}` | Se pide un scope que no existe o que ese Client no tiene permitido; también un separador equivocado (comas en lugar de espacios) |
| El token de otra API se acepta en `api.tienda.ejemplo.com` | Nadie valida `aud`: el Resource Server comprueba que la firma es válida pero no que el token fuera emitido **para él** |
| «Ya sé quién es la clienta, tengo su `access_token`» | Se usó un token de autorización como prueba de identidad. Falta el `id_token` de [OpenID Connect](OpenID-Connect.md) |
| Funciona en local y falla en producción | La `redirect_uri` de producción no está registrada, o está registrada con `http`/otro puerto/otra barra final que la real |
| `401` en el Resource Server con un token recién emitido | Reloj desajustado entre servidores (el `exp`/`nbf` no cuadra), o se manda el `refresh_token` en la cabecera en lugar del `access_token` |

## Buenas prácticas avanzadas

- **Usa PKCE siempre, también en clientes confidenciales.** Nació para SPAs y apps móviles que no pueden guardar un secreto, pero OAuth 2.1 lo recomienda de forma universal porque cierra un ataque que el `client_secret` no cubre: la inyección de código de autorización. Coste real de activarlo, dos parámetros y un SHA-256; coste de no tenerlo, un token de otra cuenta vinculado a la sesión de tu clienta.
- **Trata *Implicit* y *Resource Owner Password Credentials* como retirados, no como opciones legadas.** El primero expone el token en la URL del navegador; el segundo obliga al Client a manejar la contraseña de la clienta, que es la única cosa que OAuth2 existe para impedir, y de paso hace imposibles el consentimiento y el segundo factor. Si un proveedor te ofrece ROPC «para simplificar la integración», la respuesta correcta es pedirle Authorization Code.
- **Valida el `state` contra la sesión, no contra sí mismo.** El error sutil no es olvidar el parámetro: es guardarlo donde el atacante también lo controla —una cookie sin `HttpOnly`, un `localStorage` compartido, un campo oculto— y «validar» comparando dos valores que vienen ambos de la misma petición. El valor tiene que estar en la sesión del servidor o en una cookie firmada, y el flujo tiene que abortar cuando no haya ninguno guardado, no solo cuando no coincida.
- **Pide los scopes mínimos y auditalos como código.** El scope se escribe una vez, al integrar, y nadie lo vuelve a mirar en dos años. Un `pedidos.write` que se pidió «porque igual luego hará falta» es acceso de escritura real disponible el día que el token se filtre. Ten la lista de scopes de cada Client en el repositorio, revisable en un diff, y quita los que el código no use.
- **No reutilices un token entre audiencias, y haz que el Resource Server lo compruebe.** Un `access_token` emitido para `api.tienda.ejemplo.com` no debe valer en otra API, y la única defensa efectiva es que cada Resource Server valide el claim `aud` además de la firma. Es un fallo silencioso: con un Authorization Server compartido, todas las firmas son válidas en todas partes, así que sin esa comprobación un token de una integración de baja confianza abre la API de pedidos y nada en los logs lo delata.

## Documentación oficial

- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749) — la base. Ve a la sección 4.1 para el Authorization Code paso a paso con los parámetros normativos, y a la 5.2 para la lista literal de códigos de error (`invalid_grant`, `invalid_client`, `invalid_scope`) cuando estés depurando una respuesta que no entiendes.
- [RFC 7636 — Proof Key for Code Exchange](https://datatracker.ietf.org/doc/html/rfc7636) — PKCE. La sección 4 da la derivación exacta del `code_challenge` y el apéndice B un ejemplo con valores concretos, que es lo que necesitas para comprobar que tu implementación de base64url no mete relleno.
- [RFC 9700 — Best Current Practice for OAuth 2.0 Security](https://datatracker.ietf.org/doc/html/rfc9700) — la lectura más útil de las cuatro y la que menos gente conoce. Es el documento que explica *por qué* se retiraron Implicit y ROPC, describe los ataques concretos (inyección de código, `redirect_uri` laxa, mezcla de Authorization Servers) y da la recomendación aplicable en cada caso. Si solo vas a leer una, lee esta.
- [Borrador de OAuth 2.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1) — consolida RFC 6749, PKCE y las buenas prácticas en un único texto, ya sin los flujos retirados. Ve aquí para saber qué se considera vigente hoy sin tener que cruzar cuatro documentos.

## Recursos didácticos

- [OAuth 2.0 Playground de oauth.com](https://www.oauth.com/playground/) — ejecuta el flujo completo contra un servidor de pruebas mostrando **cada petición y cada respuesta** literal, incluidos el `code_verifier`, el `code_challenge` y el `state`. Es la forma más rápida de ver el flujo de esta ficha funcionando de verdad, y de comprobar qué falla al manipular un parámetro a propósito.
- [OAuth 2.0 Debugger](https://oauthdebugger.com/) — construye la URL de autorización parámetro a parámetro contra tu propio Authorization Server y muestra la respuesta cruda. Útil para aislar si el problema está en tu URL o en el registro del cliente, antes de tocar código.
- [oauth.net/2](https://oauth.net/2/) — mapa curado de todas las extensiones (Device Code, Token Introspection, DPoP, Token Exchange) con enlace a su RFC. El sitio al que ir cuando alguien menciona una sigla de OAuth que no reconoces.

---

*En resumen: OAuth2 permite que una aplicación acceda a tus datos en otro servicio con permisos limitados y revocables, sin que tu contraseña salga nunca de ese servicio — resuelve autorización delegada, no autenticación.*
