# Hashing en C#/.NET

## ¿Qué es?

El conjunto de APIs que .NET ofrece para hashear: `System.Security.Cryptography` para los hashes rápidos, `PasswordHasher<T>` para contraseñas, `RandomNumberGenerator` para todo lo que deba ser impredecible, y unos cuantos paquetes NuGet para lo que la plataforma no trae.

## ¿Por qué existe?

La teoría de las fichas anteriores hay que aterrizarla en código, y .NET tiene una peculiaridad que conviene conocer antes de abrir el gestor de paquetes: **trae de serie los hashes rápidos y una solución de contraseñas *first-party* (PBKDF2), pero no incluye bcrypt ni Argon2**. Para esos dos hay que evaluar paquetes de terceros, con su mantenimiento y su licencia, y eso es una decisión de arquitectura y no un `dotnet add package`. Saber qué hay en la caja evita las dos salidas malas: reinventar (mal) la rueda criptográfica, o arrastrar a producción una dependencia que nadie ha mirado.

> Regla de oro en criptografía: **no implementes primitivas tú misma**. Usa lo que trae la plataforma o librerías auditadas. Un PBKDF2 escrito a mano parece funcionar perfectamente mientras es inseguro, porque no hay ningún test que falle cuando el resultado es débil.

## ¿Cuándo y para qué se usa?

El ejemplo que recorre la ficha es la API en .NET de una **tienda online**, con una tabla `Clientes` (columna `PasswordHash`) y una tabla `Sesiones` (columna `TokenHash`). Para integridad, los ficheros de referencia son un `instalador.exe` y un `video-4gb.mp4`.

- **`SHA256` / `SHA512`** — integridad de ficheros, ETags, deduplicación, huellas de contenido, hashear tokens antes de guardarlos.
- **`PasswordHasher<Cliente>`** — almacenar contraseñas con lo que trae ASP.NET Core, sin dependencias externas.
- **`BCrypt.Net-Next` o un paquete Argon2** — cuando quieres bcrypt o Argon2id en lugar de PBKDF2.
- **`RandomNumberGenerator`** — sales, tokens de sesión y códigos de recuperación.

Esta ficha cubre **la parte de .NET**: qué API, qué paquete, qué licencia. El criterio para elegir entre hash rápido y lento está en [Contraseñas vs tokens de sesión](Contrasenas-Vs-Tokens-De-Sesion.md), la teoría de las KDF en [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md), y qué familia de hash elegir en [Hashes de propósito general](Hashes-De-Proposito-General.md).

---

## Hashes rápidos: los métodos estáticos

Desde .NET 5 existe la forma estática, que no crea ni libera objetos. Es la que hay que usar siempre que tengas los datos completos en memoria:

```csharp
using System.Security.Cryptography;
using System.Text;

byte[] contenido = Encoding.UTF8.GetBytes("contenido del documento");
byte[] hash = SHA256.HashData(contenido);          // 32 bytes; SHA512.HashData da 64

Console.WriteLine(Convert.ToHexString(hash));      // añade .ToLowerInvariant() si lo prefieres así
// 7B84C017E6DE4EC913DE08E115C80309D76F0AA72A84C28B42365852BEF74A1C
```

Fíjate en que no existe `SHA256.HashData("texto")`: pasas siempre bytes, y esa fricción es deliberada, porque **la codificación forma parte del hash**.

```csharp
SHA256.HashData(Encoding.UTF8.GetBytes("contenido del documento"));
// 7B84C017E6DE4EC913DE08E115C80309D76F0AA72A84C28B42365852BEF74A1C

SHA256.HashData(Encoding.Unicode.GetBytes("contenido del documento"));
// D8D02A9C8592830E5EC8FDE6B79BB6C112C4F7F3BF6334A52FD5984FB7C8A1D4
```

Mismo texto, dos digests sin nada en común. Si dos servicios hashean "lo mismo" y no coinciden, esto es lo primero que hay que mirar: fija UTF-8 por convenio y no uses `Encoding.Default`, que depende de la máquina. Decidir *qué* bytes entran al hash es el problema de canonicalización de [Hash criptográfico](Hash-Criptografico.md).

## Ficheros grandes: hashear el stream, no el `byte[]`

Con un `video-4gb.mp4` no puedes leer el fichero entero. `HashDataAsync` lo consume por bloques y mantiene el uso de memoria plano, frente al atajo que aparece en todos los ejemplos de internet:

