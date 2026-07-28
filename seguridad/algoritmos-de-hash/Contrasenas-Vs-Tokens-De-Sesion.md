# Contraseñas vs tokens de sesión: elegir el hash adecuado

## ¿Qué es?

Un criterio de decisión: ante un secreto que hay que guardar hasheado, ¿toca una función deliberadamente lenta (Argon2id, PBKDF2...) o basta un hash rápido (SHA-256)? La respuesta no depende del algoritmo de moda, sino de **cuánta entropía tiene el secreto** — es decir, de si un atacante puede adivinarlo probando candidatos.

## ¿Por qué existe?

Porque es un error frecuente **en los dos sentidos**, y ninguno de los dos da la cara: la aplicación funciona igual de bien equivocándose.

- **Hashear contraseñas con SHA-256** es inseguro. Quien se lleve la tabla puede probar del orden de diez mil millones de candidatos por segundo en una GPU doméstica, y las contraseñas humanas viven en un espacio que cabe en un diccionario.
- **Hashear tokens aleatorios con Argon2** es inútil. Gasta decenas de milisegundos y unos 20 MiB de RAM en **cada petición autenticada**, y no compra ni un bit de seguridad, porque no hay nada que adivinar.

La diferencia está en el origen del secreto, no en su aspecto:

- Una **contraseña** la elige una persona: vive en un espacio pequeño y sesgado (`veranito2024`, el nombre del perro, la matrícula del coche). Los diccionarios funcionan, así que cada intento debe costar caro.
- Un **token aleatorio de 256 bits** lo genera la máquina: 2²⁵⁶ posibilidades uniformes. No existe diccionario posible, y probar candidatos es inviable aunque el atacante calcule a la velocidad de un hash rápido.

> Piensa en un PIN de cuatro cifras frente a una llave física de perfil único. El PIN necesita una puerta que tarde en abrirse, porque 10 000 combinaciones se prueban en una tarde; la llave no lo necesita, porque nadie puede fabricar todas las llaves posibles.

## ¿Cuándo y para qué se usa?

Cada vez que diseñas la autenticación de una aplicación web y tienes que decidir cómo se guarda cada secreto. El ejemplo que recorre la ficha es una **tienda online con API en .NET** cuya base de datos tiene dos tablas relevantes: `Clientes`, con la contraseña en la columna `PasswordHash`, y `Sesiones`, con el identificador de sesión en `TokenHash`. Las dos columnas guardan un hash, y **cada una pide un algoritmo distinto**.

