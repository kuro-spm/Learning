# Hashing en C#/.NET

## ¿Qué es?

Una guía práctica de las herramientas que ofrece .NET para hashear: `System.Security.Cryptography` para hashes rápidos (SHA-2), `PasswordHasher` de Microsoft para contraseñas (PBKDF2), y los paquetes de terceros para bcrypt y Argon2.

## ¿Por qué existe?

La teoría de las otras fichas hay que aterrizarla en código, y .NET tiene una peculiaridad: **trae de serie los hashes rápidos y una solución de contraseñas first-party (PBKDF2)**, pero no incluye bcrypt ni Argon2 — para esos hay que evaluar paquetes NuGet de terceros, con sus mantenimientos y licencias. Saber qué hay en la caja evita reinventar (mal) la rueda criptográfica.

> Regla de oro en criptografía: no implementes primitivas tú misma. Usa lo que trae la plataforma o librerías auditadas.

## ¿Cuándo y para qué se usa?

- **`SHA256` / `SHA512`**: integridad de ficheros, ETags, deduplicación, huellas de contenido, hashear tokens aleatorios.
- **`PasswordHasher`**: almacenar contraseñas de usuarias con lo que trae ASP.NET Core, sin dependencias externas.
- **BCrypt.Net-Next / paquetes Argon2**: cuando quieres bcrypt o Argon2id (la recomendación OWASP) en lugar de PBKDF2.
- **`RandomNumberGenerator`**: generar sales y tokens — el aleatorio criptográfico, no `Random`.

## Lo mínimo que necesitas saber

**1. SHA-256 con `System.Security.Cryptography`**

Desde .NET 5 existe la forma estática, sin crear ni liberar objetos:

```csharp
using System.Security.Cryptography;
using System.Text;

byte[] contenido = Encoding.UTF8.GetBytes("contenido del documento");
byte[] hash = SHA256.HashData(contenido);
string hex = Convert.ToHexString(hash);
// "A591A6D40BF420404A011733CFB7B190D62C65BF0BCDA32B57B277D9AD9F146E"
```

Para un fichero grande, sin cargarlo entero en memoria:

```csharp
using var stream = File.OpenRead("video.mp4");
byte[] hash = await SHA256.HashDataAsync(stream);
```

**2. Contraseñas con `PasswordHasher` (PBKDF2, first-party)**

Vive en el paquete `Microsoft.Extensions.Identity.Core` y usa PBKDF2 con sal aleatoria y parámetros serializados dentro del propio hash (en versiones recientes: HMAC-SHA512 con cientos de miles de iteraciones).

```csharp
using Microsoft.AspNetCore.Identity;

var hasher = new PasswordHasher<User>();

// Registro: guardar el hash, nunca la contraseña
string hashAlmacenado = hasher.HashPassword(user, "contraseña-de-la-usuaria");

// Login: verificar
PasswordVerificationResult resultado =
    hasher.VerifyHashedPassword(user, hashAlmacenado, passwordIntroducida);

if (resultado == PasswordVerificationResult.SuccessRehashNeeded)
{
    // Correcta, pero hasheada con parámetros antiguos:
    // re-hashear ahora (es el único momento en que tienes la contraseña en claro)
    string nuevoHash = hasher.HashPassword(user, passwordIntroducida);
    // ... persistir nuevoHash
}
```

Fíjate en `SuccessRehashNeeded`: como el hash lleva sus parámetros dentro, la librería detecta hashes con coste anticuado y te da el momento exacto para actualizarlos. Así el coste puede subir con los años sin pedir a nadie que cambie su contraseña.

**3. bcrypt con BCrypt.Net-Next**

Paquete NuGet `BCrypt.Net-Next` (licencia MIT), el port de bcrypt mantenido para .NET:

```csharp
string hash = BCrypt.Net.BCrypt.HashPassword("contraseña", workFactor: 12);
bool ok = BCrypt.Net.BCrypt.Verify("contraseña", hash);
```

El `workFactor` es exponencial: cada +1 duplica el coste. Recuerda la limitación de bcrypt: solo considera los primeros 72 bytes de la contraseña.

**4. Argon2id: paquetes y licencias**

.NET no trae Argon2; los dos paquetes habituales son:

- **`Konscious.Security.Cryptography.Argon2`** — licencia **MIT** (permisiva estándar, sin fricción en software comercial). API de bajo nivel: tú gestionas sal y parámetros.
- **`Isopoh.Cryptography.Argon2`** — licencia **CC BY 4.0** (Creative Commons con atribución; poco habitual en código y que puede requerir revisión legal en algunas empresas). A cambio, API más cómoda con formato autodescriptivo y verificación incluidas.

```csharp
// Isopoh: alto nivel, formato $argon2id$... autodescriptivo
string hash = Isopoh.Cryptography.Argon2.Argon2.Hash("contraseña");
bool ok = Isopoh.Cryptography.Argon2.Argon2.Verify(hash, "contraseña");
```

Antes de elegir, valora licencia, mantenimiento del paquete y si tu contexto exige FIPS (en cuyo caso PBKDF2 vía `PasswordHasher` es la opción sin discusión).

**5. Sales y tokens: `RandomNumberGenerator`**

Para cualquier valor que deba ser impredecible (sales manuales, tokens de sesión, códigos de recuperación), usa el generador criptográfico:

```csharp
byte[] token = RandomNumberGenerator.GetBytes(32); // 256 bits de entropía
string tokenParaLaUsuaria = Convert.ToBase64String(token);
byte[] hashParaLaBd = SHA256.HashData(token);      // en BD se guarda el hash, no el token
```

Nunca `new Random()` para secretos: es predecible.

## Lo que NO hace

- **`PasswordHasher` no limita intentos ni gestiona sesiones** — solo hashea y verifica; el rate-limiting del login es cosa tuya.
- **`SHA256` no vale para contraseñas** — está en la caja y es tentador, pero es un hash rápido (ver [funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md)).
- **Los paquetes de terceros no se auditan solos** — fija versiones, revisa mantenimiento y licencia antes de añadirlos a producción.

---

*En resumen: en .NET, `SHA256.HashData` para huellas e integridad, `PasswordHasher` (PBKDF2 first-party, con re-hash automático) para contraseñas sin salir de la caja, y BCrypt.Net-Next o un paquete Argon2 — mirando su licencia — si quieres ir más allá.*