```csharp
// ✅ Unos KB en memoria a la vez, sin importar el tamaño del fichero
await using var stream = File.OpenRead("video-4gb.mp4");
byte[] hash = await SHA256.HashDataAsync(stream);

// ❌ Carga el fichero completo antes de hashear
byte[] todo = File.ReadAllBytes("video-4gb.mp4");
// System.IO.IOException: The file is too long. This operation is currently
// limited to supporting files less than 2 gigabytes in size.
```

Y con ficheros de 500 MB la versión mala no lanza nada: reserva 500 MB por cada petición concurrente hasta que el proceso muere por `OutOfMemoryException` un viernes por la tarde. La versión de stream no tiene ese techo.

## `IncrementalHash`: cuando los datos vienen a trozos

Si hay que hashear varios campos —número de pedido, importe, moneda— la tentación es concatenar buffers. `IncrementalHash` alimenta el mismo digest en varias llamadas:

```csharp
using var hasher = IncrementalHash.CreateHash(HashAlgorithmName.SHA256);

hasher.AppendData(Encoding.UTF8.GetBytes("4711"));
hasher.AppendData(Encoding.UTF8.GetBytes("|89.90"));
hasher.AppendData(Encoding.UTF8.GetBytes("|EUR"));

Console.WriteLine(Convert.ToHexString(hasher.GetHashAndReset()));
// 66774E9451042C4AF78971EAF9B7332D1BF08513BEA59A705029DE8630A23A19
```

Ese es exactamente el digest de la cadena `"4711|89.90|EUR"` completa. `GetHashAndReset` devuelve el resultado **y deja la instancia lista para el siguiente**, que es cómo se reutiliza en un bucle. Y fíjate en los separadores `|`: sin ellos, `("47", "1189.90")` daría el mismo hash que `("4711", "89.90")`. `IncrementalHash` no te protege de eso.

## Thread-safety: la trampa que nadie ve

Aquí está el bug más desagradable de la ficha, porque es intermitente y no se reproduce en local. Las **instancias** de `SHA256` (`SHA256.Create()`) y de `IncrementalHash` **no son thread-safe**, porque mantienen estado interno entre llamadas; los **métodos estáticos** `HashData` **sí** se pueden llamar desde cualquier hilo, porque no comparten nada.

El caso real es este: alguien registra un servicio como *singleton* y guarda el hasher en un campo, porque "crear el objeto en cada llamada es un desperdicio".

```csharp
// ❌ Singleton + instancia compartida: bomba de relojería
public class HuellaDeContenido            // registrado como singleton
{
    private readonly SHA256 _sha = SHA256.Create();

    public string Calcular(byte[] datos) =>
        Convert.ToHexString(_sha.ComputeHash(datos));
}
```

En desarrollo funciona: una petición a la vez, un hilo a la vez. En producción, con veinte peticiones concurrentes, dos hilos entran a `ComputeHash` sobre el mismo estado interno y el resultado es **un digest que no corresponde a ningún dato**. Sin excepción, sin log, sin patrón. Cuando sí salta, el mensaje tampoco ayuda:

```
System.Security.Cryptography.CryptographicException: Hash not valid for use in specified state.
```

Piensa en lo que eso significa según dónde vaya el hash: una huella que invalida una caché sin motivo, un `TokenHash` que no valida y expulsa a un cliente al azar, un `instalador.exe` marcado como corrupto sin estarlo. Y siempre en un hilo distinto, así que no hay forma de reproducirlo a voluntad.

```csharp
// ✅ Método estático: sin estado, seguro desde cualquier hilo
public class HuellaDeContenido
{
    public string Calcular(byte[] datos) =>
        Convert.ToHexString(SHA256.HashData(datos));
}
```

Además de correcto es más rápido: no hay asignación ni `Dispose`. Si necesitas `IncrementalHash`, créalo **dentro** del método con `using` y no lo guardes en ningún campo.

## Contraseñas: `PasswordHasher<T>`

Vive en el paquete `Microsoft.Extensions.Identity.Core` y no exige montar ASP.NET Core Identity entero: puedes instanciarlo suelto. Por dentro es **PBKDF2 con HMAC-SHA512, sal aleatoria y los parámetros serializados dentro del propio hash**.

