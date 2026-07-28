# JWT (JSON Web Token)

## ¿Qué es?

Un JWT (*JSON Web Token*) es un **formato de token**: tres bloques de texto en base64url separados por puntos, donde el bloque del medio son datos en JSON y el último es una **firma** que permite comprobar que nadie los ha alterado desde que se emitieron.

No es un protocolo ni un sistema de login: es el sobre. Quién lo emite y con qué flujo lo cubren [OAuth2](OAuth2.md) y [OpenID Connect](OpenID-Connect.md); qué se decide con lo que va dentro, [Autenticación vs Autorización](Autenticacion-vs-Autorizacion.md). Aquí se explica el sobre.

## ¿Por qué existe?

Cuando una API acepta tokens en lugar de mantener sesiones en memoria (la comparación completa está en [Sesiones vs Tokens](Sesiones-vs-Tokens.md)), hace falta un formato concreto para ese token. El problema a resolver es este: la API `api.tienda.ejemplo.com` recibe una petición para consultar el pedido **#4711** y necesita saber que viene de la clienta **42** y no de alguien que se ha inventado ese número. Si el token es un identificador opaco, la única forma de averiguarlo es preguntar a una base de datos en cada petición.

JWT elimina esa consulta metiendo la información **dentro** del token y firmándola. La API lee `sub: "42"` del propio token y comprueba la firma con una clave; si la firma cuadra, esos datos son exactamente los que puso el emisor.

> Piensa en un JWT como un documento con un sello de lacre. El texto está a la vista: cualquiera que lo tenga en la mano puede leerlo entero. Lo que el lacre garantiza no es el secreto, es que **nadie lo ha modificado**: al cambiar una sola letra, el sello deja de cuadrar.

## ¿Cuándo y para qué se usa?

- **APIs que validan sin consultar nada.** La API de la tienda comprueba la firma del token en local y responde. Sin viaje a la base de datos, sin tabla `Sesiones` en el camino crítico.
- **Varios servicios que confían en un mismo emisor.** El servicio de facturación acepta tokens emitidos por `https://api.tienda.ejemplo.com` sin tener que volver a autenticar a la clienta contra nada.
- **Como pieza de OAuth2 y OpenID Connect.** El `id_token` de OIDC es, por especificación, un JWT; los *access tokens* de OAuth2 suelen serlo también, aunque no está obligado.

---

## Lo primero: un JWT no está cifrado, está firmado

Esto suele contarse al final y debería contarse aquí, porque de él sale la regla más importante de toda la ficha.

Firmar y cifrar son cosas distintas. La firma garantiza **integridad** (nadie lo ha tocado) y **autenticidad** (lo emitió quien dice). No garantiza **confidencialidad**. El *payload* de un JWT es texto plano codificado en base64url, y base64url no es cifrado: es una forma de escribir bytes con caracteres seguros para URLs. Se deshace sin ninguna clave.

Tomemos el bloque central de un token real de la tienda y decodifiquémoslo:

```bash
echo 'eyJpc3MiOiJodHRwczovL2FwaS50aWVuZGEuZWplbXBsby5jb20iLCJzdWIiOiI0MiIsImF1ZCI6ImFwaS50aWVuZGEuZWplbXBsby5jb20iLCJleHAiOjE3ODUyMzQ0MjAsIm5iZiI6MTc4NTIzMzUyMCwiaWF0IjoxNzg1MjMzNTIwLCJqdGkiOiI5YzFmNGUyYS03YjAzLTRkNTgtOGE2MS0zZjBlMmQ1YzdiMTkiLCJyb2xlIjoiQ2xpZW50ZSJ9' | base64 -d
```

Salida, sin haber usado ninguna clave ni haber pedido permiso a nadie:

```json
{"iss":"https://api.tienda.ejemplo.com","sub":"42","aud":"api.tienda.ejemplo.com","exp":1785234420,"nbf":1785233520,"iat":1785233520,"jti":"9c1f4e2a-7b03-4d58-8a61-3f0e2d5c7b19","role":"Cliente"}
```

De aquí sale la regla, y no admite matices: **nunca metas en un JWT nada que no publicarías**. Ni contraseñas ni sus hashes, ni el correo si en tu contexto es un dato protegido, ni el DNI, ni el importe de un pedido, ni notas internas sobre la clienta. Todo lo que entra en el *payload* queda legible para cualquiera que llegue a ver el token: la propia usuaria en la consola del navegador, un proxy que registre la cabecera `Authorization`, el sistema de logs, quien reciba una captura de pantalla en un ticket de soporte.

