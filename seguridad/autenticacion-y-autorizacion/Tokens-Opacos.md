# Tokens opacos de sesión

## ¿Qué es?

Un token opaco (también llamado *reference token* o token de referencia) es una cadena aleatoria sin ningún significado por sí misma: no contiene datos, no se puede decodificar, no dice nada de quién eres. Es solo una **llave** que el servidor usa para buscar, en su propio almacén, a qué sesión pertenece. Es el modelo clásico de la cookie de sesión de toda la vida en la web.

## ¿Por qué existe?

Un token autocontenido como el [JWT](JWT.md) lleva la información dentro y firmada, lo que es cómodo pero tiene tres costes: cualquiera puede leer su contenido (solo va en Base64, no cifrado), no se puede revocar antes de que expire, y crece de tamaño según metes claims. El token opaco invierte el planteamiento: **no lleva nada**. Toda la información —quién es la clienta, sus permisos, cuándo caduca— vive en el servidor, y el token es apenas un identificador.

Eso le da tres superpoderes: no filtra información (no hay nada que leer), se revoca al instante (borras la fila del almacén y la sesión muere) y ocupa poquísimo. Y hay un cuarto, más sutil: como el servidor lee el estado *vivo* en cada petición, cualquier cambio —bajar un rol de `Administrador` a `Empleado`, desactivar la cuenta— surte efecto de inmediato, sin esperar a que caduque nada. El precio es que el servidor **sí tiene que recordar**, y por tanto consultar su almacén en cada petición.

> Si ya conoces los JWT, piensa en un token opaco como el resguardo numerado del guardarropa de un teatro: el papelito no dice qué abrigo es tuyo ni de qué color, solo lleva un número; toda la información está en el mostrador, y quien tenga el mostrador puede anular tu resguardo cuando quiera.

Esta ficha explica **cómo funciona este modelo por dentro**. La comparación frontal con el modelo autocontenido, y cuál conviene en cada caso, es otro tema: está en [JWT + Refresh vs Tokens opacos](JWT-Refresh-vs-Tokens-Opacos.md).

## ¿Cuándo y para qué se usa?

- **Aplicaciones web con sesión de servidor**: un panel de administración de la tienda renderizado en servidor, donde el navegador guarda una cookie con el identificador de sesión y el servidor mantiene el estado (con estado frente a sin estado como comparación general, en [Sesiones vs Tokens](Sesiones-vs-Tokens.md)).
- **Sistemas que necesitan revocación inmediata**: banca, salud o cualquier contexto donde "cerrar sesión ahora" tras detectar un robo de cuenta no es negociable.
- **Access tokens opacos en OAuth2**: en lugar de emitir JWT, el servidor de identidad puede emitir tokens opacos y ofrecer un endpoint de *introspección* que las APIs consultan para validarlos. El flujo de OAuth2 y sus grant types están en [OAuth2](OAuth2.md).
- **Como refresh token**: un refresh token es, casi siempre, un token opaco — precisamente porque hace falta poder revocarlo. Ese patrón concreto (access token corto más refresh, rotación, familias de tokens) se desarrolla en [Modelo JWT + Refresh Token](JWT-Refresh.md).

El escenario que recorre toda la ficha es una tienda online con la API en .NET en `api.tienda.ejemplo.com`, el frontend en `tienda.ejemplo.com`, y tablas `Clientes`, `Pedidos` y `Sesiones`. La clienta de referencia tiene el identificador **42** y su pedido es el **#4711**. Cuando hable del almacén compartido, supón la API desplegada en **cuatro instancias** detrás de un balanceador.

---

## Anatomía: solo entropía

Un token opaco no tiene estructura: son bytes aleatorios de una fuente criptográficamente segura (un CSPRNG, *Cryptographically Secure Pseudo-Random Number Generator*), codificados como texto. No hay partes ni claims que extraer, porque no hay nada dentro.

```csharp
// 32 bytes = 256 bits de entropía, generados con un CSPRNG
var bytes = RandomNumberGenerator.GetBytes(32);
string token = Convert.ToBase64String(bytes);
// Ejemplo: "9f8Ka2vQ7pZ1sT4xN0bY6cE3hJ8mR5wD2gL7kU9oA1s=" — no dice nada por sí mismo
```

