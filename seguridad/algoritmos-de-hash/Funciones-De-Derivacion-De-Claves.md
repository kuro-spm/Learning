# Funciones de derivación de claves (PBKDF2, bcrypt, scrypt, Argon2)

## ¿Qué es?

Son funciones de hash **deliberadamente lentas y configurables** diseñadas para proteger contraseñas: a partir de una contraseña, una sal aleatoria y un factor de coste, derivan un hash cuyo cálculo cuesta un tiempo apreciable — a propósito. También se las llama KDF (*key derivation functions*) o *password hashing functions*.

## ¿Por qué existe?

Un hash rápido como SHA-256 es pésimo para contraseñas. Cuando a una empresa le roban la base de datos, el atacante se lleva los hashes a su propia máquina y ya no hay nada que le limite los intentos: no hay bloqueo de cuenta, no hay `429 Too Many Requests`, no hay logs que le delaten. Puede probar contraseñas *offline* tan rápido como su hardware se lo permita, y una GPU doméstica calcula del orden de **diez mil millones de SHA-256 por segundo**.

La defensa es encarecer cada intento, y funciona porque el coste no se reparte igual entre las dos partes: **el login legítimo hace una sola verificación; el atacante hace mil millones**. Si verificar cuesta 100 ms en lugar de una fracción de nanosegundo, quien entra a la tienda no lo nota, y quien recorre un diccionario pasa de acabar en un segundo a necesitar décadas.

> Piensa en la diferencia entre una puerta normal y la puerta de una cámara acorazada con apertura retardada: abrirla una vez al día es asumible; intentar un millón de combinaciones se vuelve una condena.

## ¿Cuándo y para qué se usa?

- **Almacenar contraseñas de personas** en cualquier aplicación con registro propio: una tienda online, un SaaS, una intranet. Es su uso principal y casi único.
- **Derivar claves de cifrado a partir de una contraseña**: un gestor de contraseñas o un disco cifrado convierten la contraseña maestra en la clave AES real pasándola por una KDF. De ahí el nombre "función de derivación de claves".

El ejemplo que recorre la ficha es una **tienda online con API en .NET** y una base de datos con dos tablas: `Clientes`, que guarda la contraseña en la columna `PasswordHash`, y `Sesiones`, que guarda el identificador de sesión en `TokenHash`. Solo la primera necesita una KDF; el porqué de esa asimetría lo desarrolla [Contraseñas vs tokens de sesión](Contrasenas-Vs-Tokens-De-Sesion.md). Damos por conocidas las propiedades generales de un hash criptográfico — resistencia a preimagen, a colisiones, comparación en tiempo constante — que están en [Hash criptográfico](Hash-Criptografico.md), y usamos SHA-256 como ejemplo de "hash rápido" sin entrar en las familias, que es asunto de [Hashes de propósito general](Hashes-De-Proposito-General.md).

---

## El cálculo que lo justifica todo

Todo lo demás sale de aquí. Supón que el atacante ya tiene la tabla `Clientes` y quiere probar **10 000 millones de candidatos**: un diccionario de contraseñas filtradas con sus variantes habituales — mayúscula inicial, un número al final, `a` por `@` —, grande pero perfectamente normal. Los números son órdenes de magnitud para **una** GPU de gama alta reciente; lo que importa no es su exactitud, sino la distancia entre las filas.

| Algoritmo y parámetros | Hashes/s (una GPU) | Agotar 10 000 M de candidatos |
|---|---|---|
| SHA-256 a pelo | ~10 000 000 000 | **~1 segundo** |
| SHA-256 iterado 1 000 veces | ~10 000 000 | ~15 minutos |
| PBKDF2-HMAC-SHA256, 600 000 iteraciones | ~10 000 | ~12 días |
| Argon2id, 19 MiB, t=2, p=1 (mínimo OWASP) | ~4 000 | ~1 mes |
| bcrypt, factor de trabajo 12 | ~1 500 | ~2,5 meses |
| Argon2id, 64 MiB, t=3, p=1 | ~500 | ~8 meses |
| scrypt, N=2¹⁷, r=8, p=1 (128 MiB) | ~300 | ~1 año |

- **La primera fila es el escenario real de casi todas las brechas históricas.** Un segundo. No "es más débil": es que no hay defensa en absoluto.
- **El mínimo de Argon2id que recomienda OWASP no es el techo, es el suelo.** Con 19 MiB queda en el mismo orden que bcrypt-12, y por debajo de scrypt con 128 MiB. Subir la memoria es lo que separa un mes de ocho.