Esta ficha cierra la colección: es el criterio que ordena todo lo anterior. Da por conocidas las propiedades del hash criptográfico y la comparación en tiempo constante, que están en [Hash criptográfico](Hash-Criptografico.md); las familias de hash rápido, en [Hashes de propósito general](Hashes-De-Proposito-General.md); el detalle de PBKDF2, bcrypt, scrypt y Argon2id con sus parámetros, en [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md) — aquí decimos «una KDF» y remitimos allí. La sal y sus confusiones habituales están en [Sal vs seed](Sal-Vs-Seed.md), y los paquetes concretos de .NET en [Hashing en C#/.NET](Hashing-En-CSharp.md).

## La entropía, definida y con números

**Entropía** es la medida de cuánto no se puede predecir un secreto, y se expresa en **bits**: cada bit duplica el número de candidatos que hay que probar. Un secreto de *n* bits equivale a 2ⁿ posibilidades igualmente probables. Al revés, si hay `C` candidatos, la entropía son log₂(C) bits.

La palabra importante es **efectivos**. La entropía no la da la longitud del secreto, sino lo impredecible que es para quien lo ataca en el orden en que lo ataca:

```text
"contrasena123"   → 13 caracteres, pero está en cualquier diccionario: ~10 bits
"Verano2024!"     → 11 caracteres con mayúscula, número y símbolo: ~22 bits efectivos
"q7#Lm2$xR9!vB4"  → 14 caracteres de un gestor de contraseñas: ~80 bits
```

Los tres son «contraseñas fuertes» según el validador típico del formulario de registro. Solo uno lo es de verdad. Con eso en la mano, la tabla que decide todo lo demás:

| Secreto | Entropía aprox. | Candidatos | ¿Lo puede enumerar un atacante? |
|---|---|---|---|
| PIN de 4 dígitos | ~13 bits | 10 000 | **Sí**, a mano si le dejas |
| OTP de 6 dígitos | ~20 bits | 1 000 000 | **Sí**, en segundos por red |
| Contraseña humana típica | 20-30 bits **efectivos** | de 1 millón a 1 000 millones | **Sí**, es el ataque de diccionario de siempre |
| Contraseña de gestor (14 caracteres) | ~80 bits | 10²⁴ | No, con una KDF calibrada |
| Token aleatorio de 128 bits | 128 bits | 3,4 · 10³⁸ | No |
| Token aleatorio de 256 bits | 256 bits | 1,2 · 10⁷⁷ | No, en ningún universo |

Y las dos filas extremas puestas en tiempo, con un hash rápido a 10¹⁰ intentos por segundo:

```text
PIN de 4 dígitos    10 000 candidatos    → menos de un microsegundo
Token de 128 bits   3,4 · 10^38          → ~10^21 años
```

De aquí sale el criterio entero, y se puede enunciar en una frase: **si el espacio de candidatos es enumerable, encarece cada intento; si no lo es, la lentitud no compra nada.** Contra 10 000 candidatos, multiplicar el coste por un millón es la única defensa que existe. Contra 2¹²⁸, el atacante ya ha perdido a la velocidad máxima del hardware, y ralentizarlo diez millones de veces más no cambia un resultado que ya era imposible.

## La pregunta que lo decide todo

> **¿Puede un atacante *enumerar* candidatos plausibles de este secreto?**

Si la respuesta es **sí** (contraseñas, PIN, códigos cortos, cualquier cosa que elija una persona) → hash lento, una KDF con sal y factor de coste.

Si la respuesta es **no** (128 bits o más salidos de un generador criptográfico) → hash rápido, SHA-256, que además permite verificar el secreto en cada petición sin coste apreciable.

No hay una tercera respuesta, y fíjate en que la pregunta no menciona en ningún momento para qué sirve el secreto. Un código de recuperación de cuenta y un token de sesión hacen cosas muy distintas y se hashean igual, porque los genera la misma clase de generador.

## La tabla de decisión

| Tipo de secreto | Quién lo genera | Entropía | Cómo se guarda | Por qué |
|---|---|---|---|---|
| Contraseña de una persona | La persona | 20-30 bits efectivos | **KDF** con sal (Argon2id de preferencia) | Enumerable por diccionario: cada intento tiene que costar caro |
| PIN o código corto elegido | La persona | 13-20 bits | **KDF** + límite de intentos estricto | La KDF ayuda, pero no basta: ver el apartado de códigos cortos |
| OTP de 6 dígitos | La máquina | ~20 bits | **KDF** o hash rápido, y siempre con caducidad, uso único y límite de intentos | La entropía es baja por diseño; lo que protege es el control de intentos |
| Código de recuperación largo | La máquina (CSPRNG) | 128 bits o más | **SHA-256** | No hay diccionario para 2¹²⁸; solo hace falta no guardarlo en claro |
| Token de sesión opaco | La máquina (CSPRNG) | 256 bits | **SHA-256** | Igual que el anterior, y además se verifica en cada petición |
| Clave de API | La máquina (CSPRNG) | 256 bits | **SHA-256**, con prefijo identificable en claro | Mismo caso; el prefijo sirve para detectarla si se filtra |
| Token firmado (JWT) | La máquina | No aplica | **No se guarda ni se hashea** | Se verifica por su firma; ver «Cuándo NO aplica» |

Un CSPRNG es un *generador de números pseudoaleatorios criptográficamente seguro*: el del sistema operativo, al que en .NET se accede con `RandomNumberGenerator`. Todo el razonamiento de las filas de 128 y 256 bits descansa en eso. Un token generado con `Random` no tiene 256 bits de entropía: tiene los del estado interno del generador, que se puede reconstruir observando unas cuantas salidas.

```bash
# Un token de 256 bits desde consola: 32 bytes del generador del sistema, en Base64
openssl rand -base64 32
# → 8Kq3vN1pR7mYd0hZaF4LcS2XeJ9wT6bU1oQ5rV8yG3k=
```

Esos 44 caracteres son lo que viaja en la cookie. En la base de datos no se guarda ni uno.

## El flujo del token opaco, completo

Un token **opaco** es una cadena sin significado: no lleva datos dentro, solo sirve para buscar una fila. Su ciclo de vida tiene cuatro pasos y el orden importa.

Primero, la tabla. `TokenHash` es binario de 32 bytes —la salida exacta de SHA-256— con índice único, porque la búsqueda en cada petición se hace por esa columna:

```sql
CREATE TABLE Sesiones (
    Id          UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    ClienteId   INT              NOT NULL REFERENCES Clientes(Id),
    TokenHash   BINARY(32)       NOT NULL,
    CreadaEn    DATETIME2        NOT NULL,
    ExpiraEn    DATETIME2        NOT NULL,
    RevocadaEn  DATETIME2        NULL
);

CREATE UNIQUE INDEX IX_Sesiones_TokenHash ON Sesiones (TokenHash);
```

Al iniciar sesión: generar, entregar el token en claro **una sola vez** y guardar solo su hash.

```csharp
// 1. Generar 32 bytes (256 bits) de un generador criptográfico
byte[] token = RandomNumberGenerator.GetBytes(32);

// 2. El valor en claro sale de la aplicación aquí y nunca más
string tokenEnClaro = Convert.ToBase64String(token);

// 3. Lo único que se persiste es el hash
db.Sesiones.Add(new Sesion
{
    ClienteId = cliente.Id,
    TokenHash = SHA256.HashData(token),
    CreadaEn  = DateTime.UtcNow,
    ExpiraEn  = DateTime.UtcNow.AddHours(8)
});
await db.SaveChangesAsync();

Response.Cookies.Append("sesion", tokenEnClaro, new CookieOptions
{
    HttpOnly = true, Secure = true, SameSite = SameSiteMode.Lax,
    Expires = DateTimeOffset.UtcNow.AddHours(8)
});
```

Tras ese bloque, `tokenEnClaro` solo existe en el navegador de quien acaba de entrar. Si mañana se filtra la tabla `Sesiones`, el atacante tiene 200 000 huellas y ninguna llave: no puede reconstruir un token de 256 bits a partir de su SHA-256, y no puede adivinarlo porque no hay nada que adivinar.

En cada petición: hashear lo recibido y buscar el hash.

```csharp
// El token llega en claro desde la cookie; se hashea igual que al crearlo
byte[] recibido  = Convert.FromBase64String(Request.Cookies["sesion"]!);
byte[] candidato = SHA256.HashData(recibido);

var sesion = await db.Sesiones.SingleOrDefaultAsync(s =>
    s.TokenHash == candidato
    && s.RevocadaEn == null
    && s.ExpiraEn > DateTime.UtcNow);

if (sesion is null) return Unauthorized();
```

Que traducido a SQL es una búsqueda por índice único, y por tanto constante en tamaño de tabla:

```sql
SELECT ClienteId, ExpiraEn
FROM   Sesiones
WHERE  TokenHash = @candidato
  AND  RevocadaEn IS NULL
  AND  ExpiraEn > SYSUTCDATETIME();
```

Dos detalles que se pasan por alto:

- **Las condiciones de caducidad y revocación van en el `WHERE`**, no en un `if` después. Filtrar en memoria funciona hasta el día en que alguien añade un `return` antes de la comprobación.
- **Si por lo que sea recuperas la fila por otro criterio y comparas hashes en código, usa `CryptographicOperations.FixedTimeEquals`**, nunca `==`. El porqué está en [Hash criptográfico](Hash-Criptografico.md).

## El cálculo que justifica el hash rápido

Aquí es donde el criterio deja de ser teórico. Validar una contraseña ocurre **una vez por login**; validar un token de sesión ocurre **en cada petición**, y una página de la tienda dispara veinte llamadas a la API. Pon números a las dos opciones para una API que sirve 500 peticiones autenticadas por segundo:

```text
Argon2id (m=19 MiB, ~50 ms por verificación)
  CPU:      500 × 0,050 s  =  25 segundos de CPU por cada segundo de reloj
  Memoria:  500 × 19 MiB   =  9 500 MiB  ≈  9,3 GB simultáneos

SHA-256 sobre 32 bytes (~1 microsegundo)
  CPU:      500 × 0,000001 s  =  0,5 milisegundos de CPU por segundo
  Memoria:  irrelevante, unos cientos de bytes
```

Veinticinco segundos de CPU por segundo significa que harían falta **veinticinco núcleos dedicados solo a hashear tokens**, más 9 GB de RAM que aparecen y desaparecen 500 veces por segundo. Con SHA-256, el coste es el 0,05 % de un núcleo. El argumento es demoledor, y lo mejor es que **no se paga nada por él en seguridad**: contra 2²⁵⁶ candidatos uniformes no hay diccionario, no hay *rainbow table* y no hay hardware. La KDF no protegería el token de nada de lo que SHA-256 no lo proteja ya.

Es exactamente el razonamiento inverso al de la contraseña, y por eso conviene tener los dos lados juntos:

| | Contraseña en `Clientes.PasswordHash` | Token en `Sesiones.TokenHash` |
|---|---|---|
| Frecuencia de verificación | 1 por login | 1 por **cada** petición |
| Entropía del secreto | 20-30 bits efectivos | 256 bits |
| ¿Hay diccionario posible? | Sí, y es barato | No |
| Coste deseado por verificación | Alto, a propósito | El mínimo posible |
| Algoritmo | Una KDF con sal | SHA-256 |

## Un caso de referencia

Una API de tienda online con **login propio y sesiones en base de datos** tiene que resolver las dos columnas a la vez. La decisión que encaja es esta:

- **Contraseñas → una KDF con sal**, con rehash oportunista: al validar un login tienes por un instante la contraseña en claro, que es el único momento en que puedes recalcular el hash con parámetros más altos y actualizar la fila sin que nadie lo note. Encaja porque una contraseña es adivinable por diccionario y necesita lentitud configurable, que es justo lo que la KDF ofrece. El detalle de qué función elegir y con qué parámetros está en [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md).
- **Tokens de sesión → opacos, de 256 bits aleatorios, guardados como SHA-256.** Encaja porque no hay nada que adivinar: ningún diccionario cubre 2²⁵⁶ valores uniformes, así que la lentitud no aporta nada — y el hash rápido permite validar la sesión en cada petición casi gratis, protegiendo igualmente los tokens si la tabla se filtra.

**Dos algoritmos distintos en la misma funcionalidad, y ninguno es «mejor» que el otro.** Cada uno responde a la entropía del secreto que protege. Quien intente unificar («usemos Argon2 para todo, que es el más seguro») tumba la API; quien unifique al contrario («SHA-256 para todo, que es el más rápido») regala las contraseñas en la primera filtración.

## Los códigos cortos son el caso difícil

Un OTP de seis dígitos —el código que llega por SMS o por correo— tiene **un millón de posibilidades**. Ese número parece grande escrito y no lo es en absoluto:

```text
1 000 000 de candidatos
  ÷ 10 intentos por segundo desde un script  →  ~28 horas para agotarlos
  con 1 000 clientes atacados en paralelo    →  un acierto casi inmediato
```

La segunda línea es la que sorprende. El atacante no necesita reventar *tu* código: le basta probar el mismo código contra mil cuentas para que en alguna acierte. Es el ataque inverso al de diccionario y esquiva cualquier límite pensado por cuenta.

Y aquí está lo importante: **ni el hash lento ni el rápido resuelven este caso**. El hash protege el código *almacenado* frente a una filtración de la base de datos, que es un problema secundario cuando el código caduca en cinco minutos. El ataque real es *online*, contra el endpoint de verificación, y contra eso el algoritmo no hace nada. Lo que resuelve es otra cosa:

- **Límite de intentos duro**, y por código, no solo por IP: cinco fallos y el código se invalida, obligando a pedir uno nuevo. Es la defensa principal, no una mejora.
- **Caducidad corta**: cinco o diez minutos. Reduce la ventana en la que el millón de candidatos vale para algo.
- **Uso único**: en cuanto se acierta, el código se marca como consumido. Sin esto, un código válido durante diez minutos se puede reutilizar.
- **Límite global de emisión**, para que nadie pida cien códigos y multiplique por cien sus oportunidades.

Aquí se equivoca mucha gente al creer que el algoritmo la salva. Un OTP guardado con Argon2id y sin límite de intentos es una cuenta abierta; un OTP guardado en claro pero de un solo uso, con caducidad de cinco minutos y tres intentos, es razonablemente seguro. **Si tienes que elegir dónde poner el esfuerzo en un código corto, ponlo en el control de intentos.**

## Cuándo NO aplica este criterio

**Los tokens firmados no entran.** Un [JWT](../autenticacion-y-autorizacion/JWT.md) no se guarda en la base de datos ni se hashea: lleva sus datos dentro y se verifica comprobando su firma con la clave del servidor. No hay ninguna fila que buscar, así que la pregunta de la entropía no se le puede hacer. Este criterio es para tokens **opacos**, que son los que existen como fila en una tabla; la comparación entre los dos modelos está en [Sesiones vs Tokens](../autenticacion-y-autorizacion/Sesiones-vs-Tokens.md) y [Tokens opacos de sesión](../autenticacion-y-autorizacion/Tokens-Opacos.md).

**Y el hash no sustituye al resto de las defensas.** Elegir bien el algoritmo protege el secreto *almacenado*, y nada más. Siguen haciendo falta:

- **TLS**, porque un token de 256 bits perfectamente hasheado en la base de datos viaja en claro por la red si la conexión no está cifrada.
- **Límite de intentos en el login**, que es la defensa contra quien prueba contraseñas contra tu endpoint. La KDF protege el día después de la filtración; el *rate-limiting*, el día de antes.
- **Caducidad de sesiones y revocación**, para que un token robado tenga fecha de fin y se pueda invalidar a mano.
- **Un CSPRNG de verdad**, del que depende literalmente todo el razonamiento del token.

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| El login tarda dos segundos | Se copiaron parámetros de KDF pensados para cifrado de disco, o se calibró en una máquina más rápida que la de producción |
| Validar la sesión consume toda la CPU y la API se cae bajo carga normal | Hay una KDF en el camino de **cada** petición: 500 peticiones por segundo a 50 ms son 25 núcleos |
| La columna `TokenHash` de `Sesiones` contiene cadenas legibles idénticas a la cookie | El token se guarda en claro: quien lea la tabla puede iniciar sesión como cualquier cliente |
| Dos sesiones creadas a la vez reciben tokens parecidos, o se pueden predecir | El token se generó con `Random` en lugar de `RandomNumberGenerator` |
| Un atacante consigue adivinar tokens midiendo tiempos de respuesta | El hash se compara con `==` en código en lugar de con `FixedTimeEquals` |
| Una cuenta se compromete por un código de verificación acertado | OTP de 6 dígitos sin límite de intentos ni uso único: un millón de candidatos son enumerables |
| Alguien entra con una sesión que abrió antes de autenticarse | *Session fixation*: no se emitió un token nuevo al iniciar sesión |
| La búsqueda de la sesión hace un escaneo completo de la tabla | Falta el índice único en `TokenHash`, o se guardó el hash con sal y hay que probar fila a fila |

## Buenas prácticas avanzadas

- **Dale a cada tipo de token un prefijo identificable.** Un token `sk_live_...` o `ghp_...` no es más seguro criptográficamente, pero cambia la operativa: el *secret scanning* de GitHub y de las herramientas de CI lo detecta si acaba en un commit, los logs se pueden depurar por patrón y, en un incidente, sabes al instante qué tipo de credencial se ha expuesto y qué revocar. El prefijo va en claro y no reduce la entropía, porque los 256 bits aleatorios van detrás.
- **A los tokens aleatorios no se les pone sal, y es deliberado.** El impulso de «hashear siempre con sal» aquí rompe la funcionalidad: con una sal distinta por fila, buscar un token exige hashear el candidato **una vez por cada fila** de `Sesiones` y comparar, es decir, un escaneo completo en cada petición. Y no se pierde nada, porque la sal existe para impedir que el atacante reutilice trabajo entre cuentas con la misma contraseña — un problema que no puede darse cuando cada token es un valor único de 256 bits. Las tres cosas que se confunden con la sal están en [Sal vs seed](Sal-Vs-Seed.md).
- **Parte las claves de API en identificador público y verificador secreto.** En lugar de una sola cadena, emite `ak_7f3c1a.<32 bytes aleatorios>`: guardas el identificador en claro con índice y el SHA-256 del verificador aparte. Ganas tres cosas concretas: puedes mostrar en el panel qué claves existen y cuándo se usaron sin conocerlas, puedes rotar una sin tocar las demás, y la comparación del verificador se hace en tiempo constante con `CryptographicOperations.FixedTimeEquals` sobre una fila que ya has localizado por el identificador. Es el patrón que usan las plataformas grandes, y es lo que permite tener metadatos por clave.
- **Verifica el hash del token en tiempo constante cuando lo compares en código.** Si recuperas la fila por otro criterio y luego comparas hashes, usa `FixedTimeEquals` y nunca `==`: la comparación ordinaria termina en el primer byte distinto y filtra información por el tiempo que tarda. Buscar directamente con `WHERE TokenHash = @candidato` es aceptable justo porque SHA-256 impide al atacante controlar qué bytes se comparan: no puede construir un token cuyo hash se parezca al de otro.
- **Rota el token en cada cambio de privilegio.** Emite un token nuevo, invalidando el anterior, al iniciar sesión, al elevar permisos y al cambiar la contraseña. Corta de raíz la *session fixation*: un token que el atacante logró plantar o capturar antes de la autenticación deja de valer justo cuando la sesión empieza a tener valor. Es una línea de código y cierra una familia entera de ataques que ningún algoritmo de hash puede tocar.

## Documentación oficial

- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) — la fuente canónica de este lado del criterio. Ve directo al apartado *Session ID Properties*: fija el mínimo de 128 bits de entropía, exige que el identificador se genere con un CSPRNG y detalla la renovación en cada cambio de privilegio. Consúltala antes de diseñar la tabla `Sesiones`.
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — la fuente canónica del otro lado, y la única cuyos parámetros conviene copiar, porque se actualiza cuando cambia el hardware. Ve ahí cada vez que vayas a fijar el coste de la KDF, en lugar de reutilizar los números de un proyecto anterior.
- [NIST SP 800-63B — Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html) — el documento normativo si tienes requisitos de cumplimiento. Su apartado 5.2.2 (*Rate Limiting*) es el que respalda con cifras el apartado de códigos cortos de esta ficha, y el 5.1.1 explica por qué la longitud mínima manda sobre las reglas de composición de caracteres.