Devuelve una cadena de 44 caracteres. Nadie, mirándola, puede saber que pertenece a la clienta 42, ni cuándo caduca, ni qué rol tiene: esa información no viaja.

### Por qué 256 bits y no menos

Este es el único secreto que separa a un atacante de una sesión ajena, así que la pregunta importante es cuántos intentos hacen falta para acertar uno **vivo**. El cálculo es directo: si el espacio tiene 2^n valores y hay S sesiones activas, la probabilidad de acertar por intento es S / 2^n.

Con la tienda funcionando bien, digamos **un millón de sesiones vivas**:

| Entropía | Espacio de valores | Intentos medios para acertar una sesión viva | Tiempo a 10.000 intentos/s |
|---|---|---|---|
| 32 bits | 4.294.967.296 | ~4.295 | menos de un segundo |
| 64 bits | 1,8 × 10^19 | ~1,8 × 10^13 | ~58 años |
| 128 bits | 3,4 × 10^38 | ~3,4 × 10^32 | más que la edad del universo |
| 256 bits | 1,2 × 10^77 | ~1,2 × 10^71 | irrelevante |

Con 32 bits, un atacante que itere identificadores **entra en una sesión ajena en menos de un segundo**: no necesita romper nada, solo probar. Con 128 o 256 bits no hay fuerza bruta posible ni con todo el hardware del planeta, así que el ataque tiene que ir por otro lado (robar la cookie, no adivinarla). 256 bits es el valor que se recomienda por defecto porque el coste de generar 32 bytes en vez de 16 es exactamente cero.

### El caso `Guid.NewGuid()`: parece aleatorio y no sirve

Es el atajo más frecuente y el más razonable en apariencia — un GUID es único, largo y con pinta de aleatorio:

```csharp
// ❌ No sirve como token de sesión
string token = Guid.NewGuid().ToString();
// "b3f1c9e2-7a48-4d61-9c05-2e8f4a6b1d73"
```

Tres problemas concretos. Primero, de sus 128 bits **6 están fijados** por la especificación (los que marcan la versión y la variante), así que como máximo hay 122 bits impredecibles, y no todos lo son según la versión de GUID: los de tipo 1 se derivan de la marca de tiempo y la dirección MAC de la máquina, y son perfectamente predecibles. Segundo, la generación de GUID **está pensada para garantizar unicidad, no imprevisibilidad**, y no está garantizado que venga de un generador criptográfico en todas las plataformas y versiones. Tercero, quien lee el código no puede distinguir a simple vista si el GUID que tiene delante es de un tipo seguro o no, lo cual es en sí mismo un problema.

```csharp
// ✅ La intención es explícita: bytes de un generador criptográfico
string token = Convert.ToBase64String(RandomNumberGenerator.GetBytes(32));
```

La regla práctica: `Guid.NewGuid()` para identificadores de fila, `RandomNumberGenerator` para cualquier cosa que sea un secreto. Y nunca `Random`, que es determinista y reproducible a partir de su semilla.

## Dónde vive la sesión: la tabla `Sesiones`

Como el token no lleva nada, la sesión tiene que ser una fila en algún sitio. Este es el esquema mínimo que hace falta, y cada columna está por un motivo:

```sql
CREATE TABLE Sesiones (
    TokenHash       BINARY(32)    NOT NULL PRIMARY KEY,  -- SHA-256 del token, nunca el token
    ClienteId       INT           NOT NULL,
    CreadaEn        DATETIME2     NOT NULL,
    UltimaActividad DATETIME2     NOT NULL,              -- para la caducidad deslizante
    ExpiraAbsoluto  DATETIME2     NOT NULL,              -- tope duro, pase lo que pase
    Dispositivo     NVARCHAR(256) NULL,                  -- User-Agent resumido
    IpOrigen        NVARCHAR(45)  NULL,                   -- cabe una IPv6
    CONSTRAINT FK_Sesiones_Clientes FOREIGN KEY (ClienteId) REFERENCES Clientes(Id)
);

CREATE INDEX IX_Sesiones_ClienteId ON Sesiones(ClienteId);
```

Dos decisiones de diseño que merecen atención:

