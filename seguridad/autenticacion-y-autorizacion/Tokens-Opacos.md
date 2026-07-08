# Tokens opacos de sesión

## ¿Qué es?

Un token opaco (también llamado *reference token* o token de referencia) es una cadena aleatoria sin ningún significado por sí misma: no contiene datos, no se puede decodificar, no dice nada de quién eres. Es solo una **llave** que el servidor usa para buscar, en su propio almacén, a qué sesión pertenece. Es el modelo clásico de la cookie de sesión de toda la vida en la web.

## ¿Por qué existe?

Un token autocontenido como el [JWT](JWT.md) lleva la información dentro y firmada, lo que es cómodo pero tiene tres costes: cualquiera puede leer su contenido (solo va en Base64, no cifrado), no se puede revocar antes de que expire, y crece de tamaño según metes claims. El token opaco invierte el planteamiento: **no lleva nada**. Toda la información —quién es la usuaria, sus permisos, cuándo caduca— vive en el servidor, y el token es apenas un identificador.

Eso le da tres superpoderes: no filtra información (no hay nada que leer), se revoca al instante (borras la fila del almacén y la sesión muere) y ocupa poquísimo. Y hay un cuarto, más sutil: como el servidor lee el estado *vivo* en cada request, cualquier cambio —bajar un rol, desactivar la cuenta— surte efecto de inmediato, sin esperar a que caduque nada. El precio es que el servidor **sí tiene que recordar**, y por tanto consultar su almacén en cada request.

> Si ya conoces los JWT, piensa en un token opaco como el resguardo numerado del guardarropa de un teatro: el papelito no dice qué abrigo es tuyo ni de qué color, solo lleva un número; toda la información está en el mostrador, y quien tenga el mostrador puede anular tu resguardo cuando quiera.

## ¿Cuándo y para qué se usa?

- **Aplicaciones web con sesión de servidor**: un backoffice o panel de administración renderizado en servidor, donde el navegador guarda una cookie con el ID de sesión y el servidor mantiene el estado (ver [Sesiones vs Tokens](Sesiones-vs-Tokens.md)).
- **Sistemas que necesitan revocación inmediata**: banca, salud o cualquier contexto donde "cerrar sesión ahora" tras detectar un robo de cuenta no es negociable.
- **Access tokens opacos en OAuth2**: en lugar de emitir JWT, el servidor de identidad puede emitir tokens opacos y ofrecer un endpoint de *introspección* que las APIs consultan para validarlos.

## Lo mínimo que necesitas saber

**1. Anatomía: solo entropía**

Un token opaco no tiene estructura: son bytes aleatorios de una fuente criptográficamente segura, codificados como texto. No hay partes ni claims que extraer.

```csharp
// 32 bytes = 256 bits de entropía, generados con un CSPRNG
var bytes = RandomNumberGenerator.GetBytes(32);
string token = Convert.ToBase64String(bytes);
// Ejemplo: "9f8Ka2...==" — no dice absolutamente nada por sí mismo
```

**2. El servidor busca la sesión en su almacén**

```csharp
public async Task<Sesion?> ValidarAsync(string token)
{
    // El token es la clave; toda la información está guardada en el servidor
    var sesion = await _store.BuscarAsync(Hash(token));   // Redis, BD, caché...
    if (sesion is null || sesion.Expirada) return null;
    return sesion;   // aquí están el UsuarioId, los roles, la caducidad...
}
```

**3. La cookie que lo transporta en la web**

```csharp
Response.Cookies.Append("session_id", token, new CookieOptions
{
    HttpOnly = true,     // JavaScript no puede leerla (mitiga XSS)
    Secure = true,       // solo viaja por HTTPS
    SameSite = SameSiteMode.Lax   // reduce el riesgo de CSRF
});
```

**4. Revocar es borrar**

