# Hashes de propósito general: SHA-2, SHA-3 y BLAKE

## ¿Qué es?

Son las familias de hash criptográfico modernas y **rápidas a propósito**, las que se usan para todo lo que no sea contraseñas: verificar integridad, firmar documentos, generar identificadores de contenido o deduplicar datos. Las tres consideradas seguras hoy son **SHA-2**, **SHA-3** y **BLAKE2/BLAKE3**.

## ¿Por qué existe?

Los algoritmos de hash envejecen. La potencia de cálculo crece, el criptoanálisis avanza, y lo que era inviable en 1995 se convierte en un ataque de fin de semana en 2020. **MD5** (1992) y **SHA-1** (1995) fueron el estándar de la industria durante casi dos décadas, y hoy los dos están roídos: se sabe fabricar dos ficheros distintos con el mismo hash de ambos. Las familias modernas existen para reemplazarlos con márgenes de seguridad mucho más amplios y, en el caso de BLAKE, además con más velocidad.

> Piensa en las familias de hash como en las cerraduras de una puerta: las antiguas no eran "malas" cuando se instalaron, pero hoy cualquier cerrajero abre una cerradura de los 90 en segundos. Se cambia la cerradura, no la puerta.

Esta ficha da por sabido **qué propiedades tiene un hash criptográfico** y cómo se usa correctamente; eso lo cubre [Hash criptográfico](Hash-Criptografico.md). Aquí se elige entre familias concretas y se justifica la elección.

## ¿Cuándo y para qué se usa?

El ejemplo que recorre la ficha es una **tienda online con API en .NET**. Su base de datos tiene una tabla `Clientes` con la columna `PasswordHash` y una tabla `Sesiones` con la columna `TokenHash`; publica un `instalador.exe` de su aplicación de escritorio junto al hash para que quien lo descargue lo verifique, y guarda vídeos de producto como `video-4gb.mp4`.

De esos cuatro casos, **solo uno no es trabajo de las familias de esta ficha**: `PasswordHash`. Los demás sí, y se detallan más adelante con su ejemplo.

---

## Por qué los algoritmos envejecen: la historia con fechas

Decir "MD5 está roto" no explica nada. Lo que hay que saber es **qué se rompió, cuándo y a qué coste**, porque de ahí sale el criterio para saber si un algoritmo viejo te afecta a ti.

| Año | Qué pasó |
|---|---|
| 1992 | Se publica MD5. Salida de 128 bits. |
| 1995 | Se publica SHA-1. Salida de 160 bits. |
| 2004-2005 | Xiaoyun Wang y su equipo publican colisiones reales de MD5 y el primer ataque teórico contra SHA-1. |
| 2008 | Un grupo de investigadores fabrica un **certificado de CA falso** aprovechando una colisión de MD5: podían firmar certificados para cualquier dominio. Aquí MD5 deja de ser discutible. |
| 2017 | *SHAttered*: Google y el CWI publican **dos PDF distintos con el mismo hash SHA-1**. Coste: unos 6.500 años-CPU y 110 años-GPU, comprados en la nube. |
| 2017 | Chrome y Firefox dejan de aceptar certificados TLS firmados con SHA-1. |
| 2019-2020 | *SHAmbles*: colisión **con prefijo elegido** contra SHA-1 por unas decenas de miles de dólares en GPU alquiladas. Los autores la usan para suplantar una clave PGP. Este es el ataque que de verdad rompe las firmas. |

Hoy fabricar una colisión de MD5 en un portátil es cuestión de **segundos**. Con SHA-1 se paga dinero, pero es dinero que cualquier organización tiene.

Un detalle que se cuenta mal a menudo: **Git no ha migrado a SHA-256**. Lo que hizo en 2017 fue incorporar una detección de colisiones (SHA-1DC) que aborta la operación si detecta la estructura característica de un ataque; el soporte de repositorios con SHA-256 existe pero sigue siendo experimental. Es un parche, no un cambio de cerradura.