```csharp
using Microsoft.AspNetCore.Identity;

var hasher = new PasswordHasher<Cliente>();
var cliente = new Cliente { Email = "ana@ejemplo.com" };

cliente.PasswordHash = hasher.HashPassword(cliente, "contraseña-del-cliente");
db.Clientes.Add(cliente);
await db.SaveChangesAsync();

// PasswordHash queda así, ~84 caracteres en Base64:
// AQAAAAIAAYagAAAAEP2r9k...
```

El primer byte es la versión del formato, y detrás vienen el PRF, el número de iteraciones y la longitud de la sal. **El hash es autodescriptivo**: no necesitas otra columna para la sal ni para el coste. Guardar la sal aparte es un error que esta API te ahorra (ver [Sal vs seed](Sal-Vs-Seed.md)).

### `SuccessRehashNeeded`: la migración que se hace sola

La verificación devuelve tres valores: `Failed`, `Success` y `SuccessRehashNeeded`. El tercero es la pieza más elegante de esta API. Como el hash lleva sus parámetros dentro, la librería compara el coste con el que está configurado **ahora** y detecta los anticuados; y te lo dice en el único instante en que tienes la contraseña en claro para poder rehacerlo: el login.

```csharp
var resultado = hasher.VerifyHashedPassword(
    cliente, cliente.PasswordHash, passwordIntroducida);

switch (resultado)
{
    case PasswordVerificationResult.Failed:
        return Unauthorized();

    case PasswordVerificationResult.SuccessRehashNeeded:
        // Correcta, pero hasheada con parámetros antiguos
        cliente.PasswordHash = hasher.HashPassword(cliente, passwordIntroducida);
        await db.SaveChangesAsync();          // ← el paso que se olvida
        goto case PasswordVerificationResult.Success;

    case PasswordVerificationResult.Success:
        return await EmitirSesion(cliente);
}
```

El `SaveChangesAsync` es imprescindible y se cae de casi todos los ejemplos que hay por ahí: sin persistir, el hash se recalcula en cada login para siempre y la migración nunca termina.

### Fija el coste explícitamente

El `IterationCount` por defecto **depende de la versión del runtime**. Microsoft lo sube con los años, que es lo correcto, pero significa que tu coste cambia sin que tú lo decidas:

```csharp
// ✅ El coste es una decisión tuya, versionada con el código
builder.Services.Configure<PasswordHasherOptions>(o =>
{
    o.IterationCount = 600_000;
    o.CompatibilityMode = PasswordHasherCompatibilityMode.IdentityV3;
});
```

Ahora las dos piezas encajan: fijas 600 000 iteraciones, los hashes viejos (con 100 000, con 210 000) devuelven `SuccessRehashNeeded`, y la migración se despliega sola, login a login. **Nadie cambia su contraseña y nadie ejecuta un script** — porque un script no podría, ya que las contraseñas en claro no las tienes. Los parámetros que tiene sentido poner ahí los razona [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md).

## bcrypt con `BCrypt.Net-Next`

Paquete NuGet `BCrypt.Net-Next`, licencia **MIT**, el port de bcrypt mantenido para .NET:

```csharp
string hash = BCrypt.Net.BCrypt.HashPassword("contraseña-del-cliente", workFactor: 12);
bool ok = BCrypt.Net.BCrypt.Verify("contraseña-del-cliente", hash);

// $2a$12$N9qo8uLOickgx2ZMRZoMye.IjZAgcfl7p92ldGxad68LJZdL17lhW
//      ^^ el workFactor viaja dentro del hash, igual que la sal
```

**El `workFactor` es exponencial: cada +1 duplica el coste.** Traducido a la única unidad que importa, lo que tarda un login:

| workFactor | Tiempo aproximado por hash |
|---|---|
| 10 | ~65 ms |
| 12 | ~260 ms |
| 14 | ~1 040 ms |

Pasar de 10 a 14 multiplica por 16 el trabajo de quien intente romper la base de datos y añade un segundo al login de cada cliente. Mide en **tu** hardware antes de fijarlo: estos números varían con la CPU.

**Y la limitación que hay que conocer: bcrypt solo considera los primeros 72 bytes.** Todo lo que pase de ahí se ignora en silencio:

```csharp
var corta = new string('a', 72);
BCrypt.Net.BCrypt.Verify(corta + "SUFIJO-QUE-SE-IGNORA",
                         BCrypt.Net.BCrypt.HashPassword(corta));
// true  ← las dos contraseñas son equivalentes para bcrypt
```