Y el otro lado de la moneda, que es lo que hace la idea comprensible:

| | Verificaciones que ejecuta | Coste total a 100 ms cada una |
|---|---|---|
| El login de una clienta | 1 | 0,1 segundos |
| El atacante con el diccionario | 10 000 000 000 | ~31 años de un núcleo |

Ese factor de diez mil millones es todo el argumento. Encarecer la operación es **asimétrico a tu favor**: le cobras a quien intenta una vez lo mismo que a quien intenta mil millones, y solo uno de los dos lo nota. Un atacante con dinero acorta el plazo comprando hardware — ocho GPUs convierten los 2,5 meses de bcrypt-12 en diez días — pero eso es exactamente lo que buscabas: que atacarte tenga un precio en euros.

## El atacante no revierte el hash: adivina la contraseña

Este punto se enuncia mal casi siempre, y de él dependen todas las decisiones que vienen después. Un hash no se "desencripta": nadie coge `a3f9...` y saca la contraseña de dentro. Lo que hace el atacante es **generar candidatos, hashearlos y comparar**. Es un juego de adivinar, y por eso la velocidad del hash es la variable que decide el resultado: no cambia si la contraseña es adivinable, cambia cuántas adivinanzas caben en un día.

Y las contraseñas humanas son extraordinariamente adivinables, porque viven en un espacio pequeño y sesgado:

```text
"123456"                          → puesto 1 del diccionario
"Verano2024!"                     → patrón palabra + año + símbolo, muy explorado
8 caracteres minúsculas (26^8)    → 208 827 064 576 combinaciones
4 palabras de una lista de 7776   → 3 656 158 440 062 976 combinaciones
```

Combinando cada línea con la tabla anterior:

- `"123456"` cae en el **primer** intento, con Argon2id de 1 GiB y con lo que quieras. Una KDF no arregla una contraseña mala.
- Las 26⁸ combinaciones de ocho minúsculas se agotan enteras con SHA-256 en **21 segundos**. Con Argon2id a 4 000 hashes/s, en unos 1 650 años.
- Las cuatro palabras aleatorias, con esa misma Argon2id, están a **29 000 años**.

De ahí sale el objetivo realista, más modesto de lo que suena el discurso habitual: **una KDF no salva las contraseñas malas ni hace falta para las excelentes; protege a la enorme mayoría que están en medio.** `Verano2024!` con SHA-256 es una cuenta perdida, y con Argon2id bien configurada es una cuenta que probablemente aguante. Ahí se juega el partido, y ahí está casi todo el mundo.

Corolario práctico: la KDF **no sustituye al control de intentos online**. Contra alguien que prueba contraseñas contra tu endpoint de login sigues necesitando rate-limiting y bloqueo de cuenta. La KDF es la defensa para el día después de la filtración.

## La sal: qué es y qué dos ataques impide

La **sal** (*salt*) es un valor aleatorio, único por contraseña, que se mezcla con ella antes de hashear y **se guarda en claro** junto al hash. Lo primero que sorprende es eso: no es un secreto, y no hace falta que lo sea, porque su trabajo no es esconder nada sino **impedir que el atacante reutilice trabajo**. Sin sal, el `PasswordHash` de dos clientas con la misma contraseña es idéntico, y esas dos filas iguales delatan dos cosas a la vez: que ambas usan el mismo secreto, y que su hash ya está calculado en algún sitio.

```text
❌ Sin sal — tabla Clientes
ana@ejemplo.es      8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
lucia@ejemplo.es    8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92

✅ Con sal única — tabla Clientes
ana@ejemplo.es      $argon2id$v=19$m=19456,t=2,p=1$C7RPZ4qRolZ5...$5d6P0PU3q2kX...
lucia@ejemplo.es    $argon2id$v=19$m=19456,t=2,p=1$Qk1vTn9xWmEy...$8bYc2Lm1RtQz...
```

**1. Con sal, las *rainbow tables* dejan de servir.** Una *rainbow table* es un diccionario precalculado: contraseña → hash, hecho una vez y consultado millones de veces. Precalcular es rentable porque el hash de `"Verano2024!"` es siempre el mismo... salvo que haya sal. Con 16 bytes de sal aleatoria hay 2¹²⁸ hashes posibles para esa contraseña, así que la tabla habría que construirla **para tu sal concreta**, y entonces ya no es una tabla precalculada: es el ataque de fuerza bruta de siempre.