```csharp
// Cerrar sesión, expulsar tras robo de cuenta, invalidar todo un dispositivo:
await _store.EliminarAsync(Hash(token));
// La próxima request con ese token no encuentra nada → 401 al instante
```

**5. Introspección (OAuth2)**

Cuando un token opaco tiene que validarlo un servicio distinto del que lo emitió, ese servicio no puede leerlo (es opaco): pregunta al emisor mediante el endpoint de introspección estándar ([RFC 7662](https://datatracker.ietf.org/doc/html/rfc7662)), que responde si el token está activo y a quién pertenece.

```http
POST /introspect
token=9f8Ka2...   →   { "active": true, "sub": "42", "scope": "pedidos:leer" }
```

## Lo que NO hace

- **No se valida solo** — a diferencia de un JWT, no puedes comprobar un token opaco mirándolo: siempre hace falta una consulta al almacén (o al endpoint de introspección). Es un viaje de ida y vuelta más por request.
- **No escala sin estado compartido** — como la sesión vive en el servidor, varias instancias necesitan un almacén común (Redis, base de datos) o *sticky sessions*; no basta con validar una firma en local como con un JWT.
- **No lleva información utilizable por el cliente** — si el frontend necesita saber el nombre o el rol de la usuaria, no puede sacarlo del token; tiene que pedirlo a una API aparte.

## Buenas prácticas avanzadas

- **Exige entropía suficiente y una fuente criptográfica** — el token es lo único que separa a un atacante de una sesión ajena, así que debe ser imposible de adivinar o iterar: mínimo 128 bits (mejor 256) desde un CSPRNG (`RandomNumberGenerator`), nunca `Guid.NewGuid()` ni `Random`, que no están pensados para secretos.
- **Guarda en el almacén solo el hash del token, no el token en claro** — igual que con las contraseñas: si te filtran la base de datos de sesiones, un hash no permite secuestrar sesiones vivas. Aquí, además, un SHA-256 rápido es lo *correcto*, no una concesión: como el token ya tiene entropía alta y es imposible de adivinar, no necesita un hash lento tipo bcrypt o PBKDF2. Esos existen para las contraseñas —que tienen poca entropía y hay que encarecer su fuerza bruta a propósito—; aplicarlos a un token aleatorio solo añadiría latencia inútil en cada request.
- **Rota el identificador al autenticar** — genera un token nuevo justo después del login en lugar de reutilizar el que la usuaria tenía como anónima. Evita la *session fixation*, donde un atacante fija un ID de sesión conocido antes del login y luego lo aprovecha ya autenticado.
- **Combina caducidad deslizante con una absoluta** — renueva la ventana de expiración con la actividad, pero impón un tope absoluto tras el cual la sesión muere pase lo que pase; así una sesión activa no vive para siempre.
- **Añade un prefijo o marca al token para detectarlo si se filtra** — un prefijo reconocible (estilo `sess_...`) permite que escáneres de secretos y sistemas de detección identifiquen tokens filtrados en logs o repositorios, y de paso descarta de inmediato entradas con formato inválido antes de tocar el almacén.
- **Aprovecha el estado para gestionar sesiones concurrentes** — como cada sesión es una fila en el almacén, tienes casi gratis lo que a un JWT le cuesta sangre: listar los dispositivos activos de una usuaria, cerrar la sesión de uno solo o hacer un "cerrar sesión en todos los dispositivos" borrando todas sus filas. Modela la tabla pensando en ello (una fila por sesión, con identificador de usuaria, dispositivo y última actividad) y esa funcionalidad sale casi sola.

## Recursos didácticos divertidos

La [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) es la referencia práctica sobre entropía, atributos de cookie y ciclo de vida de sesión, con ejemplos concretos de qué hacer y qué no.

---

*En resumen: un token opaco no lleva información, solo es una llave hacia el estado guardado en el servidor — a cambio de una consulta por request, obtienes revocación instantánea y cero filtración de datos.*