Ojo con la "solución" de pre-hashear con SHA-256: si pasas el digest en binario y contiene un byte cero, bcrypt trunca ahí. Si lo haces, pásalo en Base64 o hexadecimal, nunca los bytes crudos.

## Argon2id en .NET: qué paquete y con qué licencia

.NET no trae Argon2. Los dos paquetes habituales son muy distintos, y la diferencia relevante no es técnica sino **legal**:

| Opción | Licencia | API | Sal y parámetros | ¿Vale en entorno FIPS? |
|---|---|---|---|---|
| `Konscious.Security.Cryptography.Argon2` | **MIT** — permisiva estándar, sin fricción en software comercial | Bajo nivel: `Argon2id` con propiedades explícitas | Las gestionas y las almacenas tú | No |
| `Isopoh.Cryptography.Argon2` | **CC BY 4.0** — Creative Commons con atribución, poco habitual en código y que **puede requerir revisión legal** | Alto nivel: `Hash` y `Verify` | Automáticas, formato `$argon2id$...` | No |
| `PasswordHasher<Cliente>` (PBKDF2) | Microsoft, MIT | Alto nivel | Automáticas, dentro del hash | **Sí, sin discusión** |

Si tu contexto exige FIPS, la última fila cierra el debate: Argon2 no está aprobado por el NIST y PBKDF2 sí. No es cuestión de qué algoritmo es mejor —Argon2id lo es— sino de qué puedes certificar.

Con Isopoh el código es de dos líneas y el hash lleva sus propios parámetros:

```csharp
string hash = Isopoh.Cryptography.Argon2.Argon2.Hash("contraseña-del-cliente");
bool ok = Isopoh.Cryptography.Argon2.Argon2.Verify(hash, "contraseña-del-cliente");

// $argon2id$v=19$m=65536,t=3,p=1$c29tZXNhbHQ...$RdescudvJCsgt3ub+b+dWRWJTmaa
//                ^^^^^^^^^^^^^^^ memoria, iteraciones y paralelismo, dentro del hash
```

Con Konscious lo construyes tú: sal, memoria, iteraciones, paralelismo y longitud de salida, y **decides dónde guardas todo eso**, porque el paquete devuelve solo bytes. Más trabajo y más superficie para equivocarse; a cambio, la licencia no requiere ninguna conversación con nadie. Antes de elegir cualquiera de los dos, mira la fecha del último *release* y si hay alguien respondiendo *issues*: **los paquetes de terceros no se auditan solos**.

## Aleatoriedad: `RandomNumberGenerator`, nunca `Random`

Para cualquier valor que deba ser impredecible —tokens de sesión, códigos de recuperación, sales manuales— usa el generador criptográfico:

```csharp
byte[] token = RandomNumberGenerator.GetBytes(32);        // 256 bits de entropía
string paraElCliente = Convert.ToBase64String(token);     // se entrega una sola vez
byte[] paraLaBd = SHA256.HashData(token);                 // esto es lo que se guarda
```

En la columna `TokenHash` va `paraLaBd`; `paraElCliente` viaja en la cookie y **no se almacena en ningún sitio**, así que una filtración de la base de datos no entrega ni una sesión. Por qué aquí basta un hash rápido y en las contraseñas no, en [Contraseñas vs tokens de sesión](Contrasenas-Vs-Tokens-De-Sesion.md). El contraste importa porque el atajo está a mano y compila igual:

```csharp
// ❌ Predecible: Random es un PRNG estadístico, no criptográfico
var rnd = new Random();
byte[] token = new byte[32];
rnd.NextBytes(token);
```

`Random` genera una secuencia determinista a partir de una semilla pequeña. Quien vea **un** token puede reconstruir el estado interno y calcular los siguientes; y si el código usa `new Random(semilla)` con algo adivinable —un `DateTime.Now`, un id de cliente— el espacio a probar se queda en unos miles de valores. No es un ataque teórico, es un bucle, y no hay ningún síntoma que lo delate porque los tokens *parecen* aleatorios. La regla es corta: **si el valor protege algo, `RandomNumberGenerator`; si es para elegir un color de gráfico, `Random`.**

## `GetHashCode()` no es un hash

Esto sorprende a casi todo el mundo. Desde .NET Core, `string.GetHashCode()` está **aleatorizado por proceso**: cada arranque usa una semilla distinta.