**2. El ataque en masa se convierte en muchos ataques individuales.** Sin sal, el atacante calcula el hash de un candidato **una vez** y lo busca contra las 200 000 filas de `Clientes` de golpe. Con sal única tiene que calcularlo **una vez por fila**, porque cada una lleva una sal distinta. El coste se multiplica por el número de cuentas: 10 000 millones de candidatos contra 200 000 clientas son 2·10¹⁵ operaciones en lugar de 10¹⁰.

En la práctica **no vas a manejar la sal a mano**: las cuatro funciones de esta ficha la generan, la guardan dentro de su cadena de salida y la recuperan al verificar. Lo único que debes garantizar es que, si alguna vez la generas tú, salga de un generador criptográfico:

```csharp
var sal = new byte[16];
new Random().NextBytes(sal);                    // ❌ secuencia reproducible

var sal = RandomNumberGenerator.GetBytes(16);   // ✅ 16 bytes impredecibles del sistema
```

`Random` está pensado para simulaciones y barajar listas: quien conozca la semilla reconstruye todas tus sales. `RandomNumberGenerator` pide entropía al sistema operativo, y 16 bytes es el mínimo razonable. Por cierto: la palabra "sal" se confunde a menudo con la *seed* de datos de prueba y con la semilla de un generador aleatorio, que son tres cosas sin relación entre sí; están separadas en [Sal vs seed](Sal-Vs-Seed.md).

## El factor de coste: calibrar la lentitud

Toda KDF expone un ajuste de coste, y cada una lo llama de otra forma:

| Función | Qué se ajusta | Cómo escala |
|---|---|---|
| PBKDF2 | Número de iteraciones | Lineal: 600 000 iteraciones cuestan el doble que 300 000 |
| bcrypt | Factor de trabajo (*cost*) | **Exponencial**: cada +1 duplica el tiempo. De 10 a 12 es ×4 |
| scrypt | `N` (memoria/CPU), `r` (tamaño de bloque), `p` (paralelismo) | `N` es exponencial; la memoria usada es ≈ 128 · N · r bytes |
| Argon2id | `m` (memoria en KiB), `t` (iteraciones), `p` (hilos) | Lineal en `m` y en `t`; el producto es lo que cuenta |

La regla de calibración es sencilla: **apunta a decenas o centenas de milisegundos de verificación en el servidor de producción**, no en tu portátil. De 100 a 250 ms es el compromiso habitual entre seguridad y una latencia de login que nadie percibe. Y se mide, no se estima:

```csharp
// Calibración: sube el parámetro hasta entrar en el rango objetivo
var reloj = Stopwatch.StartNew();
var hash = DerivarHash("una contraseña de prueba", memoriaKiB: 19456, iteraciones: 2, paralelismo: 1);
reloj.Stop();
Console.WriteLine($"{reloj.ElapsedMilliseconds} ms");   // → p. ej. "58 ms"
```

Ejecútalo **en el contenedor de producción** y repítelo unas cuantas veces descartando la primera medida, que el arranque en frío distorsiona. La diferencia no es teórica: el parámetro que da 58 ms en un portátil reciente puede dar 400 ms en un contenedor con una fracción de núcleo asignada.

Este número caduca: el hardware mejora, así que un coste calibrado en 2019 protege menos hoy y hay que revisarlo cada uno o dos años. Que eso se pueda hacer sin romper los logins antiguos es exactamente lo que resuelve el formato autodescriptivo.

## Memory-hard: por qué una GPU no puede con Argon2

Una GPU es devastadora contra PBKDF2 por una razón muy concreta: PBKDF2 solo quema CPU. Es una cadena de HMAC-SHA256 que cabe en unos pocos registros, así que **cada uno de los miles de núcleos de la tarjeta puede ejecutar su propia instancia** sin estorbar a los demás. Una GPU con 16 000 núcleos ataca 16 000 contraseñas a la vez. Y ese mismo argumento se agrava con hardware dedicado: una FPGA o un ASIC diseñados para SHA-256 llevan la asimetría todavía más lejos, porque el circuito se construye para hacer solo eso.

Una función **memory-hard** rompe ese paralelismo cambiando el recurso escaso. En lugar de exigir cálculo, exige **memoria**: Argon2 rellena un bloque de memoria de tamaño configurable y lo recorre con accesos dependientes de lo anterior, de modo que no puedes hacer el cálculo con menos RAM ni saltarte pasos. Haz la cuenta con el mínimo de OWASP:

```text
Argon2id con m = 19456 KiB  →  19 MiB de memoria por cada hash en curso

GPU con 8 GB:   8192 MiB / 19 MiB  ≈    430 cálculos en paralelo
GPU con 24 GB: 24576 MiB / 19 MiB  ≈  1 290 cálculos en paralelo
```

Cientos, no decenas de miles. La tarjeta tiene los núcleos, pero no tiene dónde ponerlos a trabajar: el límite pasa a ser la capacidad y el ancho de banda de memoria, que es justo donde una GPU no es órdenes de magnitud mejor que una CPU. Y sube el precio de fabricar hardware específico, porque un ASIC o una FPGA memory-hard necesitan RAM de verdad, no solo puertas lógicas.

Por eso `m` es el parámetro que más rinde de Argon2. Pasar de 19 MiB a 64 MiB triplica el coste de tu servidor por login, pero divide por más de tres el paralelismo del atacante **y** encarece cada uno de sus cálculos.

## Las cuatro candidatas

| Función | Año | ¿Memory-hard? | Parámetros que expone | Limitaciones propias |
|---|---|---|---|---|
| **PBKDF2** | 2000 | No | Iteraciones, función hash interna, longitud de salida | La más débil frente a GPUs y ASIC: solo quema CPU y se paraleliza de maravilla. Correcta con iteraciones muy altas. Su ventaja es que está en todas las plataformas y es la única con bendición NIST/FIPS |
| **bcrypt** | 1999 | No, pero usa 4 KB de tabla, algo hostil a GPUs | Factor de trabajo (2ⁿ) | **Trunca la contraseña a 72 bytes.** Algunas implementaciones se cortan además en el primer byte nulo. Sin parámetro de memoria: si el atacante consigue hardware, solo puedes subir CPU |
| **scrypt** | 2009 | Sí | N, r, p, longitud de salida | Primera memory-hard popular y sigue siendo sólida. Los tres parámetros interactúan de forma poco intuitiva y es fácil configurarla con poca memoria por error, lo que la deja al nivel de PBKDF2 |
| **Argon2id** | 2015 | Sí | m (memoria), t (iteraciones), p (paralelismo), longitud de salida | Ganadora de la Password Hashing Competition y recomendación actual (RFC 9106). Más joven que las otras, y hay que elegir bien la variante |

**Las tres variantes de Argon2**, porque una librería te ofrecerá las tres y solo una es la respuesta:

| Variante | Accesos a memoria | Resiste GPUs | Resiste canal lateral | Para contraseñas |
|---|---|---|---|---|
| Argon2**d** | Dependientes de los datos | Mejor | **No** — el patrón de accesos filtra información sobre la contraseña | No |
| Argon2**i** | Independientes de los datos | Peor | Sí | No |
| Argon2**id** | Híbrido: primera pasada independiente, resto dependiente | Bien | Sí | **Sí** |

Un ataque de **canal lateral** es el que no rompe las matemáticas sino que observa el comportamiento: qué direcciones de memoria se tocan, cuánto tarda, qué consume. En un servidor compartido, un proceso vecino puede deducir cosas del patrón de accesos de Argon2d. Argon2id hace la primera pasada con accesos independientes de los datos y las siguientes dependientes, y consigue lo razonable de ambas. Si tu librería te da a elegir, **no hay decisión que tomar**: `id`.

## Los 72 bytes de bcrypt, con demostración

bcrypt ignora todo lo que pase del byte 72 de la contraseña. No avisa, no lanza una excepción, no trunca visiblemente: simplemente el resto no entra en el cálculo, y la consecuencia sorprende a todo el mundo.

```csharp
// Los primeros 72 bytes son idénticos; solo difieren a partir del 73
var registrada = "Esta-es-la-frase-de-paso-larguisima-que-uso-en-la-tienda-online-2024-ab" + "PRIMERA";
var inventada  = "Esta-es-la-frase-de-paso-larguisima-que-uso-en-la-tienda-online-2024-ab" + "OTRACOSA";

var hash = BCryptHash(registrada, factorDeTrabajo: 12);
Console.WriteLine(BCryptVerify(inventada, hash));   // → True
```

`BCryptVerify` devuelve `True` con una contraseña que **no es** la que se registró. A partir del byte 73, bcrypt no mira. Cualquier persona que use un gestor de contraseñas con frases largas tiene una cuenta cuya seguridad real acaba en el carácter 72 — y quien conozca solo ese prefijo entra.