El único identificador seguro es uno que **no signifique nada por sí solo**: `sub: "42"` es la clave primaria de la tabla `Clientes`, y saberla no revela quién es esa persona.

```json
// ❌ El correo y el teléfono quedan legibles para cualquiera con el token
{ "sub": "42", "email": "ana.ruiz@correo.ejemplo.com", "telefono": "600123456" }

// ✅ Un identificador interno; los datos personales se consultan en la BD si hacen falta
{ "sub": "42", "role": "Cliente" }
```

Existe un formato hermano que sí cifra, JWE (*JSON Web Encryption*), pero es poco habitual en APIs y no resuelve el problema de fondo: si el dato es sensible, lo natural es no ponerlo en un token que viaja en cada petición.

## Anatomía: un token real, desglosado

Este es un token completo emitido por la API de la tienda para la clienta 42. Son tres bloques separados por dos puntos literales:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InRpZW5kYS0yMDI2LTA3In0.eyJpc3MiOiJodHRwczovL2FwaS50aWVuZGEuZWplbXBsby5jb20iLCJzdWIiOiI0MiIsImF1ZCI6ImFwaS50aWVuZGEuZWplbXBsby5jb20iLCJleHAiOjE3ODUyMzQ0MjAsIm5iZiI6MTc4NTIzMzUyMCwiaWF0IjoxNzg1MjMzNTIwLCJqdGkiOiI5YzFmNGUyYS03YjAzLTRkNTgtOGE2MS0zZjBlMmQ1YzdiMTkiLCJyb2xlIjoiQ2xpZW50ZSJ9.YEMsdLKcl3a453y1eavEVlhNaIRlXZSHpPrb5TvVLx1pLlygsVuDkRd6ZIleMgbtCm0zO9pg8bup2efGtYYvsgwXiNem0Q2jzuwJMzuyjgNeuE2VVyeeZQYy6cfPEUVDCeAleTU-4to3rFAhX88wASLnnPsGhwccE3_m4QVzIJMS2xAjp2wgCPsV8Ddu5wYs5kHERwSWo4G8FZLfiL9MyEaKfnoE8ZuxoEWL7EA3NSqMPOpxGEICK_0HxCkzvSBzhnPYsdD0UrKshsdPu62tCifRYIhK4B8JgoufYKxKcPDtkMBVZ9mMs_bT7KvbsRoPh-WMKA5BYV9m_egOtE-LhA
```

Son 671 caracteres en una sola línea, sin saltos. Partido por sus dos puntos separadores queda así —el `.` de la izquierda es el punto literal que hay en el token, no una viñeta:

```
  eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InRpZW5kYS0yMDI2LTA3In0     ← cabecera, 67 caracteres
.                                                                         ← punto separador nº 1
  eyJpc3MiOiJodHRwczovL2FwaS50aWVuZGEuZWplbXBsby5jb20iLCJzdWIiOiI0Mi...    ← payload, 260 caracteres
.                                                                         ← punto separador nº 2
  YEMsdLKcl3a453y1eavEVlhNaIRlXZSHpPrb5TvVLx1pLlygsVuDkRd6ZIleMgbt...     ← firma, 342 caracteres