## La distinción que casi nadie hace: colisión, prefijo elegido y preimagen

Esta es la sección que decide si un algoritmo roto te afecta o no, y es donde más gente se equivoca en las dos direcciones: unos creen que MD5 no sirve para nada y otros creen que sirve para todo.

Hay tres ataques distintos, en orden de dificultad creciente:

| Ataque | Qué consigue el atacante | ¿Roto en MD5/SHA-1? |
|---|---|---|
| **Colisión** | Dos mensajes `A` y `B`, **ambos elegidos por él**, con el mismo hash | Sí. MD5 en segundos, SHA-1 con presupuesto |
| **Colisión con prefijo elegido** | Lo mismo, pero **partiendo de un principio que no controla** (una plantilla, una cabecera, un texto que tú redactaste) | Sí. MD5 trivialmente, SHA-1 desde 2020 |
| **Preimagen** | Dado un hash, **encontrar cualquier entrada** que lo produzca | **No.** Sigue siendo inviable |

La diferencia entre las dos primeras es la que importa en la práctica. Una colisión "libre" solo sirve si el atacante controla los dos documentos, lo que en muchos escenarios no le da nada. El prefijo elegido es otra historia: significa que puede coger **un contrato que tú has escrito**, añadirle un bloque de basura invisible y producir un segundo contrato con condiciones distintas y el mismo hash. Si tú firmas el primero, la firma vale igual para el segundo.

Y la preimagen es la que la gente da por rota cuando no lo está. El mejor ataque de preimagen conocido contra MD5 sigue costando del orden de 2¹²³ operaciones: inviable con cualquier hardware imaginable. De ahí sale una frase que conviene tener clara:

> Un hash MD5 de una contraseña **no se descifra: se adivina**. Nadie invierte el hash; lo que se hace es probar candidatos a mil millones por segundo hasta que uno coincide. Que funcione es culpa de la contraseña y de la velocidad del algoritmo, no de las colisiones.

Cómo se defiende eso es el tema de [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md).

**El veredicto operativo, aun así, es simple**: en código nuevo, ni MD5 ni SHA-1, ni siquiera para el caso "aquí no hay adversario". No porque el ataque aplique hoy, sino porque el código dura más que las suposiciones: el checksum interno de caché de este trimestre es el mecanismo de verificación de descargas del año que viene, y nadie va a releer el comentario que decía "esto es solo interno".

## SHA-2: la opción por defecto

Familia publicada por el NIST en 2001. Sus miembros son **SHA-224, SHA-256, SHA-384, SHA-512** y **SHA-512/256**, y el número es simplemente **el tamaño de la salida en bits**: SHA-256 devuelve 256 bits, o 32 bytes, o 64 caracteres en hexadecimal.

```bash
sha256sum instalador.exe
```

```
9b7c4e1a8f03d5b2c6e94a71f8d02b3e5c17a94d6f8b20e3c95a7d14b8e60f2a  instalador.exe
```

Son 64 caracteres hexadecimales y dos espacios antes del nombre del fichero — ese formato exacto importa, porque es lo que espera `sha256sum -c` al verificar. Los equivalentes para los otros miembros son `sha512sum`, `sha384sum` y compañía.

Es la opción por defecto por tres razones que no tienen nada que ver con ser el más elegante:

- **Soporte universal.** Está en la librería estándar de todos los lenguajes, en todos los navegadores y en todos los HSM. No hay ningún sistema al que tengas que integrarte que no lo entienda.
- **Aceleración en hardware.** Los procesadores modernos traen instrucciones dedicadas (**SHA-NI** en x86, extensiones de criptografía en ARM) que hacen SHA-256 varias veces más rápido sin cambiar una línea de código.
- **Normativa.** Cuando un auditor, un pliego público o un requisito PCI-DSS te pide un algoritmo aprobado, SHA-2 está en la lista. Convencer a un auditor de que BLAKE3 es seguro es una conversación que no quieres tener.