```csharp
Console.WriteLine("ana@ejemplo.com".GetHashCode());

// Primera ejecución:  -1483298031
// Segunda ejecución:   874253119     ← mismo binario, misma máquina, misma cadena
```

No es un bug: es una protección **deliberada** contra ataques de colisión sobre diccionarios. Si el atacante no puede predecir en qué *bucket* cae cada clave, no puede degradar un `Dictionary` a lista enlazada enviando claves elegidas. El precio es que el valor solo tiene sentido **dentro de la vida del proceso actual**. Las tres cosas que nunca hay que hacer con él:

1. **Persistirlo.** Un identificador o una clave de caché derivados de `GetHashCode()` dejan de coincidir tras el primer reinicio, y la pista es demoledora: "funcionaba hasta que desplegamos".
2. **Particionar datos.** `GetHashCode() % numeroDeShards` manda el mismo cliente a particiones distintas según qué instancia atienda la petición.
3. **Compararlo entre procesos.** Dos réplicas tras un balanceador dan valores diferentes para la misma cadena. Con una instancia el bug no aparece; al escalar a dos, sí.

Para una huella estable, `SHA256.HashData`. Y cuando no hay adversario y prima la velocidad —deduplicar filas de un CSV, detectar si un blob ha cambiado—, `XxHash64.HashToUInt64(bytes)` del paquete `System.IO.Hashing` es mucho más rápido y **estable entre procesos y versiones**. No es criptográfico: cualquiera puede fabricar colisiones a propósito, así que sirve para detectar cambios accidentales y nunca para validar nada frente a quien quiera engañarte.

## Comparar secretos en tiempo constante

`SequenceEqual` y `==` **cortocircuitan en el primer byte distinto**. Eso los hace rápidos y, para un secreto, los hace filtrar cuántos bytes iniciales ha acertado quien lo intenta.

```csharp
// ❌ El tiempo de comparación depende de cuántos bytes coinciden
if (hashRecibido.SequenceEqual(hashAlmacenado)) { ... }

// ✅ Mismo tiempo siempre, independientemente de dónde esté la diferencia
if (CryptographicOperations.FixedTimeEquals(hashRecibido, hashAlmacenado)) { ... }
```

El caso concreto: validar el token de una cookie contra la columna `TokenHash`. La diferencia por byte es de nanosegundos y a través de la red suena a ruido, pero con suficientes intentos promediados se distingue, y automatizar suficientes intentos es gratis. Cuesta lo mismo escribirlo bien.

Su hermana `CryptographicOperations.ZeroMemory` sobrescribe un `byte[]` con ceros cuando terminas con él, y es lo que hay que usar con sales y claves derivadas: `finally { CryptographicOperations.ZeroMemory(clave); }`. El recolector no lo hace por ti, y una clave olvidada en el *heap* sale en cualquier volcado del proceso.

## La contraseña en un `string` vive más de lo que crees

Los `string` de .NET son **inmutables**: no puedes sobrescribirlos. Cuando la contraseña llega como `string`, permanece en el *heap* —y en cualquier volcado de memoria del proceso— hasta que el recolector decida limpiar ese hueco, que puede ser mucho después de que tu método haya terminado. Poner `password = null` no borra nada: solo suelta la referencia.

En ASP.NET Core **no puedes evitarlo del todo**, porque el *model binding* ya te entrega la contraseña como `string` antes de que tu código exista. Lo honesto es reconocerlo y limitar el daño: no la registres nunca —un `logger.LogDebug("Login {@Peticion}", peticion)` la vuelca entera—, no la interpoles en mensajes de excepción o de validación, no la caches "para revalidar luego", y pásala directa al hasher sin copiarla a campos ni propiedades de vida más larga. `SecureString` existe y **no es la respuesta**: Microsoft recomienda no usarla en código nuevo, porque no cifra en todas las plataformas y el `string` intermedio suele aparecer igual.

## Qué API usar para qué