```

**La cabecera** describe cómo está firmado el token. Decodificada:

```bash
echo 'eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InRpZW5kYS0yMDI2LTA3In0=' | base64 -d
```

```json
{"alg":"RS256","typ":"JWT","kid":"tienda-2026-07"}
```

- `alg` — el algoritmo con el que se firmó. **Es un dato del token, no una verdad**: quien lo envía puede escribir lo que quiera ahí. De esto va toda la sección de ataques.
- `typ` — el tipo. Casi siempre `JWT`; sirve para distinguirlo de otros usos de la misma estructura.
- `kid` (*key id*) — qué clave se usó, para poder tener varias a la vez y rotarlas.

**El *payload*** son los *claims*: las afirmaciones sobre la identidad. Es el bloque que ya hemos decodificado arriba. Aquí lleva los siete claims registrados por la especificación más un `role` propio, que en la tienda vale `Cliente`, `Empleado` o `Administrador` (cómo se traduce eso en permisos lo cubre [RBAC y Claims](RBAC-y-Claims.md)).

**La firma** son los 342 caracteres finales: el resultado de firmar la cadena literal `cabecera.payload` —los dos bloques ya codificados, con el punto en medio— con la clave privada del emisor. Cambiar un solo carácter de cualquiera de los dos bloques anteriores hace que la firma deje de cuadrar.

### ¿Por qué base64url y no base64?

El alfabeto de base64 clásico usa `+`, `/` y `=`, y los tres se rompen en una URL: `+` se interpreta como espacio en una *query string*, `/` parece un separador de ruta y `=` es el separador de parámetros. Base64url sustituye `+` por `-`, `/` por `_` y **elimina el relleno de `=`** del final.

Se ve en el token de arriba: la firma contiene `TU-4to3` y `E3_m4QVzIJMS`. Esos `-` y `_` serían `+` y `/` en base64 normal.

Un efecto práctico: como el relleno se elimina, `base64 -d` protesta cuando la longitud del bloque no es múltiplo de 4. Por eso la cabecera de arriba lleva un `=` añadido a mano y el *payload* no lo necesita. No es un error del token: es el relleno que la especificación manda quitar.

## Los claims registrados, uno a uno y con su trampa

La especificación reserva siete nombres. Ninguno es obligatorio y **el formato no valida ninguno**: validarlos es responsabilidad de quien recibe el token. Esta es la diferencia que más confusión causa — que un claim exista no significa que alguien lo esté comprobando.

| Claim | Significa | Qué se rompe si no lo validas |
|---|---|---|
| `iss` (*issuer*) | Quién emitió el token: `https://api.tienda.ejemplo.com` | Aceptas tokens de cualquier emisor cuya clave conozcas, incluido un proveedor de identidad de pruebas que alguien dejó configurado |
| `sub` (*subject*) | A quién identifica: `42`, la clienta | Sin `sub` el token no identifica a nadie; si lo lees sin validar la firma, cualquiera pone `sub: 1` |
| `aud` (*audience*) | Para qué servicio es: `api.tienda.ejemplo.com` | **El olvido más frecuente.** Ver debajo |
| `exp` (*expiration*) | Instante a partir del cual no vale (segundos Unix) | **El segundo olvido más frecuente.** El token vale para siempre |
| `nbf` (*not before*) | Instante antes del cual no vale | Un token emitido para activarse más tarde se puede usar ya |
| `iat` (*issued at*) | Cuándo se emitió | Pierdes la capacidad de rechazar tokens «demasiado viejos» aunque no hayan expirado |
| `jti` (*JWT ID*) | Identificador único del token | No puedes detectar reutilización ni construir una lista de revocación |

**`aud`: el token válido que no era para ti.** Supongamos que el emisor `https://api.tienda.ejemplo.com` emite tokens para dos servicios: la API de la tienda (`aud: "api.tienda.ejemplo.com"`) y un servicio de facturación (`aud: "facturacion.tienda.ejemplo.com"`). Los dos están firmados con la misma clave, así que **los dos tienen firma válida para los dos servicios**. Si la API no comprueba `aud`, un token que la clienta obtuvo para el servicio de facturación —de permisos más restringidos, o al revés— entra en la API como si fuera suyo. La firma no protege de esto: es correcta. Lo único que separa un token del otro es el claim que nadie miró.

**`exp`: expira quien lo comprueba.** Un JWT no caduca solo. No hay proceso que lo borre ni servidor que lo marque como usado; el número en `exp` es texto dentro del token. Si el verificador no compara ese número con la hora actual, el token sirve indefinidamente. Y como el token es autocontenido, ese `exp` de hace tres meses seguirá teniendo firma perfectamente válida.

## Cómo se valida un token, en el orden correcto

El orden importa porque cada paso presupone el anterior. No tiene sentido mirar `aud` de un *payload* cuya firma no has comprobado: podría haberlo escrito el atacante.

1. **La firma**, con la clave correcta.
2. **El algoritmo esperado**, declarado por ti de antemano, no leído del token.
3. **`iss` y `aud`**, comparados con valores fijos.
4. **Las fechas** `exp` y `nbf`.