**Cuándo no usarlo:** cuando el volumen de datos es el cuello de botella real y mides que no llegas, o cuando necesitas un MAC y no quieres montar HMAC (ambas cosas más abajo). SHA-2 también es la única de las tres familias vulnerable al **ataque de extensión de longitud**, que no es un problema si la usas para lo que toca, pero sí lo es si te la juegas con un MAC casero; el mecanismo lo explica [Hash criptográfico](Hash-Criptografico.md).

## SHA-3: la reserva estratégica

Ganador de un concurso público del NIST y estandarizado en 2015 sobre el algoritmo **Keccak**. Su interior es **completamente distinto** al de SHA-2: usa una construcción llamada *sponge* (esponja), que "absorbe" el mensaje en un estado interno grande y luego lo "escurre" para producir la salida, en lugar de la construcción de Merkle-Damgård que comparten MD5, SHA-1 y SHA-2.

Ese detalle interno es exactamente **la razón de que exista**. SHA-3 no llegó porque SHA-2 estuviese roto —no lo está— sino como **diversificación**: si algún día apareciese un ataque contra la construcción de Merkle-Damgård, se llevaría por delante a MD5, SHA-1 y SHA-2 de golpe, y SHA-3 seguiría en pie. Es un seguro de vida, y de paso resuelve gratis la extensión de longitud, que es un defecto de esa construcción y no de SHA-2 en particular.

Trae además dos primitivas llamadas **SHAKE128** y **SHAKE256**, que son *extendable-output functions*: producen **una salida de la longitud que pidas**.

```bash
openssl dgst -shake256 -xoflen 64 instalador.exe
# SHAKE-256(instalador.exe)= 4f1a8c...   ← 128 caracteres hex, es decir 64 bytes
```

Eso sirve cuando necesitas más bytes de los que un hash fijo te da: derivar varias claves de un mismo material, generar una máscara del tamaño exacto de un mensaje, o rellenar un campo de longitud concreta sin concatenar hashes a mano.

**Cuándo no usarlo:** por defecto. En la práctica se usa poco, y hay un motivo tangible además de la inercia: **en software sin aceleración específica suele ser más lento que SHA-2**, y no aporta seguridad adicional frente a los ataques que existen hoy. Elígelo si un requisito lo pide explícitamente, si quieres diversificar frente a SHA-2 a conciencia, o si necesitas SHAKE.

## BLAKE2 y BLAKE3: los rápidos

**BLAKE2** (2012) salió del mismo concurso que ganó Keccak, quedando finalista. Ofrece seguridad comparable siendo **más rápido que MD5**, y es el hash por defecto de librerías como libsodium y de herramientas como `argon2`. Sus dos variantes principales son BLAKE2b (optimizada para 64 bits) y BLAKE2s (para 32 bits y sistemas empotrados).

**BLAKE3** (2020) va bastante más lejos con una idea concreta: en lugar de procesar el mensaje como una cadena de bloques donde cada uno depende del anterior, lo trocea y construye un **árbol de hashes** (*Merkle tree*). Los bloques del mismo nivel no dependen entre sí, así que se pueden hashear **en paralelo**: en varios hilos, o incluso dentro de un solo hilo aprovechando las instrucciones vectoriales (AVX2, AVX-512, NEON).

```bash
b3sum video-4gb.mp4
```

```
af1349b9f5f9a1a6a0404dea36dcc9499bcb25c9adc112b7cc9a93cae41f3262  video-4gb.mp4
```

La otra ventaja de la familia BLAKE, menos conocida y muy práctica, es el **modo con clave** nativo. Un hash normal no autentica nada, y para eso se usa HMAC, que es una construcción de dos pasadas sobre el hash. BLAKE2 y BLAKE3 aceptan una clave directamente:

```csharp
// Autenticar un TokenHash con BLAKE3 en modo keyed: una llamada, una pasada
var mac = Blake3.Hasher.NewKeyed(claveDeAplicacion);
mac.Update(Encoding.UTF8.GetBytes(tokenDeSesion));
byte[] etiqueta = mac.Finalize().AsSpan().ToArray();
```