- **La clave primaria es el hash del token, no un `Id` autoincremental.** La búsqueda de cada petición es exactamente "dame la fila cuyo hash es este", así que el índice que necesitas es ese. Un autoincremental añadiría una columna que nadie consulta.
- **El índice sobre `ClienteId` no es decorativo.** Es lo que hace baratas las consultas "¿qué dispositivos tiene abiertos la clienta 42?" y "cierra todas sus sesiones". Sin él, esas dos operaciones recorren la tabla entera.

Y esa segunda es la clave: **este esquema es lo que habilita la gestión de sesiones concurrentes** de la última buena práctica. Listar los dispositivos activos es un `SELECT`, cerrar la sesión de uno solo es un `DELETE` por hash, y "cerrar sesión en todos los dispositivos" es un `DELETE ... WHERE ClienteId = 42`. Tres funcionalidades que en el modelo autocontenido exigen inventarse una lista de revocación, aquí salen del esquema.

```sql
-- Los dispositivos abiertos de la clienta 42, el más reciente primero
SELECT Dispositivo, IpOrigen, UltimaActividad
FROM Sesiones
WHERE ClienteId = 42 AND ExpiraAbsoluto > SYSUTCDATETIME()
ORDER BY UltimaActividad DESC;
```

## La validación en cada petición

Este es el corazón del modelo, y el orden de las comprobaciones importa: primero lo que se descarta sin tocar el almacén, después la consulta, y solo al final la actualización.

```csharp
public async Task<Sesion?> ValidarAsync(string token)
{
    // 1. Descarte barato: formato inválido, ni siquiera vamos al almacén
    if (string.IsNullOrEmpty(token) || !token.StartsWith("sess_"))
        return null;

    // 2. El token nunca se guarda en claro: buscamos por su hash
    byte[] hash = SHA256.HashData(Encoding.UTF8.GetBytes(token));
    var sesion = await _store.BuscarPorHashAsync(hash);   // Redis, BD, caché...
    if (sesion is null) return null;

    // 3. Comparación en tiempo constante del hash recuperado
    if (!CryptographicOperations.FixedTimeEquals(sesion.TokenHash, hash))
        return null;

    // 4. Caducidad: primero la absoluta, luego la deslizante
    var ahora = DateTime.UtcNow;
    if (sesion.ExpiraAbsoluto <= ahora ||
        sesion.UltimaActividad.AddMinutes(30) <= ahora)
    {
        await _store.EliminarAsync(hash);   // limpieza oportunista
        return null;
    }

    // 5. Solo si todo lo anterior pasó, renovamos la ventana deslizante
    await _store.TocarAsync(hash, ahora);
    return sesion;   // aquí están el ClienteId, los roles, la caducidad...
}
```

Devuelve la sesión con el `ClienteId = 42` y su rol (`Cliente`, `Empleado` o `Administrador`), o `null`, que la capa de autenticación traduce a un `401`.

Sobre el paso 3: `FixedTimeEquals` recorre siempre los 32 bytes completos, mientras que una comparación normal se detiene en el primer byte distinto y el tiempo de respuesta filtra información sobre cuántos bytes acertaste. Aquí es cinturón sobre tirantes —el hash ya viene de un token imposible de adivinar— pero es gratis y evita el error de fondo: en C#, `==` sobre dos `byte[]` compara **referencias**, no contenido, así que siempre devuelve `false` y la validación falla para todo el mundo.

### El coste: una ida y vuelta por petición