En .NET, `AddJwtBearer` hace los cuatro pasos si se lo configuras. Cada opción, comentada:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        // De dónde saca las claves públicas: descubre /.well-known/openid-configuration
        // y desde ahí el JWKS. Evita tener claves copiadas en la configuración.
        options.Authority = "https://api.tienda.ejemplo.com";

        options.TokenValidationParameters = new TokenValidationParameters
        {
            // 1. Firma: exige que la clave del token esté entre las conocidas
            ValidateIssuerSigningKey = true,

            // 2. Algoritmo: la lista la decides TÚ, no el campo alg del token
            ValidAlgorithms = new[] { SecurityAlgorithms.RsaSha256 },

            // 3. Emisor y audiencia, comparados con cadenas fijas
            ValidateIssuer = true,
            ValidIssuer = "https://api.tienda.ejemplo.com",
            ValidateAudience = true,
            ValidAudience = "api.tienda.ejemplo.com",

            // 4. Fechas: exp y nbf frente al reloj del servidor
            ValidateLifetime = true,
            RequireExpirationTime = true,      // rechaza tokens sin exp en absoluto

            // Margen para relojes desincronizados. El defecto son 5 minutos,
            // que es mucho para un token de 15: bajarlo es lo correcto.
            ClockSkew = TimeSpan.FromSeconds(30)
        };
    });
```

Y el aviso que hay que dar por escrito: **`ValidateIssuerSigningKey` no se pone en `false` nunca**. Con esa opción desactivada la librería deja de exigir que la clave firmante sea una de las que conoce, y el mensaje de error desaparece — que es justo lo que buscaba quien la tocó. También desaparece la única garantía de que el token lo emitió tu servidor. Lo mismo vale para `ValidateAudience = false`, que es la forma habitual de silenciar el error `IDX10214` en lugar de arreglar la configuración.

## `alg: none` y la confusión de algoritmos

Estas son las dos vulnerabilidades clásicas del formato, y las dos nacen del mismo error de diseño: **el token dice cómo verificarse a sí mismo**. Es como aceptar una entrada que trae impresas las instrucciones de cómo comprobar si es falsa.

### Ataque 1: `alg: none`

La especificación admite el valor `none` para tokens sin firma, pensado para casos donde la integridad se garantiza por otra vía. Una librería ingenua lee `alg` de la cabecera, ve `none` y concluye que no hay nada que verificar.

El atacante toma el token legítimo, reescribe la cabecera, se sube a `Administrador` y **borra la firma**, dejando el punto final:

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJpc3MiOiJodHRwczovL2FwaS50aWVuZGEuZWplbXBsby5jb20iLCJzdWIiOiI0MiIsImF1ZCI6ImFwaS50aWVuZGEuZWplbXBsby5jb20iLCJleHAiOjE3ODUyMzQ0MjAsIm5iZiI6MTc4NTIzMzUyMCwiaWF0IjoxNzg1MjMzNTIwLCJqdGkiOiI5YzFmNGUyYS03YjAzLTRkNTgtOGE2MS0zZjBlMmQ1YzdiMTkiLCJyb2xlIjoiQWRtaW5pc3RyYWRvciJ9.
```

La cabecera decodificada es `{"alg":"none","typ":"JWT"}` y el *payload* ahora dice `"role":"Administrador"`. Construirlo no requiere ninguna clave: es concatenar dos base64url y un punto.

### Ataque 2: confusión de algoritmos (RS256 → HS256)

Más sutil y más peligroso, porque el token sí lleva firma y sí la verifica.

Con RS256 la clave privada firma y la **clave pública** verifica. Con HS256 hay un único secreto compartido que sirve para las dos cosas. Si el verificador acepta los dos algoritmos y elige según el campo `alg`, pasa lo siguiente: el atacante cambia `RS256` por `HS256`, y firma el token con HMAC-SHA256 **usando la clave pública como si fuera el secreto**. El verificador lee `alg: HS256`, coge «la clave» que tiene configurada —que es la pública— y comprueba el HMAC. Cuadra.

