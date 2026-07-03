# Contraseñas vs tokens de sesión: elegir el hash adecuado

**¿Qué es?** Un criterio de decisión: ante un secreto que hay que guardar hasheado, ¿toca una función lenta (Argon2id, PBKDF2...) o basta un hash rápido (SHA-256)? La respuesta no depende del algoritmo "de moda", sino de **cuánta entropía tiene el secreto** — es decir, de si un atacante puede adivinarlo probando.

---

## ¿Por qué existe?

Es un error frecuente en ambos sentidos: hashear contraseñas con SHA-256 (inseguro: se prueban miles de millones por segundo) o hashear tokens aleatorios con Argon2 (inútil: desperdicia 20 MiB de RAM y decenas de milisegundos por cada request autenticada, sin ganar seguridad).

La clave está en el origen del secreto:

- Una **contraseña** la elige una persona: vive en un espacio pequeño y sesgado ("veranito2024", el nombre del perro...). Los diccionarios funcionan, así que cada intento debe costar caro.
- Un **token aleatorio de 256 bits** lo genera la máquina: 2^256 posibilidades uniformes. No existe diccionario posible; probar candidatos es inviable aunque el atacante calcule a la velocidad de un hash rápido.

> Piensa en un PIN de 4 cifras frente a una llave física de perfil único: el PIN necesita una puerta que tarde en abrirse (10.000 combinaciones se prueban rápido); la llave no, porque nadie puede fabricar todas las llaves posibles.

---

## ¿Cuándo y para qué se usa?

Cada vez que diseñas autenticación en una aplicación web típica (una tienda online, un SaaS):

- **Contraseñas de usuarias** → función de derivación lenta con sal y factor de coste.
- **Tokens de sesión opacos, API keys, códigos de recuperación largos** generados con un CSPRNG → basta un hash rápido (SHA-256) para no guardarlos en claro: si roban la base de datos, el ladrón tiene huellas, no llaves.
- **Códigos cortos elegibles o de pocos dígitos** (un OTP de 6 cifras) → vuelven a ser adivinables: rate-limiting estricto y, si se almacenan, tratamiento como secreto de baja entropía.

---

## Lo mínimo que necesitas saber

**1. La pregunta que lo decide todo**

¿Puede un atacante *enumerar* candidatos plausibles del secreto? Si sí (contraseñas, PINs) → hash lento. Si no (256 bits aleatorios) → hash rápido, que además permite verificar tokens en cada request sin coste apreciable.

**2. El flujo del token opaco**

```csharp
// Al iniciar sesión: generar y entregar el token; guardar solo su hash
byte[] token = RandomNumberGenerator.GetBytes(32);
string tokenParaLaCookie = Convert.ToBase64String(token);
byte[] hashEnBd = SHA256.HashData(token);

// En cada request: hashear lo recibido y buscar el hash en BD
byte[] candidato = SHA256.HashData(Convert.FromBase64String(tokenRecibido));
// SELECT ... FROM Sesion WHERE TokenHash = @candidato
```

**3. Un caso real de referencia**

En una aplicación de gestión real (el proyecto *Seamless Studio*, una app web con login propio y sesiones en base de datos) se decidió exactamente así:

- **Contraseñas → PBKDF2** vía `PasswordHasher` de Microsoft: es la opción *first-party* de .NET (sin dependencias de terceros que auditar) y su `SuccessRehashNeeded` re-hashea automáticamente los hashes con parámetros anticuados cuando la usuaria hace login. Encaja porque una contraseña es adivinable por diccionario: necesita lentitud configurable y sal.
- **Tokens de sesión → opacos, de 256 bits aleatorios, guardados como SHA-256** en la base de datos. Encaja porque no hay nada que "adivinar": ningún diccionario cubre 2^256 valores uniformes, así que la lentitud no aporta — y el hash rápido permite validar la sesión en cada request casi gratis, protegiendo igualmente los tokens si la tabla se filtra.

Dos algoritmos distintos en la misma feature, y ninguno es "mejor" que el otro: cada uno responde a la entropía del secreto que protege.

---

## Lo que NO hace

- **No convierte el hash en la única defensa** — sigue haciendo falta TLS, rate-limiting del login, expiración de sesiones y revocación de tokens.
- **No aplica a JWT ni tokens firmados** — esos no se guardan en BD ni se hashean: se verifican por su firma. Este criterio es para tokens *opacos*.
- **No exime de usar un CSPRNG** — todo el razonamiento del token descansa en que los 256 bits sean realmente aleatorios (`RandomNumberGenerator`, nunca `Random`).

---

## Buenas prácticas avanzadas

- **Dale a cada tipo de token un prefijo identificable** — un token `sk_live_...` o `ghp_...` no es más seguro criptográficamente, pero cambia la operativa: el *secret scanning* (GitHub, herramientas de CI) lo detecta si acaba en un commit, los logs pueden depurarse por patrón y, en un incidente, sabes al instante qué tipo de credencial se ha expuesto y qué revocar.
- **Verifica el hash del token en tiempo constante** — si recuperas la fila por otro criterio y luego comparas hashes en código, usa `CryptographicOperations.FixedTimeEquals`, nunca `==`: la comparación ordinaria termina en el primer byte distinto y filtra información por timing. Buscar directamente con `WHERE TokenHash = @hash` es aceptable justo porque SHA-256 impide al atacante controlar qué bytes se comparan.
- **Rota el token en cada cambio de privilegio** — emite un token nuevo (invalidando el anterior) al iniciar sesión, al elevar permisos o al cambiar la contraseña. Corta de raíz la *session fixation*: un token que el atacante logró plantar o capturar antes de la autenticación deja de valer justo cuando la sesión empieza a tener valor.

---

*En resumen: la entropía del secreto elige el algoritmo — contraseña humana ⇒ KDF lenta con sal; token aleatorio de 256 bits ⇒ SHA-256 basta, porque contra 2^256 posibilidades no hay diccionario que valga.*