| Necesidad | API | Paquete | Aviso |
|---|---|---|---|
| Integridad de `instalador.exe` | `SHA256.HashData` | En la caja | Compara con `FixedTimeEquals`, no con `==` |
| Huella de `video-4gb.mp4` | `SHA256.HashDataAsync(stream)` | En la caja | Nunca `File.ReadAllBytes` primero |
| Huella de varios campos | `IncrementalHash` | En la caja | Instancia no thread-safe; usa separadores |
| Hasheo rápido sin adversario | `XxHash64` | `System.IO.Hashing` | No criptográfico: colisiones fabricables |
| Contraseña de un `Cliente` | `PasswordHasher<Cliente>` | `Microsoft.Extensions.Identity.Core` | Fija `IterationCount` y persiste el re-hash |
| Contraseña con bcrypt | `BCrypt.Net.BCrypt` | `BCrypt.Net-Next` (MIT) | Solo los primeros 72 bytes cuentan |
| Contraseña con Argon2id | `Argon2` | Konscious (MIT) / Isopoh (CC BY 4.0) | Revisa la licencia antes del `add package` |
| Token de sesión o código de recuperación | `RandomNumberGenerator.GetBytes` | En la caja | Nunca `Random`; entrega el token, guarda su hash |
| Comparar dos secretos | `CryptographicOperations.FixedTimeEquals` | En la caja | `SequenceEqual` filtra por timing |
| Clave de un `Dictionary` en memoria | `GetHashCode()` | En la caja | Válido solo dentro del proceso; jamás se persiste |

Y el que no está en la tabla: **`SHA256` para contraseñas**. Está en la caja, es cómodo y es la respuesta equivocada; el razonamiento completo, en [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md).

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| Digests distintos de forma intermitente, sin patrón. A veces `System.Security.Cryptography.CryptographicException: Hash not valid for use in specified state.` | Una instancia de `SHA256` o `IncrementalHash` guardada en un campo de un servicio *singleton* y usada desde varios hilos |
| `System.IO.IOException: The file is too long...`, o el proceso muere con `OutOfMemoryException` bajo carga | `File.ReadAllBytes` en lugar de `HashDataAsync(stream)`: cada petición reserva el fichero completo |
| Los identificadores derivados de un hash dejan de coincidir tras un reinicio o al añadir una réplica | Se persistió `string.GetHashCode()`, aleatorizado por proceso |
| Una filtración de la base de datos deja las contraseñas descifradas en horas | Se usó `SHA256` para `PasswordHash`: es un hash rápido, diseñado para ir a miles de millones por segundo |
| Alguien predice los tokens de la tabla `Sesiones` | `new Random()` en lugar de `RandomNumberGenerator` |
| Un secreto se adivina byte a byte midiendo tiempos de respuesta | Comparación con `==` o `SequenceEqual` en vez de `FixedTimeEquals` |
| Tras actualizar el runtime ningún login valida. Aviso previo en compilación: `SYSLIB0041` | PBKDF2 hecho a mano dependiendo del `IterationCount` por defecto, que cambió. El coste debe guardarse junto al hash |
| Dos servicios calculan hashes distintos del mismo texto | Codificaciones distintas: `Encoding.UTF8` en uno y `Encoding.Unicode` o `Encoding.Default` en el otro |
| `SuccessRehashNeeded` en cada login del mismo cliente, para siempre | Se re-hashea pero no se persiste el nuevo valor |

## Buenas prácticas avanzadas