La clave pública no es un secreto por definición: está publicada en el JWKS, a un `curl` de distancia.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InRpZW5kYS0yMDI2LTA3In0.eyJpc3MiOiJodHRwczovL2FwaS50aWVuZGEuZWplbXBsby5jb20iLCJzdWIiOiI0MiIsImF1ZCI6ImFwaS50aWVuZGEuZWplbXBsby5jb20iLCJleHAiOjE3ODUyMzQ0MjAsIm5iZiI6MTc4NTIzMzUyMCwiaWF0IjoxNzg1MjMzNTIwLCJqdGkiOiI5YzFmNGUyYS03YjAzLTRkNTgtOGE2MS0zZjBlMmQ1YzdiMTkiLCJyb2xlIjoiQWRtaW5pc3RyYWRvciJ9.OZrZJ7FFXC0jS4_z9SkvjixE2VdAq949JUGQT15p8Rc
```

Fíjate en dos detalles. Primero, el `kid` sigue siendo `tienda-2026-07`: el atacante quiere que el verificador busque exactamente esa clave. Y segundo, la firma mide 43 caracteres en lugar de 342, porque un HMAC-SHA256 son 32 bytes y una firma RSA de 2048 bits son 256.

### La defensa

Es una sola frase: **no confíes nunca en el `alg` del token**. Declara de antemano qué algoritmos aceptas y verifica con ese, ignorando lo que el token diga traer.

```csharp
// ❌ La librería decide según el alg del token; ambos ataques funcionan
new TokenValidationParameters { IssuerSigningKey = clave };

// ✅ Solo RS256, decidido por el servidor. Un token con alg none o HS256
//    se rechaza antes de mirar su firma.
new TokenValidationParameters
{
    ValidAlgorithms = new[] { SecurityAlgorithms.RsaSha256 },
    ValidateIssuerSigningKey = true
};
```

Con `ValidAlgorithms` fijado, el token de `alg: none` falla en el primer paso y el de HS256 también, sin que su firma llegue a comprobarse. Las librerías mantenidas —incluida la de .NET— rechazan `none` de serie, pero la lista explícita es lo que hace que la defensa no dependa de la versión que tengas instalada.

## HS256 frente a RS256

Los dos firman; la diferencia es **quién puede verificar y quién puede emitir**.

| | HS256 (simétrico, HMAC) | RS256 (asimétrico, RSA) |
|---|---|---|
| Quién puede firmar | Cualquiera con el secreto | Solo quien tiene la clave privada |
| Quién puede verificar | Cualquiera con el secreto | Cualquiera, con la clave pública |
| Secretos a distribuir | El mismo secreto en el emisor **y en cada verificador** | La clave privada solo en el emisor; la pública se publica |
| Consecuencia directa | **Cada servicio que verifica también puede emitir** | El emisor emite; los demás solo comprueban |
| Rotación de claves | Hay que actualizar todos los servicios a la vez | Se publica la clave nueva en el JWKS; los verificadores la recogen |
| Coste de cómputo | Muy bajo | Más alto al firmar, bajo al verificar |
| Cuándo elegirlo | Un solo servicio emite y verifica sus propios tokens | Varios servicios verifican, o el emisor es un proveedor externo |

El argumento decisivo es la cuarta fila. Con HS256, el servicio de facturación necesita el secreto para verificar tokens; con ese mismo secreto puede fabricar un token con `role: "Administrador"` y `sub: "42"` que la API de la tienda aceptará como legítimo. No hay forma criptográfica de distinguirlo. Con RS256 eso es imposible: la clave pública solo sirve para comprobar.

La regla práctica: **si el token cruza el límite de un servicio, RS256**. Si la API emite y consume sus propios tokens y nadie más los mira, HS256 es suficiente y más simple, siempre con un secreto de al menos 32 bytes aleatorios.

Existe una tercera opción, **ES256** (curvas elípticas): mismas propiedades asimétricas que RS256 con claves y firmas mucho más cortas —64 bytes de firma frente a 256—, lo que ayuda cuando el tamaño del token importa.

## JWKS y `kid`: de dónde sale la clave pública

Con RS256, el verificador necesita la clave pública del emisor. Copiarla en la configuración de cada servicio funciona hasta el día que hay que cambiarla. La alternativa estándar es que el emisor la publique en un endpoint:

```bash
curl https://api.tienda.ejemplo.com/.well-known/jwks.json
```

Un JWKS (*JSON Web Key Set*) es una lista de claves públicas en JSON:

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "tienda-2026-07",
      "alg": "RS256",
      "n": "y-eRktbEG1huPwvujQgqkFoP6M_YE7jCazE6W6AmBBbrYvGPgGs43l8eCHCuGsstTQ_8m7il5BXYYA62Qv6IWZrjp0qkemUPadDErf6T0Nh6J73icv9oltOVEfh_foBLXmNQrkYsDr0h2enQpmGsQ8VIizo5O6K1hSJptP8oPnu5lF9FzPKyCZ150xWyU6FWsgoWaOmBO3TpEA-7xyhUkDFxTjFOtRgRDApAG-7WsBanHadjuOb1gBIVl0iwKmggxsIp9N4wWxuouva8GsJh_1upIoOJHo4S7yl55Ov_yaA-kj_tn42V0bgj4mIZ6KeSnPbMRp9Te0kiDYlvxWTjiw",
      "e": "AQAB"
    }
  ]
}
```