Si eliges bcrypt sabiendo esto, hay dos salidas: **rechazar en el registro las contraseñas de más de 72 bytes** (honesto y feo), o **pre-hashear**, que parece obvio y tiene dos trampas serias explicadas en las buenas prácticas del final. Con Argon2id el problema no existe: acepta cualquier longitud. Y cuidado con la aritmética, porque son 72 **bytes**: en UTF-8 una `ñ` ocupa dos y un emoji cuatro.

## Las recomendaciones de OWASP

La *Password Storage Cheat Sheet* de OWASP ordena las opciones así, y el orden importa tanto como los números:

1. **Argon2id** — mínimo orientativo: 19 MiB de memoria (`m=19456`), 2 iteraciones (`t=2`), paralelismo 1 (`p=1`).
2. **scrypt**, si Argon2 no está disponible — `N=2^17`, `r=8`, `p=1` (unos 128 MiB).
3. **bcrypt**, en sistemas heredados o cuando la plataforma no ofrece nada mejor — factor de trabajo 10 como mínimo; 12 si el servidor lo aguanta.
4. **PBKDF2**, cuando se exige cumplimiento FIPS-140 — con HMAC-SHA256, unas 600 000 iteraciones.

Y en los cuatro casos: sal única por contraseña (la librería lo hace sola) y coste recalibrado periódicamente. **Estos números caducan, y ese es el punto que más se olvida.** Retratan lo que cuesta el hardware ahora mismo, no son constantes. Antes de fijar parámetros, abre la *cheat sheet* vigente y mira qué dice hoy en lugar de copiar los valores de una guía — incluida esta. Un `m=19456` escrito en 2025 y no revisado hasta 2032 protege bastante menos de lo que su autor creía.

## El formato de salida autodescriptivo

Estas funciones no devuelven solo el hash. Devuelven una cadena que lleva dentro todo lo necesario para volver a calcularlo:

```text
$argon2id$v=19$m=19456,t=2,p=1$C7RPZ4qRolZ5b1lQZ2s2dw$5d6P0PU3q2kXvHhKZ1uYcQ
 └───┬──┘ └─┬─┘ └──────┬──────┘ └──────────┬───────┘ └──────────┬─────────┘
  variante  │      parámetros:          la sal,              el hash,
            │   memoria en KiB (m),   en Base64            en Base64
     versión del   iteraciones (t),   y en claro
   algoritmo (19)  paralelismo (p)
```

Es el formato **PHC** (*Password Hashing Competition*), y lo comparten Argon2, scrypt y las bcrypt modernas — bcrypt usa una variante más antigua, `$2b$12$...`, con el factor y la sal pero sin nombres de parámetro. Todo esto va en una sola columna `PasswordHash`; **no guardes la sal ni los parámetros en columnas aparte**, o tarde o temprano perderás la correspondencia entre fila y sal.

Lo valioso es que puedes subir el coste sin migraciones traumáticas, porque al verificar un login tienes por un instante algo que no vuelves a tener nunca: **la contraseña en claro**. Es el único momento en que puedes recalcular el hash con parámetros nuevos.

```csharp
// Flujo de login con rehash oportunista
var cliente = await db.Clientes.SingleOrDefaultAsync(c => c.Email == email);
if (cliente is null || !VerificarHash(contrasena, cliente.PasswordHash))
    return Unauthorized();

// La contraseña es correcta y la tenemos en claro justo aquí: aprovecha
if (NecesitaRehash(cliente.PasswordHash, memoriaKiB: 65536, iteraciones: 3))
{
    cliente.PasswordHash = DerivarHash(contrasena, memoriaKiB: 65536, iteraciones: 3);
    await db.SaveChangesAsync();
}
```

`NecesitaRehash` parsea la cadena, lee `m=19456,t=2` y compara con los parámetros actuales; si son inferiores, devuelve `true`. La clienta entra igual, sin notar nada, y su fila queda migrada. Con las cuentas activas eso ocurre en semanas; para las inactivas, el apartado de migración de más abajo.

## El login como vector de denegación de servicio

Una KDF cara tiene un efecto secundario que casi nunca se anticipa: convierte tu endpoint de login en el sitio más caro de toda la API, y **cualquiera puede llamarlo sin autenticarse**. Haz la cuenta:

```text
Verificar una contraseña:  100 ms de CPU
Núcleos disponibles:       4
Capacidad máxima:          4 / 0,1 s  =  40 verificaciones por segundo
```

Cuarenta por segundo, y eso dedicando la máquina entera a hashear. Unas pocas decenas de peticiones concurrentes con emails reales y contraseñas inventadas agotan la CPU, y a partir de ahí no se cae solo el login: se cae el catálogo, el carrito y la pasarela de pago, porque compiten por los mismos núcleos. No hace falta una botnet, basta un script. Y peor si aceptas contraseñas de longitud arbitraria: pon un tope (128 o 256 caracteres) y rechaza por encima.

Las tres mitigaciones se acumulan, no se eligen:

- **Rate-limiting por IP**, para el atacante ruidoso desde un solo origen.
- **Rate-limiting por cuenta**, porque el anterior no sirve contra peticiones distribuidas, y además es lo que frena el ataque de adivinar contraseñas online.
- **Límite de verificaciones concurrentes**, que es la que de verdad salva el servidor:

```csharp
// Como máximo 8 verificaciones de contraseña simultáneas en todo el proceso
private static readonly SemaphoreSlim Puerta = new(initialCount: 8);

await Puerta.WaitAsync(cancellationToken);
try     { esValida = VerificarHash(contrasena, cliente.PasswordHash); }
finally { Puerta.Release(); }
```

Con el semáforo, el exceso de peticiones **espera en cola** en lugar de saturar los núcleos: los logins se degradan en latencia y el resto de la aplicación sigue respondiendo. Sin él, todo cae junto. Ajusta el número al de núcleos disponibles y pasa un `CancellationToken` con timeout para que la cola no crezca sin límite.

## Migrar hashes heredados

Heredas la tabla `Clientes` con `PasswordHash` en MD5 o SHA-1 sin sal. El reflejo habitual es "rehasheamos a cada persona cuando vuelva a entrar", y es un mal plan: **las cuentas inactivas — que son muchas y suelen tener las peores contraseñas — se quedan en MD5 para siempre.** Si la filtración ocurre mañana, esas están perdidas. La alternativa es proteger toda la tabla hoy mismo, envolviendo el hash antiguo en un proceso por lotes. No hace falta la contraseña original, solo una columna que anote **qué hay envuelto**:

```csharp
// 1. Migración por lotes: 5f4dcc3b... (md5) pasa a $argon2id$...$ que envuelve ese md5
foreach (var cliente in lote)
{
    cliente.PasswordHash = DerivarHash(cliente.PasswordHash);   // argon2id(md5(contraseña))
    cliente.EsquemaPassword = "argon2id(md5)";                  // los nuevos llevan "argon2id"
}
```

```csharp
// 2. Login: el esquema decide qué se compara
bool esValida = cliente.EsquemaPassword switch
{
    "argon2id"      => VerificarHash(contrasena, cliente.PasswordHash),
    "argon2id(md5)" => VerificarHash(Md5Hex(contrasena), cliente.PasswordHash),
    _               => throw new InvalidOperationException($"Esquema desconocido: {cliente.EsquemaPassword}")
};

// 3. Si entra y venía envuelto, se sustituye por el hash directo
if (esValida && cliente.EsquemaPassword != "argon2id")
{
    cliente.PasswordHash = DerivarHash(contrasena);
    cliente.EsquemaPassword = "argon2id";
    await db.SaveChangesAsync();
}
```

El resultado: la base está protegida desde el primer lote, y el esquema envuelto va desapareciendo solo. Cuando la columna `EsquemaPassword` no tenga ni una fila con `argon2id(md5)`, borras esa rama del `switch` — y hasta entonces, el `_ => throw` te avisa si aparece un valor que no esperabas en lugar de dejar entrar a alguien por error.

## El pepper: el secreto que la sal no es

La sal es pública y está en la base de datos. Un **pepper** es lo contrario: un secreto **común a todas las cuentas** que vive **fuera** de la base de datos — variable de entorno, un KMS, un *vault*. Se aplica con un HMAC antes de pasar por la KDF:

```csharp
// El pepper NO está en la base de datos: viene de la configuración del servidor
var pepper = Convert.FromBase64String(config["Seguridad:Pepper"]!);
using var hmac = new HMACSHA256(pepper);

// Base64 y no el digest binario: un byte nulo dentro rompe algunas implementaciones
var conPepper = Convert.ToBase64String(hmac.ComputeHash(Encoding.UTF8.GetBytes(contrasena)));
cliente.PasswordHash = DerivarHash(conPepper);   // argon2id(hmac(pepper, contraseña))
```