- **Fija el coste de `PasswordHasher` por opciones y deja que el re-hash lo despliegue.** El `IterationCount` por defecto depende de la versión del runtime, así que decláralo explícito para que el coste sea una decisión versionada con tu código: `services.Configure<PasswordHasherOptions>(o => o.IterationCount = 600_000);`. Los hashes con el coste antiguo irán devolviendo `SuccessRehashNeeded` y la migración se hace sola, login a login, sin scripts ni pedirle a nadie que cambie su contraseña. Es la única estrategia de migración que existe, precisamente porque las contraseñas en claro no las tienes.
- **Prefiere los métodos estáticos `HashData` y no guardes nunca una instancia de hasher en un campo.** Las instancias de `SHA256` y `IncrementalHash` no son thread-safe y el fallo es silencioso: digests corruptos e intermitentes en producción, imposibles de reproducir en local. Los estáticos no tienen estado, se llaman desde cualquier hilo y encima evitan la asignación y el `Dispose`. Si necesitas `IncrementalHash`, créalo dentro del método con `using`.
- **`GetHashCode()` no es un hash: está aleatorizado por proceso.** Desde .NET Core devuelve un valor distinto en cada ejecución, y es una protección anti-DoS deliberada, no un descuido. Nunca lo persistas, nunca particiones datos con él, nunca lo compares entre procesos: el bug aparece el día del despliegue o el día que escalas a dos réplicas. Para huellas estables, `SHA256.HashData`; y `XxHash64` de `System.IO.Hashing` cuando no haya adversario y prime la velocidad.
- **Compara secretos con `CryptographicOperations.FixedTimeEquals`.** `SequenceEqual` y `==` cortocircuitan en el primer byte distinto y filtran por timing cuántos bytes ha acertado quien lo intenta. Al validar el hash de un token recibido contra el almacenado, la comparación en tiempo constante cuesta lo mismo de escribir. Su hermana `CryptographicOperations.ZeroMemory` borra sales y claves de un `byte[]` cuando terminas con ellas, algo que el recolector no hará por ti.
- **La contraseña en un `string` vive más de lo que crees.** Los `string` son inmutables: no se pueden sobrescribir y permanecen en memoria —y en cualquier volcado del proceso— hasta que el recolector decida. En ASP.NET no lo puedes evitar del todo, porque el *model binding* ya te la entrega como `string`, pero sí puedes no multiplicar el problema: no la logues, no la interpoles en mensajes, no la caches y pásala directa al hasher. `SecureString` no es la solución, y Microsoft recomienda no usarla en código nuevo.
- **Recuerda qué no hace `PasswordHasher`: no limita intentos ni gestiona sesiones.** Solo hashea y verifica. El *rate limiting* del login, el bloqueo temporal de la cuenta y la expiración de la sesión son código tuyo. Un hasher con 600 000 iteraciones no protege de nada si se le puede llamar diez mil veces por minuto contra la misma cuenta.

## Documentación oficial

- [`System.Security.Cryptography`](https://learn.microsoft.com/dotnet/api/system.security.cryptography) — la referencia de las APIs en la caja. Ve directamente a las páginas de `SHA256`, `IncrementalHash` y `CryptographicOperations`: cada una dice explícitamente qué es thread-safe y qué no, que es la información que evita el bug del *singleton*.
- [Hashing de contraseñas en ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/data-protection/consumer-apis/password-hashing) — describe byte a byte el formato del hash que produce `PasswordHasher`. Es donde acudir cuando necesites saber qué hay dentro de esa cadena Base64, o migrar hashes desde otro sistema.
- [Repositorio de `BCrypt.Net-Next`](https://github.com/BcryptNet/bcrypt.net) — el README documenta el `workFactor`, el comportamiento con los 72 bytes y la API de *enhanced hashing*. Míralo también para comprobar la actividad reciente antes de fijar la dependencia.
- [Repositorio de `Konscious.Security.Cryptography`](https://github.com/kmaragon/Konscious.Security.Cryptography) — la implementación de Argon2 con licencia MIT, la recomendable de las dos. El README trae el ejemplo mínimo de `Argon2id` con sal y parámetros explícitos, que es exactamente lo que te toca escribir con esta opción.

## Recursos didácticos

- [BenchmarkDotNet](https://benchmarkdotnet.org/articles/guides/getting-started.html) — la forma correcta de medir cuánto tarda un hash en **tu** hardware. Veinte líneas comparando `SHA256.HashData` con `PasswordHasher.HashPassword` a distintos `IterationCount` hacen tangible la diferencia entre un hash rápido y uno lento, y te dan el número real para fijar el coste. Medir con `Stopwatch` en un bucle engaña, por el JIT y por el calentamiento.
- [Argon2 Calculator](https://argon2.online/) — ejecuta Argon2id en el navegador con los parámetros que le pongas y muestra el hash resultante con su formato `$argon2id$...`. Útil para entender qué significa cada campo antes de configurarlo en código.
- [Password Storage Cheat Sheet de OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — la comparativa de PBKDF2, bcrypt y Argon2 con parámetros concretos y actualizados. Es el documento al que apuntar cuando alguien pregunte de dónde sale el número de iteraciones que has puesto.
- [dotnetfiddle.net](https://dotnetfiddle.net/) — para comprobar en treinta segundos los efectos que sorprenden: ejecuta dos veces `Console.WriteLine("ana@ejemplo.com".GetHashCode())` y verás dos valores distintos con tus propios ojos.

---

*En resumen: en .NET, `SHA256.HashData` para huellas e integridad, `PasswordHasher<Cliente>` (PBKDF2 first-party, con re-hash automático) para contraseñas sin salir de la caja, y BCrypt.Net-Next o un paquete Argon2 — mirando su licencia — si quieres ir más allá.*