## Recursos didácticos

- [Demo interactiva de zxcvbn](https://lowe.github.io/tryzxcvbn/) — escribe una contraseña y te dice cuántas conjeturas necesitaría un atacante y cómo la ha descompuesto (palabra de diccionario, año, sustitución de letras por símbolos). Es la forma más rápida de entender qué significa «bits **efectivos**»: prueba `Verano2024!` y luego `q7#Lm2$xR9!vB4` y compara las cifras.
- [haveibeenpwned.com/Passwords](https://haveibeenpwned.com/Passwords) — busca una contraseña entre los cientos de millones ya filtradas y te dice cuántas veces aparece. Sirve para ver que el diccionario del atacante no es hipotético: existe, está publicado y se descarga.
- [Colección de autenticación y autorización](../autenticacion-y-autorizacion/README.md) — el ciclo de vida completo de las sesiones y los tokens: emisión, renovación, revocación y la elección entre modelo opaco y firmado. Esta ficha decide cómo se guardan; esa colección decide cómo se gestionan.

---

*En resumen: la entropía del secreto elige el algoritmo — contraseña humana ⇒ KDF lenta con sal; token aleatorio de 256 bits ⇒ SHA-256 basta, porque contra 2²⁵⁶ posibilidades no hay diccionario que valga.*
