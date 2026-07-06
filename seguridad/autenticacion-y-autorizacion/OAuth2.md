# OAuth2

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

OAuth2 es un estándar de **autorización delegada**: permite que una aplicación acceda a recursos de una usuaria alojados en otro servicio (sus fotos, su calendario, su lista de contactos) sin que esa usuaria tenga que entregarle su usuario y contraseña de ese otro servicio.

## ¿Por qué existe?

Antes de OAuth2, si una app de gestión de tareas quería leer tu calendario de Google, la única forma era pedirte directamente tu usuario y contraseña de Google y guardarlos. Esto es un desastre de seguridad: la app de tareas tiene acceso total e indefinido a tu cuenta de Google, no solo al calendario, y tú no puedes revocar ese acceso sin cambiar tu contraseña en todas partes.

OAuth2 rompe ese acoplamiento: la usuaria nunca comparte su contraseña con la aplicación de terceros. En su lugar, se autentica directamente en el servicio dueño del recurso (Google, en el ejemplo) y ese servicio le entrega a la aplicación un token con permisos limitados y revocables, sin que la contraseña salga nunca de su sitio.

> Piensa en OAuth2 como la llave de hotel de una tarjeta magnética: no le das al botones la llave de tu casa; el hotel te da una tarjeta que abre solo tu habitación, solo durante tu estancia, y que pueden desactivar en cualquier momento sin que tú cambies nada.

## ¿Cuándo y para qué se usa?

- **"Conceder acceso a" un servicio de terceros**: una app de gestión de tareas que quiere leer y crear eventos en tu Google Calendar.
- **"Iniciar sesión con Google/GitHub/Microsoft"**: aunque el caso de uso parezca autenticación, técnicamente lo que ocurre por debajo es autorización — se explica bien en la ficha de [OpenID Connect](OpenID-Connect.md), que se construye encima de OAuth2 precisamente para cubrir ese hueco.
- **Comunicación máquina a máquina**: un servicio de facturación que llama a la API de otro servicio interno usando credenciales propias, sin que haya ninguna persona detrás en ese momento.

## Lo mínimo que necesitas saber

**1. Los cuatro roles**

- **Resource Owner**: la usuaria dueña de los datos (tú, dueña de tu calendario).
- **Client**: la aplicación que quiere acceder a esos datos (la app de tareas).
- **Authorization Server**: quien autentica a la usuaria y emite los tokens (el servidor de identidad de Google).
- **Resource Server**: la API que realmente guarda el recurso y acepta el token (la API de Google Calendar).

**2. El flujo Authorization Code (el más habitual)**

```
1. El Client redirige a la usuaria al Authorization Server con los scopes solicitados.
2. La usuaria se autentica ahí (no en el Client) y aprueba el acceso.
3. El Authorization Server redirige de vuelta al Client con un "código" de un solo uso.
4. El Client intercambia ese código, en una llamada servidor-a-servidor, por un access_token.
5. El Client usa el access_token para llamar al Resource Server.
```

La contraseña de la usuaria solo se escribe en el paso 2, en la web del Authorization Server — nunca la ve el Client.

**3. Grant types principales**

- **Authorization Code (+ PKCE)**: el flujo estándar para aplicaciones con usuaria presente (webs, SPAs, móvil). PKCE añade un secreto de un solo uso generado por el propio cliente para que el código intercambiado en el paso 4 no sirva si alguien lo intercepta.
- **Client Credentials**: sin usuaria de por medio — un servicio se autentica con su propio `client_id`/`client_secret` para hablar con otro servicio (machine-to-machine).
- Los flujos *Implicit* y *Resource Owner Password Credentials* existían en la especificación original pero están desaconsejados hoy (ver buenas prácticas).

**4. Scopes: permisos concretos, no todo o nada**

```
scope=calendar.readonly contacts.readonly
```

El Client solicita exactamente los permisos que necesita, y la usuaria ve y aprueba esa lista concreta antes de conceder el acceso.

**5. Access token vs refresh token**

El *access token* es el que se usa para llamar a la API y suele durar poco. El *refresh token*, de vida más larga, permite pedir un access token nuevo sin que la usuaria vuelva a autenticarse — se guarda con más cuidado porque su robo tiene más alcance.

## Lo que NO hace

- **No es un protocolo de autenticación** — OAuth2 nunca dice al Client quién es la usuaria, solo le entrega un token con ciertos permisos. Confundir "conseguí un access_token" con "sé quién es esta persona" es el error de diseño más común con OAuth2; para eso existe [OpenID Connect](OpenID-Connect.md).
- **No cifra ni firma nada por sí mismo** — la seguridad depende de TLS en todas las comunicaciones y del formato concreto del token (a menudo JWT, ver esa ficha).
- **No sustituye la gestión de permisos interna de tu aplicación** — que un Client tenga un access_token válido con scope `orders.read` no dice qué pedidos concretos puede ver; esa lógica de negocio sigue siendo responsabilidad del Resource Server.

## Buenas prácticas avanzadas

- **Usa PKCE siempre, incluso en aplicaciones confidenciales** — nació para SPAs y apps móviles que no pueden guardar un secreto, pero OAuth 2.1 lo recomienda de forma universal porque cierra el hueco de interceptación del código de autorización sin ningún coste real.
- **Evita el Implicit grant y el Resource Owner Password Credentials grant** — el primero expone el token directamente en la URL del navegador (más superficie de fuga); el segundo obliga al Client a manejar la contraseña de la usuaria, justo lo que OAuth2 existe para evitar. Ambos están retirados en OAuth 2.1.
- **Valida siempre el parámetro `state`** — el Client debe generar un valor aleatorio antes de redirigir al Authorization Server y comprobar que vuelve intacto; sin esto, el flujo es vulnerable a que un atacante fuerce a la víctima a autorizar una sesión que el atacante controla (CSRF sobre el propio login).
- **Pide los scopes mínimos, no los máximos "por si acaso"** — cada scope adicional es superficie de exposición si el token se filtra; una app que solo necesita leer el calendario no debería pedir también permisos de escritura.
- **No reutilices el mismo token entre audiencias distintas** — un access_token pensado para la API de calendario no debería aceptarse también en la de contactos; cada Resource Server debe validar que el token fue emitido para él (`aud`).

---

*En resumen: OAuth2 permite que una aplicación acceda a tus datos en otro servicio con permisos limitados y revocables, sin que tu contraseña salga nunca de ese servicio — resuelve autorización delegada, no autenticación.*