Lo que compras es el escenario más frecuente de todos. En una inyección SQL o un backup perdido, el atacante se lleva la base de datos y **nada más**: tiene los hashes y las sales, pero sin el pepper no puede probar ni un solo candidato. El diccionario de 10 000 millones no le sirve ni para empezar. Lo que pagas, y hay que decirlo entero:

- **Gestión de una clave más**, con todo lo que eso implica: que esté en producción, que no acabe en git, que exista en cada entorno.
- **Rotación complicada.** No puedes cambiar el pepper y recalcular: haría falta la contraseña en claro de cada cuenta. La rotación real es versionar (`pepper_v1`, `pepper_v2`, guardando la versión junto al hash) y migrar en cada login, exactamente como el rehash oportunista.
- **Si se pierde, se pierden todas las contraseñas.** No hay recuperación posible: nadie puede volver a entrar y toca reinicio de contraseña para el 100 % de las cuentas. Un pepper que solo existe en una variable de entorno de un contenedor, sin copia en ningún gestor de secretos, es un incidente esperando su turno.

Por eso el pepper es una capa **adicional**, nunca un sustituto de la sal ni de una KDF bien calibrada.

## Cuándo NO usar una KDF

Una KDF lenta no es "más seguro" por defecto. Aplicada donde no toca, solo gasta CPU y memoria:

- **Tokens aleatorios de alta entropía.** El `TokenHash` de la tabla `Sesiones` guarda el hash de un token generado por ti con 256 bits de aleatoriedad. Ahí no hay nada que adivinar — un atacante tendría que recorrer 2²⁵⁶ candidatos — así que la lentitud no compra nada, y en cambio pagas 100 ms de CPU **en cada petición autenticada**, no solo en el login. Para eso, SHA-256 a secas. El criterio completo está en [Contraseñas vs tokens de sesión](Contrasenas-Vs-Tokens-De-Sesion.md).
- **Ficheros y datos.** Comprobar la integridad de una factura en PDF o deduplicar imágenes por su contenido pide un hash rápido; una KDF haría el proceso miles de veces más lento sin ganar nada, porque un fichero no es un secreto adivinable.
- **Firmas y autenticación de mensajes.** Los HMAC y las firmas digitales usan una clave real, ya aleatoria. Su hash interno debe ser rápido.

La pregunta que decide es siempre la misma: **¿la entrada es adivinable?** Si lo es, KDF; si es aleatoria de verdad, hash rápido.

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| El login tarda dos segundos | El coste se calibró en una máquina más rápida que la de producción; o falta el límite de concurrencia y las verificaciones se pelean por los núcleos |
| Dos clientas con la misma contraseña tienen el mismo `PasswordHash` | No hay sal, o la sal es una constante compartida en el código |
| Una contraseña larguísima valida acertando solo las primeras 72 letras | bcrypt truncando en 72 bytes, sin límite de longitud en el registro |
| Al restaurar un backup, ningún login valida | La sal se guardó en una columna aparte y se perdió la correspondencia con su fila |
| Dos cuentas creadas el mismo segundo comparten sal | La sal se genera con `Random` (o con el `DateTime` de creación) en lugar de con un generador criptográfico |
| Se subió el coste y los logins antiguos dejan de validar | Se cambió el algoritmo o los parámetros sin conservar el formato autodescriptivo, así que ya no se sabe con qué se calculó cada hash |
| Nadie puede entrar tras un despliegue, y no se tocó el código de login | Falta el pepper en la configuración del entorno nuevo, o cambió su valor |
| El endpoint de login tumba el servidor bajo carga y se cae toda la API | KDF cara sin rate-limiting ni semáforo: el login agota la CPU compartida |

## Buenas prácticas avanzadas