`n` es el módulo RSA y `e` el exponente: entre los dos son la clave pública. `use: "sig"` dice que es para firmas y no para cifrar.

El `kid` es lo que une las dos piezas. La cabecera del token dice `"kid":"tienda-2026-07"`, el verificador busca en el JWKS la entrada con ese `kid` y verifica con ella. Y ahí está la utilidad real: **la rotación sin cortar el servicio**. El emisor publica la clave nueva junto a la vieja, con `kid` distinto:

```json
{ "keys": [ { "kid": "tienda-2026-07", "...": "..." },
            { "kid": "tienda-2026-10", "...": "..." } ] }
```

Empieza a firmar los tokens nuevos con `tienda-2026-10` y deja de emitir con la vieja. Los tokens ya en circulación siguen validándose porque su clave sigue publicada. Cuando el último haya expirado —minutos, si los tokens son cortos— se retira la clave vieja del JWKS. En ningún momento hay usuarias con tokens que no se pueden verificar.

Sin `kid`, cambiar de clave es un cambio simultáneo y coordinado, y todo token en vuelo en ese instante se cae.

Un detalle de operación: las librerías **cachean el JWKS**, típicamente unos minutos, con reintento cuando aparece un `kid` desconocido. Publica la clave nueva un rato antes de empezar a usarla, o los primeros tokens llegarán a verificadores que aún no la tienen.

## El tamaño del token y el error `431`

Un JWT viaja en la cabecera `Authorization` de **cada petición**. No es un coste que se pague una vez al iniciar sesión: es un coste por llamada, en todas.

El token de esta ficha mide 671 caracteres. Unos cientos de bytes es lo normal y no molesta a nadie. El problema aparece cuando alguien decide meter en el *payload* la lista de grupos de un directorio corporativo:

```json
{ "sub": "42", "groups": ["tienda-ventas-es", "tienda-ventas-pt", "..." ] }
```

Con 150 grupos de unos 45 caracteres de media son 6,7 KB de JSON. Base64url añade un tercio: **unos 9 KB** solo en el *payload*, más la cabecera, la firma y el prefijo `Bearer `. La línea de cabecera resultante supera los 9 KB.

Y ahí es donde se rompe, de forma distinta según quién esté delante:

| Componente | Límite por defecto | Qué devuelve al superarlo |
|---|---|---|
| nginx | 8 KB por búfer de cabecera | `400 Bad Request`, sin explicación en la respuesta |
| Node.js / Express | 16 KB de cabeceras | `431 Request Header Fields Too Large` |
| Kestrel (.NET) | 32 KB de cabeceras en total | `431 Request Header Fields Too Large` |

Lo característico de este fallo es **cuándo aparece**. En desarrollo la cuenta de pruebas tiene tres grupos y el token mide 800 bytes; todo funciona. En producción, la primera persona con muchos grupos recibe un `400` que ni siquiera menciona el tamaño, y en el log de la aplicación no hay nada porque la petición no llegó a entrar. Se diagnostica mirando el log de acceso del proxy, no el de la aplicación.

La solución es no meter en el token lo que se puede consultar: el `sub` y el `role` bastan para decidir, y los grupos se leen de la base de datos cuando de verdad hacen falta.

## La revocación, que el formato no resuelve

Este es el precio de no consultar nada: **un JWT es válido hasta que expira, y no hay nada que borrar**.

El escenario concreto. Un empleado con `role: "Empleado"` sale de la empresa a las 10:15. A las 10:16 se desactiva su cuenta en la tabla `Clientes` y se le quitan los permisos. Su token, emitido a las 10:12 con `exp` a las 10:27, sigue teniendo firma válida, `iss` correcto, `aud` correcto y fecha en vigor. La API lo acepta durante los once minutos siguientes porque **no tiene forma de saber que la cuenta ya no existe**: no consulta nada, que era exactamente el objetivo.

Hay tres formas de convivir con esto:

- **Tokens de vida corta más un mecanismo de renovación.** Es la respuesta estándar: el token de acceso vale minutos y se renueva con un *refresh token* que sí es revocable, porque ese sí está en la base de datos. La ventana de exposición pasa a ser la vida del token de acceso. Lo desarrolla [Modelo JWT + Refresh Token](JWT-Refresh.md).
- **No usar JWT para la sesión.** Un token opaco se invalida borrando una fila. Ver [Tokens opacos de sesión](Tokens-Opacos.md) y la comparación en [JWT + Refresh vs Tokens opacos](JWT-Refresh-vs-Tokens-Opacos.md).
- **Lista de revocación por `jti`**, como parche. Se guardan los `jti` revocados en Redis con TTL igual al `exp` restante, y cada petición comprueba si el token está en la lista. Funciona, pero conviene ver lo que cuesta: **has reintroducido la consulta por petición que motivaba usar JWT**. Ahora tienes la complejidad del token autocontenido *y* el estado en el servidor. Si acabas aquí, la pregunta honesta es si el token opaco no era mejor desde el principio.

## Cuándo NO usar un JWT

- **Para la sesión de una aplicación web propia con un solo dominio.** Si el frontend `tienda.ejemplo.com` y la API son tuyos, una cookie de sesión es más simple, más pequeña, la gestiona el navegador y —lo importante— se cierra de verdad borrando una fila. JWT resuelve un problema (verificar sin estado compartido) que este caso no tiene. La comparación completa está en [Sesiones vs Tokens](Sesiones-vs-Tokens.md).
- **Para guardar datos que cambian.** Si se asciende a la clienta 42 a `Empleado`, su token sigue diciendo `Cliente` hasta que expire. Cualquier claim que pueda cambiar antes de la expiración es un dato desactualizado que la API tratará como cierto.
- **Para nada que deba ser secreto.** Por lo de la primera sección: no está cifrado.
- **Como identificador de sesión de larga duración.** Un JWT con `exp` a treinta días es un permiso de acceso de treinta días que no se puede retirar. Si necesitas poder cerrar una sesión en el momento, el formato no te lo va a dar.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `IDX10223: Lifetime validation failed. The token is expired. ValidTo: '07/28/2026 10:27:00'` | El `exp` ya pasó. Normal si el token es corto y el cliente no lo renueva; ver [JWT + Refresh](JWT-Refresh.md) |
| `IDX10214: Audience validation failed. Audiences: 'facturacion.tienda.ejemplo.com'. Did not match: validationParameters.ValidAudience: 'api.tienda.ejemplo.com'` | El token es legítimo pero se emitió para otro servicio. La tentación es `ValidateAudience = false`; el arreglo es pedir el token con la audiencia correcta |
| `IDX10501: Signature validation failed. Unable to match key: kid: 'tienda-2026-10'` | El verificador tiene un JWKS cacheado que no incluye la clave nueva, o se empezó a firmar con ella antes de publicarla |
| `IDX10503: Signature validation failed` con la clave correcta | El `alg` esperado no coincide con el del token, o el token se truncó al copiarlo (el `Bearer ` de más es otro clásico) |
| Funciona en desarrollo y falla en producción con token expirado | Relojes desincronizados. El *clock skew* es el margen que el verificador tolera entre su reloj y el del emisor; el defecto de .NET son 5 minutos, y si un servidor va 6 minutos adelantado rechaza tokens recién emitidos. Se arregla con NTP en las dos máquinas, no subiendo `ClockSkew` |
| El correo de la clienta aparece en la consola del navegador y en los logs del proxy | Se metió en el *payload*. El token no cifra: lo ve todo el que lo toca |
| Un usuario accede a datos de otro servicio con un token válido | Se validó la firma pero no `aud` |
| Se puso `ValidateIssuerSigningKey = false` porque «así deja de fallar» | Deja de fallar porque deja de comprobar que la firma la hizo tu emisor. El error real suele ser un `Authority` mal escrito o un JWKS inalcanzable |

## Buenas prácticas avanzadas