El resultado es equivalente en propósito a `HMAC-SHA256(clave, token)` y sale más rápido, porque no hay dos pasadas. Con SHA-2 esto **no** se puede hacer a mano concatenando la clave: ahí está justo la trampa de la extensión de longitud.

**Cuándo no usarlos:** cuando te obliga la normativa (no están en las listas de algoritmos aprobados del NIST), cuando el consumidor del hash es un sistema ajeno que solo entiende SHA-2, o cuando tus entradas son muchas y minúsculas. El paralelismo de BLAKE3 solo luce con entradas grandes o varios hilos: para hashear millones de tokens de 32 bytes, la ventaja sobre BLAKE2 o sobre un SHA-256 acelerado por hardware se difumina hasta casi desaparecer.

## Cómo elegir: tabla de decisión

| Necesidad | Elección | Por qué |
|---|---|---|
| Compatibilidad máxima, "no quiero pensarlo" | **SHA-256** | Está en todas partes y acelerado en hardware moderno |
| Normativa, auditoría, pliego público | **SHA-256 o SHA-384** | Son los que aparecen en las listas aprobadas |
| Diversificación deliberada frente a SHA-2 | **SHA-3-256** | Construcción interna distinta: un ataque a Merkle-Damgård no le afecta |
| Volumen grande, el hash es el cuello de botella | **BLAKE3** | Árbol de hashes y paralelismo interno; varios GB/s por hilo |
| Volumen grande sin dependencias nuevas | **SHA-512 o SHA-512/256** | Palabras de 64 bits; suele batir a SHA-256 sin aceleración |
| Autenticar un dato (hace falta un secreto) | **HMAC-SHA256** o **BLAKE3 keyed** | Un hash a secas no autentica; ver más abajo |
| Salida de longitud arbitraria | **SHAKE256** | Es la única de la lista que la produce |
| Contraseñas de `Clientes.PasswordHash` | **Ninguno de estos** | Ver [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md) |
| Tokens de `Sesiones.TokenHash` | **SHA-256 basta** | Un token aleatorio ya tiene entropía; ver [Contraseñas vs tokens de sesión](Contrasenas-Vs-Tokens-De-Sesion.md) |

## El rendimiento, con el matiz que casi nadie sabe

El orden de magnitud realista, por hilo y en una CPU de servidor actual: **cientos de MB/s** para un hash sin aceleración, **algunos GB/s** para SHA-256 con SHA-NI o para BLAKE3 vectorizado. Es decir, hashear `video-4gb.mp4` está entre unos pocos segundos y una fracción de segundo, y en casi ningún sistema el hash es el cuello de botella: normalmente lo es el disco o la red.

Dicho eso, hay un detalle que sorprende. **En CPU de 64 bits, SHA-512 suele ser más rápido que SHA-256**, aunque produzca el doble de salida: trabaja con palabras de 64 bits y procesa 1024 bits por ronda en lugar de 512, así que hace menos rondas por megabyte. Existe incluso **SHA-512/256**, que es SHA-512 truncado a 32 bytes: te da la salida corta con el rendimiento de la variante larga (y, de propina, inmunidad a la extensión de longitud, porque el atacante no conoce el estado interno completo).

Pero el matiz del matiz: **las instrucciones SHA-NI aceleran SHA-256 en hardware y no SHA-512**, así que en un procesador que las tenga la comparación se invierte y SHA-256 gana con holgura. El veredicto depende de la máquina, de modo que no se decide leyendo, se decide midiendo — y **en la máquina de producción**, no en el portátil de desarrollo:

```bash
openssl speed -evp sha256
openssl speed -evp sha512
openssl speed -evp blake2b512
```

La salida relevante es la última columna, con bloques de 16 KB:

```
type             16 bytes     64 bytes    256 bytes   1024 bytes   8192 bytes  16384 bytes
sha256          151824.63k   448291.20k   977351.68k  1584301.06k  2137382.91k  2159542.27k
sha512           98442.11k   394517.33k   680127.66k   902341.63k   974252.03k   978321.41k
blake2b512      112904.55k   431228.16k   839467.09k  1042128.90k  1093458.77k  1096024.06k
```