Aquí está el verdadero precio del modelo, y conviene verlo con números en vez de en abstracto. Cargar una página de la tienda son fácilmente **20 peticiones** a la API (el listado de `Pedidos`, el detalle del #4711, el carrito, las notificaciones, cada imagen protegida...), y cada una valida la sesión:

| Almacén de sesiones | Latencia por consulta | Coste de una carga de página (20 peticiones) |
|---|---|---|
| Base de datos relacional | ~2 ms | **40 ms** solo en validar |
| Redis en la misma red | ~0,5 ms | **10 ms** |
| Memoria del proceso | ~0 ms | 0 ms — pero no funciona con varias instancias (ver más abajo) |

40 ms de latencia añadida que no hacen nada visible para la clienta, multiplicados por cada usuaria simultánea, es una carga considerable sobre la base de datos que además atiende `Pedidos`. De ahí que **el almacén de sesiones sea uno de los casos de uso canónicos de una caché**: los datos son pequeños, se leen constantemente, se escriben poco y son prescindibles (perderlos obliga a volver a iniciar sesión, no corrompe nada). El tema se desarrolla en la [colección de caching](../../bases-de-datos/caching/README.md).

## La cookie que lo transporta

En una aplicación web, el token viaja en una cookie. Cada atributo de esta llamada está resolviendo un problema distinto:

```csharp
Response.Cookies.Append("session_id", token, new CookieOptions
{
    HttpOnly = true,
    Secure   = true,
    SameSite = SameSiteMode.Lax,
    Path     = "/",
    Expires  = null                  // cookie de sesión: muere al cerrar el navegador
});
```

- **`HttpOnly = true`** — el JavaScript de la página no puede leer la cookie (`document.cookie` no la ve). Si alguien consigue inyectar un script en `tienda.ejemplo.com`, no puede robar el token y mandárselo a su servidor. No impide que use la sesión desde el navegador de la víctima, pero sí que se la lleve.
- **`Secure = true`** — el navegador solo envía la cookie por HTTPS. Sin esto, una sola petición accidental a `http://` entrega el token en claro a cualquiera que esté en la red.
- **`SameSite = SameSiteMode.Lax`** — el navegador no adjunta la cookie en peticiones originadas en otro sitio, salvo cuando es una navegación de nivel superior por `GET`. Esa excepción es exactamente lo que necesitas: si alguien comparte el enlace `tienda.ejemplo.com/pedidos/4711` en un correo o en un chat, con `Lax` la clienta llega ya autenticada, y con **`Strict` aterrizaría en la pantalla de login** aunque tenga la sesión abierta. `Strict` es más seguro y correcto para un panel de administración sin enlaces entrantes; para una tienda pública es una fricción que nadie entiende. Los conceptos de CSRF y XSS que hay debajo de estos dos atributos están en [Sesiones vs Tokens](Sesiones-vs-Tokens.md).
- **`Path = "/"`** — la cookie se envía a toda la aplicación. Restringirla a una ruta suena a defensa en profundidad y casi nunca lo es: el aislamiento real entre sitios lo da el dominio, no la ruta.
- **`Expires = null`** — sin fecha, es una *cookie de sesión* y desaparece al cerrar el navegador. Que la sesión sobreviva o no al cierre es una decisión de producto ("mantenerme conectada"), pero lo importante es que **la caducidad real la manda el servidor**: la fecha de la cookie es una sugerencia que el cliente puede ignorar, y la fila de `Sesiones` es la verdad.
- **`Domain` sin fijar** — al no ponerlo, la cookie queda ligada al host exacto que la emitió (`api.tienda.ejemplo.com`) y no se envía a ningún otro. Poner `Domain = ".tienda.ejemplo.com"` la reparte por **todos** los subdominios, presentes y futuros: un `blog.tienda.ejemplo.com` con una vulnerabilidad, o un subdominio de un proveedor externo, recibiría el token de sesión sin necesitarlo. Fíjalo solo cuando de verdad compartas sesión entre subdominios, y sabiendo lo que amplías.

## Revocar es borrar

Este es el argumento decisivo del modelo, y se ve mejor con un caso concreto que con una explicación.

A las 10:00 se despide a la persona con rol `Empleado` que tiene acceso al panel de pedidos. Su sesión sigue abierta en el navegador de su portátil, con un token válido emitido esa mañana:

```csharp
// Revocar la sesión: una fila menos
await _store.EliminarAsync(SHA256.HashData(Encoding.UTF8.GetBytes(token)));

// O todas las sesiones de esa persona, en todos sus dispositivos:
await _store.EliminarPorClienteAsync(clienteId);
```

A las 10:00:01 su siguiente petición a `api.tienda.ejemplo.com` busca la fila, no la encuentra y recibe un `401`. No hay ventana, no hay margen, no hay "hasta que expire".

Contrasta esto con el modelo autocontenido: un [JWT](JWT.md) firmado es válido porque la firma cuadra y la fecha `exp` no ha pasado, y **no hay nada que borrar** — el token no vive en tu servidor, vive en el cliente. Puedes montar una lista de revocación, pero eso significa consultar un almacén en cada petición, que es precisamente lo que el JWT venía a evitar: acabas con el coste del modelo con estado y sin sus ventajas. Con un token opaco, la revocación no es una funcionalidad que se añade, es una consecuencia del diseño.

## El estado compartido es obligatorio con varias instancias

Supón la API en cuatro instancias detrás de un balanceador, y la sesión guardada en memoria del proceso (un diccionario, o la caché en memoria por defecto de ASP.NET Core). La clienta 42 inicia sesión y el balanceador la manda a la instancia A, que crea la entrada en su memoria. Su siguiente petición cae en la instancia C, que **no sabe nada de ese token**: `401`, sesión perdida, sin ningún error en los logs que lo explique.

El síntoma es característico: la sesión se cae "aleatoriamente", más o menos tres de cada cuatro veces con cuatro instancias, y funciona perfectamente en local, donde hay una sola.

Hay dos salidas:

| Opción | Cómo funciona | Qué cuesta |
|---|---|---|
| **Almacén compartido** (Redis, base de datos) | Las cuatro instancias leen y escriben en el mismo sitio | Una dependencia más y la latencia por petición de la tabla anterior |
| ***Sticky sessions*** en el balanceador | El balanceador manda siempre a la misma clienta a la misma instancia | Rompe los despliegues y el reparto de carga |

**El almacén compartido es la respuesta correcta.** Las *sticky sessions* parecen gratis y no lo son: cada vez que despliegas, las instancias se reinician una a una y **todas las sesiones que vivían en su memoria se pierden**, así que un despliegue rutinario echa a la calle a las personas que estaban comprando. Además el reparto de carga se degrada (una instancia con las sesiones "pesadas" no puede aliviarse), y escalar añadiendo una quinta instancia no reequilibra nada, porque las sesiones ya están pegadas donde estaban. Sirven como parche temporal para una aplicación heredada; no como diseño.

## Introspección: cuando quien valida no es quien emitió

Hasta aquí, el servicio que valida el token es el mismo que lo creó y tiene acceso a `Sesiones`. En OAuth2 eso a menudo no se cumple: el token lo emite un servidor de identidad y lo tiene que validar la API de la tienda, que **no puede leerlo** (es opaco) ni tiene acceso a la base de datos del emisor.

La solución estándar es preguntar al emisor por el endpoint de *introspección* del [RFC 7662](https://datatracker.ietf.org/doc/html/rfc7662). La API manda el token y sus propias credenciales de cliente:

```http
POST /introspect HTTP/1.1
Host: identidad.tienda.ejemplo.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic YXBpLXRpZW5kYTpzZWNyZXRv

token=sess_9f8Ka2vQ7pZ1sT4xN0bY6cE3hJ8mR5wD2gL7kU9oA1s
```

Si el token está vivo, la respuesta cuenta quién es y qué puede hacer:

```json
{
  "active": true,
  "sub": "42",
  "scope": "pedidos:leer pedidos:crear",
  "client_id": "tienda-web",
  "exp": 1753700400
}
```

Y si no vale —caducado, revocado, inexistente o simplemente inventado— la respuesta es siempre la misma, sin distinguir el motivo:

```json
{ "active": false }
```

Que sea idéntica en todos los casos es deliberado: un mensaje que dijera "caducado" frente a "no existe" confirmaría a un atacante que ese token fue válido alguna vez.

**El aviso práctico**: la introspección es una llamada de red por petición, mucho más caro que la consulta a Redis de antes. En la práctica se cachea la respuesta unos segundos, y ahí está el precio oculto: durante esos segundos la API sigue aceptando un token que el emisor ya ha revocado. Es decir, **cachear la introspección reintroduce la ventana de revocación** que era la mejor propiedad del modelo. El ajuste habitual es una caché muy corta (5-30 s) y aceptar explícitamente esa ventana, o hacer que el emisor avise a las APIs cuando revoca algo.

## Lo que este modelo no te da

- **No se valida solo** — a diferencia de un JWT, no puedes comprobar un token opaco mirándolo: siempre hace falta una consulta al almacén (o al endpoint de introspección). Es un viaje de ida y vuelta más por petición, siempre.
- **No escala sin estado compartido** — como la sesión vive en el servidor, varias instancias necesitan un almacén común o *sticky sessions*; no basta con validar una firma en local como con un JWT.
- **No lleva información utilizable por el cliente** — si el frontend de `tienda.ejemplo.com` necesita saber el nombre o el rol de la clienta para pintar el menú, no puede sacarlo del token: tiene que pedirlo a un endpoint aparte del tipo `GET /api/yo`. Con un JWT, el frontend decodifica la parte de datos y lo tiene gratis.

## Cuándo NO usar un token opaco

| Situación | Por qué falla el token opaco | Hacia dónde mirar |
|---|---|---|
| El consumidor de tu API es un tercero, en otra organización | No puede consultar tu almacén, y darle acceso a `Sesiones` es impensable | [JWT](JWT.md) con clave pública, o introspección si el tercero acepta la dependencia |
| La latencia de una consulta por petición no es aceptable y no puedes poner una caché delante | Los 40 ms de la tabla no se pueden reducir sin caché | [JWT](JWT.md), que valida en local con la firma |
| Arquitectura donde cada servicio debe validar por sí mismo, sin depender del emisor | El emisor se convierte en un punto único de fallo: si cae, nadie valida nada | [JWT](JWT.md) con verificación local de firma |
| Necesitas revocación inmediata **y** validación local | Ninguno de los dos modelos puros lo da | El modelo híbrido, en [JWT + Refresh vs Tokens opacos](JWT-Refresh-vs-Tokens-Opacos.md) |

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| La sesión se pierde de forma intermitente, y en local nunca pasa | Cuatro instancias con la sesión en memoria del proceso, sin almacén compartido |
| Todo el mundo pierde la sesión en cada despliegue | La sesión estaba en memoria del proceso; el reinicio se la lleva |
| Una filtración de la base de datos entrega sesiones vivas y utilizables | El token se guardó en claro en `Sesiones` en vez de su hash |
| Los tokens se pueden predecir o iterar | Se usó `Guid.NewGuid()` (o `Random`) en vez de un CSPRNG |
| La validación falla para todo el mundo desde un cambio "inocente" | Se comparó el hash con `==` sobre `byte[]`, que compara referencias y siempre da `false` |
| Una sesión anónima sigue viva y autenticada tras el login | No se rotó el token al autenticar: es *session fixation* |
| La tabla `Sesiones` tiene millones de filas y las consultas se degradan | Nadie escribió el proceso de limpieza: hace falta un trabajo periódico que borre las filas con `ExpiraAbsoluto` pasado. Es la pieza que falta casi siempre, porque la aplicación funciona perfectamente sin ella durante meses |
| Un `401` esporádico justo después de revocar en el servidor de identidad... o justo antes | La caché de introspección todavía tiene la respuesta `active: true` |

## Buenas prácticas avanzadas

- **Exige entropía suficiente y una fuente criptográfica.** El token es lo único que separa a un atacante de una sesión ajena, así que debe ser imposible de adivinar o iterar: mínimo 128 bits, mejor 256, desde un CSPRNG (`RandomNumberGenerator`), nunca `Guid.NewGuid()` ni `Random`, que no están pensados para secretos. La tabla de entropía de arriba es el argumento: con 32 bits se entra en una sesión viva en menos de un segundo.
- **Guarda en el almacén solo el hash del token, no el token en claro.** Igual que con las contraseñas: si te filtran la base de datos de sesiones, un hash no permite secuestrar sesiones vivas. Y aquí un SHA-256 rápido es lo *correcto*, no una concesión: como el token ya tiene 256 bits de entropía y es imposible de adivinar, no necesita un hash lento tipo bcrypt o Argon2. Esos existen para las contraseñas —que tienen poca entropía y hay que encarecer su fuerza bruta a propósito—; aplicarlos a un token aleatorio solo añadiría latencia inútil en cada petición. El razonamiento completo está en [Contraseñas vs tokens de sesión](../algoritmos-de-hash/Contrasenas-Vs-Tokens-De-Sesion.md).
- **Rota el identificador al autenticar.** Genera un token nuevo justo después del login y borra el anterior, en lugar de reutilizar el que la clienta tenía como anónima mientras llenaba el carrito. Evita la *session fixation*, donde un atacante fija un identificador de sesión que él conoce antes del login y luego lo aprovecha ya autenticado. Rota también al cambiar la contraseña y al elevar privilegios.
- **Combina caducidad deslizante con una absoluta.** Renueva la ventana con la actividad (`UltimaActividad`, 30 minutos) para no expulsar a quien está trabajando, pero impón un tope absoluto (`ExpiraAbsoluto`, 12 horas o 7 días) tras el cual la sesión muere pase lo que pase. Con solo la deslizante, una sesión que se toca cada 29 minutos vive eternamente, y eso es exactamente lo que hace una pestaña abierta con *polling* en un ordenador compartido.
- **Añade un prefijo detectable al token.** Un prefijo reconocible estilo `sess_` cumple dos funciones a la vez: permite que los escáneres de secretos y los sistemas de detección de fugas identifiquen tokens de sesión en logs, capturas de pantalla o repositorios —y avisen antes que el atacante—, y descarta de inmediato las entradas con formato inválido **antes de tocar el almacén**, que es el paso 1 del código de validación. Un escaneo masivo de tokens inventados deja de generar carga en Redis.
- **Aprovecha el estado para gestionar sesiones concurrentes.** Como cada sesión es una fila, tienes casi gratis lo que a un JWT le cuesta sangre: listar los dispositivos activos de la clienta 42, cerrar uno solo, o hacer un "cerrar sesión en todos los dispositivos" con un `DELETE ... WHERE ClienteId = 42`. Es también el mecanismo de respuesta ante un robo de cuenta. Modela la tabla pensando en ello desde el principio —una fila por sesión, con `ClienteId` indexado, `Dispositivo` e `UltimaActividad`— y esa funcionalidad sale casi sola; añadirla después obliga a migrar sesiones vivas, o a expulsar a todo el mundo.

## Documentación oficial

- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) — la referencia normativa de este tema. Ve directamente a las secciones *Session ID Properties* (fija los mínimos de entropía y longitud que justifican los 256 bits) y *Session Expiration*. Es el documento que citar cuando alguien pregunte de dónde salen estas cifras.
- [RFC 7662 — OAuth 2.0 Token Introspection](https://datatracker.ietf.org/doc/html/rfc7662) — la especificación del endpoint de introspección. La sección 2.2 lista todos los campos de la respuesta y la 4 explica por qué `active: false` no debe revelar el motivo. Consúltalo cuando tengas que implementar el endpoint o consumir el de un tercero.
- [RFC 6265bis — Cookies: HTTP State Management](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis) — la especificación viva de cookies, la que define el comportamiento real de `SameSite`, `Secure` y `Domain` en los navegadores actuales. Es la fuente a la que ir cuando el comportamiento observado no coincide con lo que esperabas de un atributo.
- [Cookie Authentication en ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authentication/cookie) — la documentación del framework, con la configuración de caducidad deslizante y absoluta y los puntos de extensión para validar la sesión contra tu almacén en cada petición (`OnValidatePrincipal`).

## Recursos didácticos

- **La pestaña *Application* de las herramientas de desarrollo del navegador** — abre `tienda.ejemplo.com` en Chrome o Firefox, pulsa F12 y ve a *Application → Storage → Cookies* (en Firefox, *Storage → Cookies*). Ahí está la cookie de sesión real con todos sus atributos en columnas: `HttpOnly`, `Secure`, `SameSite`, `Domain`, `Expires`. Es el ejercicio de cinco minutos que convierte esta ficha en algo tangible: cambia un atributo en el servidor, recarga, y mira qué columna cambia. Prueba también a escribir `document.cookie` en la consola y comprobar que la cookie con `HttpOnly` no aparece.
- [PortSwigger Web Security Academy — Authentication labs](https://portswigger.net/web-security/authentication) — laboratorios gratuitos y ejecutables sobre gestión de sesiones. Los de tokens predecibles y de sesión que no se invalida al cerrar son exactamente los fallos de esta ficha, explotados con las manos, que es la forma en la que se entienden de verdad.
- [MDN — Using HTTP cookies](https://developer.mozilla.org/docs/Web/HTTP/Cookies) — la explicación didáctica de los atributos de cookie, con la tabla de qué envía el navegador con `Lax` frente a `Strict` en cada tipo de petición. Más fácil de leer que el RFC cuando lo que quieres es entender, no citar.

---

*En resumen: un token opaco no lleva información, solo es una llave hacia el estado guardado en el servidor — a cambio de una consulta por petición, obtienes revocación instantánea y cero filtración de datos.*