- **Declara los algoritmos aceptados aunque tu librería ya lo haga.** `ValidAlgorithms = ["RS256"]` es una línea que convierte la defensa contra `alg: none` y la confusión de algoritmos en algo que no depende de la versión instalada ni de que nadie cambie un valor por defecto en la próxima actualización. Es la línea más rentable de toda la configuración.
- **Usa `Authority` y JWKS en lugar de una clave copiada en la configuración.** Una clave pública pegada en `appsettings.json` funciona hasta la primera rotación, y entonces hay que desplegar todos los servicios el mismo día. Con `Authority`, el verificador descubre el JWKS, lo cachea y recoge las claves nuevas solo. Además evita la vía por la que la clave pública acaba usada por error como secreto HMAC.
- **Trata el tamaño del *payload* como una decisión de diseño, no como un detalle.** Cada claim se paga en cada petición de cada usuaria. Antes de añadir uno, pregunta si el verificador puede consultarlo cuando lo necesite; casi siempre sí. Un token de más de 1 KB es una señal de que hay datos dentro que no deberían estar.
- **Pon `RequireExpirationTime = true` y baja el `ClockSkew`.** Sin lo primero, un token sin `exp` pasa la validación de vigencia sin más. Y los 5 minutos de margen por defecto son más que la vida entera de un token de 15 minutos bien configurado: los reduce a 30 segundos y arregla los relojes con NTP, que es el problema de verdad.
- **Prueba tú los ataques contra tu propia API antes de que los pruebe otro.** Coge un token válido, reescribe el *payload* subiendo el rol, deja la firma vieja y llama: debe salir `401`. Repite con `alg: none` y firma vacía, y con el token de otra audiencia. Son tres peticiones con `curl` y contestan de forma definitiva a lo que ninguna revisión de código responde con certeza.
- **Registra el `jti` de los tokens que aceptas, aunque no vayas a revocar nada.** No cuesta nada y es lo que permite responder a «¿este token se usó desde otra IP?» durante un incidente. Sin `jti` en los logs, dos usos del mismo token son indistinguibles de dos sesiones distintas.

## Documentación oficial

- [RFC 7519 — JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519) — la definición del formato. La sección 4.1 es la lista de los claims registrados con su semántica exacta; ve ahí cuando dudes de si `aud` admite un array (sí) o de qué formato tienen las fechas.
- [RFC 7515 — JSON Web Signature](https://datatracker.ietf.org/doc/html/rfc7515) — cómo se construye la firma. El apéndice con el cálculo paso a paso es lo que aclara qué se firma exactamente (la cadena `cabecera.payload`, no el JSON).
- [RFC 7517 — JSON Web Key](https://datatracker.ietf.org/doc/html/rfc7517) — el formato JWKS. Consúltala para saber qué significa cada campo (`kty`, `use`, `n`, `e`) cuando tengas que leer o publicar un endpoint de claves.
- [RFC 8725 — JWT Best Current Practices](https://datatracker.ietf.org/doc/html/rfc8725) — **la más útil y la que menos gente conoce.** Es un documento corto y en prosa que enumera los ataques reales, `alg: none` y la confusión de algoritmos incluidos, con la mitigación de cada uno. Si solo vas a leer una de estas cuatro, lee esta.
- [Autenticación JWT bearer en ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authentication/configure-jwt-bearer-authentication) — la referencia de `AddJwtBearer` y de todas las propiedades de `TokenValidationParameters`, con sus valores por defecto. Ve ahí para comprobar qué se valida sin que lo pidas.

## Recursos didácticos

- [jwt.io](https://jwt.io) — pega cualquier token y verás las tres partes separadas, decodificadas y coloreadas, con la firma verificada en el navegador. Es la forma más rápida de convencerse de que el *payload* no está cifrado: pega el token de esta ficha y lee el `sub` sin tener ninguna clave.
- [JWT attacks — Web Security Academy de PortSwigger](https://portswigger.net/web-security/jwt) — laboratorios interactivos donde explotas tú `alg: none`, la confusión de algoritmos y la inyección de `jwk`/`kid` contra una aplicación real. Hacerlo con las manos deja la lección de la sección de ataques mucho mejor asentada que leerla.
- [JWT Security Cheat Sheet de OWASP](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html) — el resumen práctico de qué validar y qué no meter dentro, en formato lista de comprobación para repasar antes de dar por cerrada una implementación.

---

*En resumen: un JWT es un JSON firmado que cualquiera puede leer y nadie puede modificar — así que no metas dentro nada secreto, no te fíes nunca del `alg` que trae, valida `aud` y `exp` además de la firma, y ten un plan para el rato en que un token revocado sigue siendo válido.*