Las cifras están en **k = 1000 bytes por segundo**, así que `2159542.27k` son unos 2,1 GB/s. En esta máquina SHA-256 gana claramente: tiene SHA-NI. En una sin ellas, la fila de `sha512` se pone por delante. Y una comprobación cruzada de un segundo, que además mide el disco real:

```bash
time sha256sum video-4gb.mp4
# real    0m2.184s
# user    0m1.905s
# sys     0m0.271s
```

Cuatro gigas en 2,2 segundos son unos 1,9 GB/s: coincide con lo que dijo `openssl speed`, luego el fichero venía de la caché de página y no del disco. Si el número real fuese mucho peor, el cuello de botella no es el algoritmo.

## Hashear en streaming: memoria constante

Toda librería seria ofrece una **API incremental** con tres operaciones: iniciar, ir alimentando el dato a trozos, y cerrar para obtener el hash. Se suele llamar *init / update / final*, y consume **memoria constante** sea el fichero de 4 KB o de 4 TB, porque solo mantiene el estado interno del algoritmo (unas decenas de bytes) más el búfer de lectura.

La alternativa es el error clásico: cargar el fichero entero en un array.

```csharp
// ❌ Carga los 4 GB en memoria antes de empezar a hashear
byte[] contenido = File.ReadAllBytes("video-4gb.mp4");
byte[] hash = SHA256.HashData(contenido);
```

Esto no es "ineficiente": es que **no funciona**. En .NET, `File.ReadAllBytes` está limitado por el tamaño máximo de un array y lanza una excepción antes de leer nada útil:

```
Unhandled exception. System.IO.IOException: The file is too long. This operation is
currently limited to supporting files less than 2 gigabytes in size.
   at System.IO.File.ReadAllBytes(String path)
```

Con un fichero de 1,5 GB no salta ese límite, y entonces es peor: el proceso sí lo intenta, reserva el bloque, y en un contenedor con memoria limitada muere con `OutOfMemoryException` o directamente lo mata el kernel por OOM. Es un fallo que solo aparece en producción, con el fichero grande, en la hora punta.

```csharp
// ✅ Lee el fichero a trozos: memoria constante, cualquier tamaño
await using var fichero = File.OpenRead("video-4gb.mp4");
byte[] hash = await SHA256.HashDataAsync(fichero);
string hex = Convert.ToHexString(hash).ToLowerInvariant();
```