- **Calibra en el hardware de producción y trata el login como vector de DoS.** El coste que en tu portátil da 50 ms puede dar 400 ms en el contenedor pequeño de producción, y una KDF cara convierte el endpoint de login en un objetivo: unas decenas de peticiones simultáneas con contraseñas inventadas bastan para saturar la CPU. Además del rate-limiting por IP y por cuenta, limita cuántas verificaciones se ejecutan a la vez con un semáforo o una cola, y pon un tope a la longitud de la contraseña aceptada.
- **Pepper: el secreto que la sal no es.** Sobre la sal (pública, en la base de datos) puedes añadir un HMAC con una clave secreta guardada *fuera* de ella — variable de entorno, KMS — antes de pasar por la KDF. En el escenario típico de filtración, inyección SQL o backup perdido, el atacante tiene los hashes pero no el pepper, y no puede probar ni una sola contraseña. A cambio asumes gestionar y rotar esa clave, versionarla para poder cambiarla, y el hecho de que perderla equivale a perder todas las contraseñas a la vez.
- **Pre-hashear para esquivar el límite de bcrypt tiene trampa.** El apaño `bcrypt(sha256(contraseña))` introduce dos fallos sutiles: el digest binario puede contener bytes nulos que algunas implementaciones de bcrypt tratan como fin de cadena — con lo que la contraseña efectiva se queda en los pocos bytes anteriores —, y habilita el *password shucking*: si el atacante tiene SHA-256 filtrados de otra brecha, los prueba **directamente como contraseñas** contra tus bcrypt, sin necesidad de adivinar nada. Si pre-hasheas, codifica el resultado en Base64 y usa HMAC con clave en lugar de un hash a pelo.
- **Migra hashes heredados envolviéndolos, no esperando logins.** Si heredas una tabla con MD5 o SHA-1, aplica ya `argon2id(hash_viejo)` a todos los registros y anota el esquema; la base queda protegida hoy y no dentro de tres años. Esperar al siguiente login de cada persona suena elegante y deja indefinidamente en MD5 justo las cuentas inactivas, que son las que tienen las peores contraseñas. En el siguiente acceso de cada cuenta sustituyes por el Argon2id de la contraseña original.
- **De las tres variantes de Argon2, siempre la id.** Argon2d resiste mejor las GPUs pero su patrón de accesos a memoria depende de los datos y filtra información por canal lateral; Argon2i es lo contrario, seguro frente a canal lateral y más flojo ante GPUs. Argon2**id** hace la primera pasada como `i` y el resto como `d`, y es la única que recomiendan OWASP y el RFC 9106 para contraseñas: si una librería te ofrece las tres, no hay decisión que tomar.

## Documentación oficial

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — la fuente canónica de esta ficha, y **el único documento cuyos parámetros deberías copiar**, porque se actualiza cuando el hardware cambia. Consúltala cada vez que vayas a fijar un coste, en lugar de reutilizar los números de un proyecto anterior o de esta guía.
- [RFC 9106 — Argon2](https://www.rfc-editor.org/rfc/rfc9106.html) — la especificación. Ve a sus apartados de elección de parámetros y de variantes cuando necesites el razonamiento completo sobre memoria, paralelismo y canal lateral que aquí se resume en un párrafo.
- [RFC 8018 — PBKDF2](https://www.rfc-editor.org/rfc/rfc8018) y [el artículo original de scrypt](https://www.tarsnap.com/scrypt/scrypt.pdf) — las fuentes de las otras dos funciones. La del scrypt es además la que introdujo el argumento del coste en memoria, así que se lee bien como fundamento y no solo como referencia.
- [NIST SP 800-63B](https://pages.nist.gov/800-63-3/sp800-63b.html) — la normativa que consultar cuando el requisito viene de cumplimiento y no de criterio propio: qué exige sobre almacenamiento de verificadores, longitud mínima y qué prohíbe (las reglas de composición y la caducidad obligatoria).

## Recursos didácticos

- Una **calculadora interactiva de parámetros de Argon2** (busca "argon2 parameter calculator"; hay varias en web que corren en el navegador) — mueves memoria, iteraciones y paralelismo y ves el tiempo resultante. Sirve para intuir cómo escala cada parámetro, pero la calibración de verdad se hace midiendo en tu servidor.
- [haveibeenpwned.com](https://haveibeenpwned.com/Passwords) — busca una contraseña entre los cientos de millones ya filtradas y te dice cuántas veces aparece. Es la forma más directa de entender el apartado sobre adivinar en lugar de revertir: prueba `Verano2024!` y mira el número. Su API por prefijos de SHA-1 permite además rechazar en el registro contraseñas ya conocidas sin enviarlas a ningún sitio.
- Los paquetes concretos de .NET para cada una de estas funciones, con sus diferencias y licencias, están en [Hashing en C#/.NET](Hashing-En-CSharp.md).

---

*En resumen: las contraseñas se protegen con funciones lentas a propósito — Argon2id como primera opción según OWASP — porque contra un ladrón de bases de datos la única defensa real es que cada intento le cueste caro.*