`HashDataAsync` recibe el `Stream` y se encarga del bucle de lectura por dentro. Devuelve los 32 bytes del hash, y `Convert.ToHexString` los pasa a los mismos 64 caracteres que imprimió `sha256sum`. En la línea de comandos esto ya lo hacen bien todas las herramientas: `sha256sum` y `b3sum` leen a trozos, y por eso puedes hashear un fichero mayor que tu RAM. Los detalles de la API de .NET están en [Hashing en C#/.NET](Hashing-En-CSharp.md).

## Truncar el hash: la paradoja del cumpleaños con números

Por la **paradoja del cumpleaños**, la resistencia a colisiones de un hash es **la mitad de sus bits**: SHA-256 ofrece unos 128 bits contra colisiones, no 256. La razón intuitiva es que no buscas una coincidencia con un valor concreto, sino *cualquier* coincidencia entre todos los pares posibles, y los pares crecen con el cuadrado del número de elementos.

Esto casi nunca importa... hasta que alguien trunca. Es una tentación razonable: guardar los 32 bytes completos de un hash como clave de deduplicación cuesta espacio e índice, y "con los primeros 64 bits vamos sobrados". La regla es que con `n` elementos y una salida de `b` bits, la probabilidad de colisión ronda `n² / 2^(b+1)`. Aplicada a 64 bits:

| Escenario | Elementos | Probabilidad de al menos una colisión |
|---|---|---|
| Fotos de producto de la tienda | 10 millones | ~1 entre 370.000 |
| Fotos de producto, catálogo enorme | 100 millones | ~1 entre 3.700 |
| Un backup de 1 TB troceado en bloques de 4 KB | 268 millones | ~1 entre 500 |
| El mismo backup, 20 TB acumulados | 5.400 millones | **~50 %** |

El contraste es lo que hay que llevarse: si deduplicas fotos, 64 bits te dejan **lejísimos** del problema. Si troceas backups en bloques de 4 KB, estás **incómodamente cerca**, y el fallo no es un error visible: es un bloque que se sustituye por otro distinto y un backup que restaura basura, descubierto meses después. Con los 256 bits completos, la misma cuenta da un número tan pequeño que no tiene nombre útil.

Si vas a truncar, calcula el número. Y si el sistema es de deduplicación, **no truncues por debajo de 128 bits**.

## Los usos reales, cada uno con su ejemplo

**1. Integridad de descargas.** La tienda publica `instalador.exe` junto a un fichero con su hash. Quien descarga verifica:

```bash
sha256sum -c instalador.exe.sha256   # el fichero contiene "<hash>  instalador.exe"
```

```
instalador.exe: OK
```

Y si el fichero llegó corrupto o alterado:

```
instalador.exe: FAILED
sha256sum: WARNING: 1 computed checksum did NOT match
```

Ojo con lo que esto demuestra: solo que el fichero coincide con el hash **que publicaste tú**. Si el atacante controla la página, cambia los dos. Contra eso hace falta firma, no hash.

**2. `ETag` y caché HTTP.** El servidor usa el hash del cuerpo como identificador de versión; el cliente lo guarda y lo devuelve en la siguiente petición:

```
HTTP/1.1 200 OK
ETag: "9b7c4e1a8f03d5b2"        ← primera respuesta, con cuerpo

GET /api/catalogo HTTP/1.1
If-None-Match: "9b7c4e1a8f03d5b2"

HTTP/1.1 304 Not Modified       ← el cuerpo no se reenvía
```

Aquí no hay adversario: si el `ETag` colisionase, el peor caso es servir contenido cacheado de más — molesto, no peligroso. Es el escenario donde un hash truncado a 64 bits es perfectamente razonable.

**3. Direccionamiento por contenido y deduplicación.** El identificador de un objeto **es** su hash, así que dos objetos idénticos se guardan una sola vez y el nombre no hay que inventarlo. Es como funcionan Git, los registros de imágenes de contenedor (`sha256:...`) y cualquier almacén de blobs serio:

```bash
git hash-object plantilla-factura.html
# c8f1a4d90b7e3251af6c0d84b9e27fa3d10c5b6e
```

Vuelve a ejecutarlo tras cambiar un solo byte y el identificador es otro completamente distinto — el **efecto avalancha**, que se explica en [Hash criptográfico](Hash-Criptografico.md).

**4. Firmas digitales.** Lo que se firma **es el hash, no el documento**. Firmar con RSA es una operación matemática sobre un número del tamaño de la clave, así que un contrato de 12 MB no cabe: se hashea primero y se firma el resultado de 32 bytes.

```csharp
// El hash es el paso invisible dentro de SignData
using var rsa = RSA.Create(3072);
byte[] firma = rsa.SignData(
    contratoBytes,
    HashAlgorithmName.SHA256,          // ← esto es lo que se firma de verdad
    RSASignaturePadding.Pkcs1);
```

Y aquí se cierra el círculo con la sección de las colisiones: si ese parámetro dijese `SHA1`, un atacante con una colisión de prefijo elegido podría producir un segundo contrato con la misma firma válida. Por eso el algoritmo de hash de una firma no es un detalle de implementación.

## Cuándo NO usar estos algoritmos

**Para contraseñas.** Son rápidos **a propósito**, y esa velocidad es la del atacante. Con un `PasswordHash` calculado con SHA-256 a secas, una GPU prueba miles de millones de candidatos por segundo contra la base de datos robada. La defensa es un algoritmo deliberadamente lento y con coste en memoria: [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md).

**Para autenticar.** Un hash sin secreto solo detecta **corrupción accidental**. Cualquiera puede recalcularlo, así que un atacante que puede cambiar el dato también puede cambiar el hash y quedar todo coherente. Para autenticar hace falta un secreto (HMAC, o BLAKE3 en modo *keyed*) o una firma.

**Para anonimizar datos enumerables.** Este es el error con peores consecuencias legales, porque se comete de buena fe: "guardamos el teléfono hasheado, así ya no es un dato personal". No lo es solo si el espacio de entradas es grande, y no lo es:

```bash
# Precalcular el hash de los mil millones de móviles de 9 dígitos posibles
seq 600000000 799999999 | sha256sum --tag > tabla.txt   # 200 millones de candidatos
```

Doscientos millones de SHA-256 a los ~2 GB/s medidos antes son **unos pocos segundos** de CPU, y en una GPU es una fracción de segundo. Con un DNI (8 dígitos, porque la letra se deriva de ellos) son 100 millones de candidatos: lo mismo. Un correo electrónico no se enumera igual, pero existen listas públicas de miles de millones de direcciones filtradas, así que en la práctica se resuelve por consulta a una tabla. **Hashear un identificador con un espacio de valores pequeño no lo protege**; hace falta HMAC con una clave que el atacante no tenga, o un pseudónimo aleatorio guardado en una tabla aparte.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| El hash "no coincide" y ambos valores parecen correctos | Se compara hexadecimal (`9b7c4e1a...`, 64 caracteres) con base64 (`m3xOGo8D1bLG...`, 44). Son los mismos bytes en dos codificaciones |
| El hash del mismo fichero cambia según la máquina | Finales de línea: un fichero de texto pasado por Git con `core.autocrlf` tiene `\r\n` en Windows y `\n` en Linux. Distintos bytes, distinto hash. Hashea siempre en binario |
| Una verificación con adversario acepta un fichero manipulado | Se eligió MD5 "porque es más rápido". Con colisiones de prefijo elegido, la verificación no verifica nada |
| El regulador dice que la base de datos sigue conteniendo datos personales | Se hashearon correos o teléfonos creyendo que eso anonimiza. Espacio enumerable: no anonimiza |
| El proceso muere al procesar `video-4gb.mp4` | Se cargó el fichero completo con `File.ReadAllBytes` en lugar de hashear el `Stream` |
| Un tercero consigue "firmar" mensajes sin conocer el secreto | Se usó `SHA-256(secreto ‖ mensaje)` como firma. Es un ataque de **extensión de longitud**: hay que usar HMAC, o SHA-3/BLAKE, que son inmunes por construcción |

## Buenas prácticas avanzadas

- **La resistencia real a colisiones es la mitad de los bits.** Por la paradoja del cumpleaños, SHA-256 da ~128 bits contra colisiones, no 256. Importa sobre todo al truncar: 64 bits como clave de deduplicación es de sobra para 10 millones de fotos (~1 entre 370.000) e insensato para un almacén de bloques de 4 KB, que llega al 50 % de probabilidad hacia los 5.400 millones de bloques — unos 20 TB. Antes de truncar, haz la cuenta `n² / 2^(b+1)` con tu `n` real y con el `n` que tendrás en tres años.
- **SHA-2 sufre extensión de longitud; SHA-3 y BLAKE no.** Con SHA-256, conocer `hash(secreto ‖ dato)` y la longitud del secreto permite calcular el hash de mensajes extendidos **sin conocer el secreto**, así que jamás sirve como MAC casero. Para eso, HMAC. SHA-3 y BLAKE2/3 son inmunes por su construcción interna, y BLAKE2/3 traen además modo con clave nativo: sustituyen a HMAC con una sola llamada, una pasada y más velocidad. SHA-512/256 también es inmune, por el truncado.
- **En CPU de 64 bits SHA-512 suele ganar a SHA-256, salvo que haya SHA-NI.** Trabaja con palabras de 64 bits y procesa más datos por ronda; **SHA-512/256** te da ese rendimiento con salida de 32 bytes. Pero las instrucciones SHA-NI aceleran SHA-256 y no SHA-512, e invierten la tortilla. La conclusión no es memorizar un ganador: es correr `openssl speed -evp sha256` y `-evp sha512` **en la máquina de producción**, porque el veredicto es una propiedad del hardware, no del algoritmo.
- **Hashea en streaming, no cargando el fichero en memoria.** La API incremental (*init / update / final*) consume memoria constante; cargar el fichero entero es la diferencia entre hashear `video-4gb.mp4` y tumbar el proceso. Y un matiz sobre BLAKE3: su paralelismo interno solo luce con entradas grandes o varios hilos. Para millones de entradas pequeñas —hashear cada `TokenHash` en cada request— la ventaja sobre BLAKE2 o sobre un SHA-256 acelerado se difumina, y ahí el coste real está en el resto del código.
- **Guarda el algoritmo junto al hash, no en el código.** Un campo con `sha256:9b7c4e1a...` en lugar de `9b7c4e1a...` te permite cambiar de algoritmo dentro de cinco años y seguir validando lo antiguo, porque cada registro dice con qué se calculó. Es lo que hacen los registros de contenedores y lo que hace `$argon2id$` en un hash de contraseña. Sin ese prefijo, migrar de algoritmo obliga a invalidar todos los datos existentes de golpe, que es la razón por la que tanto sistema sigue con SHA-1 en 2026.
- **Separa dominios aunque uses el mismo algoritmo.** Si el mismo hash se usa para identificar ficheros y para indexar tokens, un valor calculado en un contexto puede colarse en el otro. La solución con SHA-2 es prefijar una etiqueta de propósito antes del dato (`"tienda/sesion/v1" ‖ token`); BLAKE2 y BLAKE3 lo traen de fábrica con sus parámetros de personalización y `derive_key`. Cuesta una constante y elimina toda una categoría de confusión entre contextos.

## Documentación oficial

- [FIPS 180-4](https://csrc.nist.gov/pubs/fips/180-4/upd1/final) — el estándar de SHA-2. La referencia que zanja discusiones sobre qué miembros tiene la familia, qué tamaños de salida están aprobados y qué es exactamente SHA-512/256.
- [FIPS 202](https://csrc.nist.gov/pubs/fips/202/final) — el estándar de SHA-3 y de las funciones SHAKE. Ve ahí cuando necesites una salida de longitud arbitraria y quieras saber qué garantiza.
- [BLAKE3 en GitHub](https://github.com/BLAKE3-team/BLAKE3) — el repositorio de referencia, con la especificación en PDF, las implementaciones y el diagrama del árbol de hashes. Es también donde están las comparativas de rendimiento oficiales.
- [Referencia de comandos de OpenSSL `dgst`](https://docs.openssl.org/master/man1/openssl-dgst/) — para cuando necesites calcular un digest concreto desde la terminal y no recuerdes el nombre exacto del algoritmo o el formato de salida.

## Recursos didácticos

- [shattered.io](https://shattered.io) — la página de la colisión de SHA-1 de 2017. Te deja **descargar los dos PDF** con el mismo hash y comprobarlo tú con `sha256sum` y `sha1sum` en la misma sesión. Es la forma más rápida de que "SHA-1 tiene colisiones" deje de ser una frase y se convierta en algo que has visto.
- [sha256algorithm.com](https://sha256algorithm.com) — visualizador interactivo de SHA-256: escribes un mensaje y ves las 64 rondas paso a paso, con el estado interno cambiando. Cambia un solo carácter y observa el efecto avalancha en directo.

---

*En resumen: SHA-256 como opción por defecto, BLAKE3 cuando la velocidad importa de verdad y lo has medido, SHA-3 como reserva estratégica — y MD5 y SHA-1 jubilados para cualquier uso con adversario, aunque su preimagen siga siendo inviable.*
