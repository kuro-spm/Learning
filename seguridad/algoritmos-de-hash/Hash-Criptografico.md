# Hash criptográfico

## ¿Qué es?

Un hash criptográfico es una función matemática que recibe datos de cualquier tamaño —un texto, un fichero, una contraseña— y devuelve siempre una "huella digital" de tamaño fijo y aparentemente aleatoria, llamada *digest* o, coloquialmente, *hash*.

## ¿Por qué existe?

Muchos problemas de seguridad e integridad se reducen a la misma pregunta: "¿estos datos son exactamente los que espero, sin haberlos guardado ni transmitido enteros?". Comparar dos ficheros de 4 GB byte a byte es caro; comparar sus huellas de 32 bytes es instantáneo. Y guardar una contraseña tal cual es un desastre esperando a ocurrir; guardar solo su huella permite verificarla sin conocerla.

> Piensa en el hash como la huella dactilar de una persona: ocupa poquísimo comparado con la persona entera, identifica de forma prácticamente única, y a partir de la huella no puedes reconstruir a la persona.

La analogía tiene un límite que conviene fijar desde ya: una huella dactilar la produce la naturaleza y no se elige, mientras que un digest lo produce un algoritmo público que **cualquiera** puede ejecutar. Esa diferencia es la raíz de casi todos los errores de esta ficha, porque significa que un hash por sí solo nunca prueba **quién** calculó el dato.

## ¿Cuándo y para qué se usa?

El ejemplo que recorre toda la ficha es una **tienda online con API en .NET**: su base de datos tiene una tabla `Clientes` con una columna `PasswordHash` y una tabla `Sesiones` con una columna `TokenHash`, y para integridad de ficheros publica un `instalador.exe` junto con su hash. Con eso en la cabeza, los cuatro usos habituales:

- **Verificar integridad**: la web publica el hash de `instalador.exe`; quien lo descarga lo recalcula y comprueba que coincide.
- **Guardar secretos verificables**: la tabla `Clientes` no guarda la contraseña, guarda un derivado de ella (ver [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md)); la tabla `Sesiones` guarda `TokenHash` en lugar del token, para que un volcado de la base de datos no permita suplantar sesiones activas.
- **Identificar y deduplicar**: dos clientes suben la misma imagen de perfil y el hash revela que es el mismo fichero, así que se almacena una vez.
- **Firmar sin firmar todo**: una firma digital no se aplica al documento entero, se aplica a su hash. Git hace lo mismo: cada commit se identifica por el hash de su contenido.

## Anatomía de un digest: el mismo hash escrito de tres formas

Un digest **es una secuencia de bytes**. El texto que ves habitualmente es solo una forma de escribirlo, y confundir la representación con el valor causa la mitad de los problemas al comparar hashes entre sistemas. Partimos del digest de la cadena `hola`, escrito de dos maneras:

```bash
printf 'hola' | sha256sum
# b221d9dbb083a7f33428d7c2a3c3198ae925614d70210e28716ccaa7cd4ddb79  -

printf 'hola' | openssl dgst -sha256 -binary | base64
# siHZ27CDp/M0KNfCo8MZiuklYU1wIQ4ocWzKp81N23k=
```

`sha256sum` imprime hexadecimal; `-binary | base64` imprime los mismos bytes en base64. Las tres representaciones lado a lado:

| Representación | Valor | Longitud |
|---|---|---|
| Bytes crudos | `178, 33, 217, 219, 176, 131, 167, 243, …` | **32 bytes** |
| Hexadecimal | `b221d9db…cd4ddb79` | **64 caracteres** (2 por byte) |
| Base64 | `siHZ27CDp/M0KNfCo8MZiuklYU1wIQ4ocWzKp81N23k=` | **44 caracteres** (con relleno `=`) |

Los tres son el mismo hash. En C# la conversión es directa, y hay un detalle que muerde:

```csharp
byte[] digest = SHA256.HashData(Encoding.UTF8.GetBytes("hola"));

Convert.ToHexString(digest);                     // "B221D9DB...CD4DDB79"  ← MAYÚSCULAS
Convert.ToHexString(digest).ToLowerInvariant();   // "b221d9db...cd4ddb79"  ← como sha256sum
Convert.ToBase64String(digest);                   // "siHZ27CDp/M0KNfCo8MZiuklYU1wIQ4ocWzKp81N23k="
```

`Convert.ToHexString` devuelve mayúsculas y las herramientas de línea de comandos escriben minúsculas: un `hashGuardado == hashCalculado` entre ambos mundos falla siempre, con dos valores que a ojo parecen idénticos. La API completa de .NET se trata en [Hashing en C#/.NET](Hashing-En-CSharp.md). Y esto decide también el tipo de columna: para `Sesiones.TokenHash` con SHA-256, `binary(32)` si guardas bytes, `char(64)` si guardas hexadecimal, `char(44)` si guardas base64. Elige una, escríbela en el esquema y no dejes que cada servicio decida por su cuenta.

## Las cinco propiedades, una por una

Estas son las cinco propiedades que convierten una función de resumen cualquiera en un hash *criptográfico*.

### 1. Determinista

La misma entrada produce siempre la misma salida: en cualquier lenguaje, en cualquier máquina y dentro de diez años. Sin esto no podrías comparar huellas en absoluto. Ejecuta esto mil veces, en Linux, en Windows y en el móvil, y el resultado no cambia:

```bash
printf 'hola' | sha256sum
# b221d9dbb083a7f33428d7c2a3c3198ae925614d70210e28716ccaa7cd4ddb79  -
# Ojo: echo sin -n añade un salto de línea, así que hashea otra cosa. Por eso aquí se usa printf.
```

### 2. Unidireccional: dos resistencias distintas

Del hash no se puede volver a la entrada. Pero "no se puede volver" son en realidad **dos** garantías separadas, y la ficha las distingue porque los ataques que las rompen son distintos:

- **Resistencia a preimagen**: dado un digest `d`, es inviable encontrar *cualquier* entrada `m` tal que `hash(m) = d`. Es la que protege `PasswordHash`: quien roba la base de datos tiene el digest y quiere una contraseña que lo produzca.
- **Resistencia a segunda preimagen**: dada una entrada concreta `m1`, es inviable encontrar una `m2` distinta con `hash(m2) = hash(m1)`. Es la que protege `instalador.exe`: el atacante no busca cualquier fichero, busca **otro instalador con el hash del tuyo**, uno que además sea un ejecutable válido y malicioso.

Lo que las distingue es **quién elige el mensaje**: en la preimagen no controlas nada, en la segunda preimagen el original está dado y tienes que acercarte a él, y en la resistencia a colisiones —la propiedad 4— el atacante elige **los dos**, que es muchísimo más fácil. La única vía conocida contra las dos primeras es probar entradas una a una. Con 256 bits eso es inviable... **salvo que la entrada tenga poca entropía**, y ahí el hash no te salva: es el motivo de que las contraseñas necesiten tratamiento aparte, que desarrolla [Contraseñas vs tokens de sesión](Contrasenas-Vs-Tokens-De-Sesion.md).

### 3. Efecto avalancha

Cambiar **un solo bit** de la entrada cambia, de media, la mitad de los bits de la salida. Cambiemos una letra minúscula por mayúscula, que en ASCII es un único bit de diferencia:

```bash
printf 'hola' | sha256sum
# b221d9dbb083a7f33428d7c2a3c3198ae925614d70210e28716ccaa7cd4ddb79
printf 'holA' | sha256sum
# e028fa3463f8bbcb76607c0819bd85103de648004fcabb0648ce3662459956f4
```

A ojo son irreconocibles, pero la demostración está en binario. Estos son los **cuatro primeros bytes** de cada digest y su XOR, que marca con un `1` cada bit que cambió:

```
hola →  10110010 00100001 11011001 11011011
holA →  11100000 00101000 11111010 00110100
XOR     01010010 00001001 00100011 11101111
        └─ 3 ──┘ └─ 2 ──┘ └─ 3 ──┘ └─ 7 ──┘   = 15 bits de 32
```

Contando los 256 bits completos, **131 han cambiado**: exactamente lo que esperarías de dos valores independientes (la mitad, 128, con algo de ruido estadístico). Un bit de entrada ha reventado medio digest. Eso tiene una consecuencia útil —el digest no filtra "cuánto se parecen" dos datos, así que publicar el hash de `instalador.exe` no revela nada de su contenido— y una limitación: **no sirve para buscar cosas parecidas**. Para detectar imágenes similares o ficheros casi iguales hacen falta un *hash perceptual* o un *fuzzy hash*, que son otra familia y no dan garantías de seguridad.

### 4. Resistencia a colisiones

Debe ser inviable encontrar **dos entradas cualesquiera** con el mismo hash. Aquí el atacante tiene libertad total para fabricarlas a medida, y por eso es la propiedad que cae primero. Las colisiones **existen** necesariamente —hay infinitas entradas y solo 2²⁵⁶ salidas—; la garantía no es que no haya, es que encontrar una cueste más de lo que nadie puede pagar. Cuando alguien aprende a fabricarlas, el algoritmo queda muerto para uso de seguridad. Le pasó a MD5 (hoy se generan colisiones en segundos en un portátil) y a SHA-1 en 2017, cuando el proyecto SHAttered publicó **dos PDF distintos con el mismo SHA-1**: `38762cf7f55934b34d179ae6a4c80cadccbb7f0a`. Aplicado a nuestro caso: dos `instalador.exe`, uno legítimo y uno con puerta trasera, ambos coincidiendo con el hash publicado en la web. El detalle de qué algoritmos están vivos y cuáles jubilados está en [Hashes de propósito general](Hashes-De-Proposito-General.md).

### 5. El tamaño de salida es fijo

SHA-256 produce 256 bits (32 bytes) tanto si la entrada es una letra como si es una película de 4 GB, y se comprueba midiendo la salida:

```bash
printf 'a' | sha256sum | cut -c1-64 | wc -c    # 65 (64 caracteres + el salto de línea)
sha256sum instalador.exe | cut -c1-64 | wc -c  # 65 — exactamente el mismo tamaño
```

Eso es lo que permite reservar una columna de tamaño conocido en `Clientes.PasswordHash` y comparar dos ficheros enormes en microsegundos. Más bits significa más resistencia, pero **no en la proporción que uno supondría**: eso es lo siguiente.

## La paradoja del cumpleaños: por qué la resistencia es de n/2 bits

Un hash de n bits tiene 2ⁿ salidas posibles, así que la intuición dice que hacen falta del orden de 2ⁿ intentos para tropezar con una colisión. Es falso, y por un margen enorme.

El nombre viene del problema clásico: con 365 días posibles de cumpleaños bastan **23 personas** para que sea más probable que no que dos coincidan, porque no comparas cada persona con una fecha fija sino **todos los pares** (23 personas son 253 pares). Con hashes ocurre igual: al acumular k elementos generas k²/2 pares, y las colisiones se vuelven esperables hacia **√(2ⁿ) = 2^(n/2)** elementos. Un hash de n bits ofrece n bits contra preimagen pero solo **n/2 contra colisiones**.

Y ahora los números, donde deja de ser abstracto:

| Bits conservados | Caracteres hex | Colisión esperable hacia | ¿Realista? |
|---|---|---|---|
| 32 | 8 | **65.536** elementos | Sí — una tabla mediana |
| 64 | 16 | **4.300 millones** | Sí a escala de logs o eventos |
| 96 | 24 | 2,8 · 10¹⁴ | No en un sistema normal |
| 128 (medio SHA-256) | 32 | 1,8 · 10¹⁹ | No |
| 256 (SHA-256 completo) | 64 | 3,4 · 10³⁸ | Jamás, a ninguna escala |

La lectura práctica: si la tienda genera un identificador corto público truncando un SHA-256 a 8 caracteres hexadecimales, a los **65.000 pedidos** empieza a ser esperable que dos pedidos distintos compartan identificador. No porque SHA-256 sea débil, sino porque has tirado 224 de sus 256 bits. Si truncas, calcula primero cuántos elementos llegarás a tener y deja varios órdenes de magnitud de margen.

## Hash, cifrado, MAC y firma: cuál necesitas

Este es el error más común del tema: usar un hash donde hacía falta un MAC. La tabla lo resuelve, pero primero el vocabulario. **Integridad** es poder detectar si el dato cambió; **confidencialidad**, que nadie más pueda leerlo; **autenticidad**, poder comprobar que viene de quien dice venir; **no repudio**, que quien lo envió no pueda negarlo después ante un tercero. Y un **MAC** (*Message Authentication Code*) es un "hash con secreto"; el más usado es **HMAC**.

| Herramienta | ¿Reversible? | ¿Clave? | Integridad | Confidencialidad | Autenticidad | No repudio |
|---|---|---|---|---|---|---|
| Hash (SHA-256) | No | **No** | Sí | No | **No** | No |
| Cifrado autenticado (AES-GCM) | Sí, con la clave | Simétrica | Sí | **Sí** | Sí | No |
| MAC (HMAC-SHA256) | No | Simétrica | Sí | No | **Sí** | No |
| Firma digital (Ed25519, RSA) | No | Asimétrica | Sí | No | Sí | **Sí** |

Aplicado a la tienda, la decisión sale sola:

- Publicar el hash de `instalador.exe` en la web → **hash**. Funciona porque el digest viaja por un canal que ya es de confianza (HTTPS de tu dominio) y el fichero por otro (un CDN).
- Guardar los datos de facturación de un cliente para poder mostrarlos luego → **cifrado**. El dato hay que recuperarlo.
- Firmar un webhook que la tienda envía a un proveedor logístico → **MAC**. Ambos comparten un secreto y solo hace falta que el receptor sepa que el mensaje es tuyo.
- Emitir un recibo que un tercero pueda verificar sin conocer tus secretos → **firma digital**. Solo la firma da no repudio, porque la clave privada la tienes tú y solo tú.

La casilla clave es la de autenticidad del hash: **No**. Un hash sin secreto no autentica nada, porque el algoritmo es público y cualquiera puede recalcularlo. Si alguien altera el fichero *y* el digest publicado, ambos cuadran. Por eso hay que resistirse a la tentación de fabricarse un MAC casero, que es lo siguiente.

## El ataque de extensión de longitud: por qué `SHA-256(secreto ‖ mensaje)` no es un MAC

Ya sabemos que un hash no autentica. La reacción natural es mezclarlo con un secreto: si el atacante no lo conoce, no puede recalcular el hash. La idea es buena; **la construcción es rota**. SHA-256 usa el esquema **Merkle–Damgård**: parte de un estado interno fijo, corta el mensaje en bloques de 64 bytes y por cada bloque actualiza ese estado. Cuando se acaban los bloques, **publica el estado interno tal cual: ese es el digest**.

Ahí está el agujero. El digest no es un resumen del mensaje, es *el punto exacto en el que el algoritmo se quedó*. Quien lo tiene puede cargarlo como estado inicial y seguir procesando bloques nuevos, igual que habría hecho la función original: no necesita el secreto, porque el secreto ya está "digerido" dentro del estado. Solo necesita saber cuántos bytes mide, para reconstruir el **relleno** (*padding*), que SHA-256 escribe siempre igual —un byte `0x80`, ceros y la longitud total en bits— y por tanto es deducible.

**El escenario.** La tienda genera enlaces de descarga firmados "a mano" con `firma = SHA256(secreto ‖ "cliente=4711")`, donde el secreto son 16 bytes que solo conoce el servidor:

```bash
printf 's3cr3t-de-firma!cliente=4711' | sha256sum
# a2d038ea6227b3b6106cc528b899a9d693b4f9aca5acb670fedfdea5e050f744
# → https://tienda.example/descargas/instalador.exe?cliente=4711&firma=a2d038ea…e050f744
```

El atacante recibe ese enlace legítimamente: conoce `cliente=4711` y la firma, no conoce el secreto. Quiere añadir `&rol=admin`. Con [hash_extender](https://github.com/iagox86/hash_extender):

```bash
./hash_extender --data 'cliente=4711' --secret 16 \
                --append '&rol=admin' --format sha256 \
                --signature a2d038ea6227b3b6106cc528b899a9d693b4f9aca5acb670fedfdea5e050f744
```

Y sale una firma válida para un mensaje que el servidor nunca emitió:

```
New signature: 925baeaa925eb652bde287c5a2198c210f5a439e5de64a7248437b0677e385a7
New string:    cliente%3d4711%80%00%00…%00%e0%26rol%3dadmin
```

Compruébalo: el mensaje forjado es el original, más el relleno (un `0x80`, 34 ceros y `0xe0` — los 224 bits de los 28 bytes iniciales), más `&rol=admin`. Su hash con el secreto real coincide:

```bash
{ printf 's3cr3t-de-firma!cliente=4711\x80'   # secreto + mensaje + inicio del relleno
  printf '\x00%.0s' {1..34}                    # los ceros del relleno
  printf '\xe0&rol=admin'; } | sha256sum       # longitud (224 bits) + la carga añadida
# 925baeaa925eb652bde287c5a2198c210f5a439e5de64a7248437b0677e385a7  ← la misma firma forjada
```

El servidor validará ese enlace: el atacante ha firmado en tu nombre sin saber tu secreto, probando solo unas cuantas longitudes hasta acertar.

**La solución es HMAC.** No es `hash(secreto ‖ mensaje)` ni `hash(mensaje ‖ secreto)`: es una construcción anidada, `hash(clave_externa ‖ hash(clave_interna ‖ mensaje))`, que aplica el hash dos veces y por eso el digest final no expone un estado interno reutilizable. En .NET es `HMACSHA256.HashData(clave, mensaje)`. Esta colección no desarrolla HMAC —está en su lista de siguientes pasos, en el [README](README.md)— pero la regla cabe en dos líneas:

```csharp
// ❌ Firma casera: vulnerable a extensión de longitud
var firma = SHA256.HashData(Encoding.UTF8.GetBytes(secreto + mensaje));

// ✅ Con secreto de por medio, siempre HMAC
var firma = HMACSHA256.HashData(claveBytes, Encoding.UTF8.GetBytes(mensaje));
```

**Si hay un secreto en la entrada, no es un hash lo que necesitas: es un HMAC.** Sin excepciones.

## Comparar hashes de secretos en tiempo constante

Al validar una sesión, la tienda calcula el hash del token recibido y lo compara con `Sesiones.TokenHash`. Una comparación normal **devuelve en cuanto encuentra el primer byte distinto**, y ese microtiempo es información:

```csharp
// ❌ Sale antes con un candidato que falla en el byte 1 que con uno que falla en el byte 20
if (hashCalculado.SequenceEqual(hashGuardado)) { /* sesión válida */ }

// ✅ Recorre siempre los 32 bytes, tarde lo que tarde
if (CryptographicOperations.FixedTimeEquals(hashCalculado, hashGuardado)) { /* sesión válida */ }
```

Un atacante que pueda medir el tiempo de respuesta afina byte a byte: prueba los 256 valores del primero, se queda con el que tarda un pelo más y pasa al siguiente. En lugar de 2²⁵⁶ intentos, son 256 × 32. La diferencia son nanosegundos, pero se promedia con muchas peticiones repetidas.

`FixedTimeEquals` recorre los dos arrays completos con operaciones sin ramificación: el tiempo depende solo de la longitud, nunca del contenido. Úsalo **siempre** que uno de los dos lados sea secreto: tokens, HMACs, códigos de un solo uso. Para el hash público de `instalador.exe`, un `==` normal está bien: no hay nada que filtrar.

## Canonicalización: no hasheas datos, hasheas bytes

"Hashear el mismo dato" no significa nada hasta que se fijan **los bytes exactos**: un mismo dato lógico tiene decenas de representaciones válidas y cada una produce un digest completamente distinto. El caso más frecuente es que dos servicios de la tienda hasheen "el mismo" pedido serializado con las claves en otro orden:

```bash
printf '{"pedido":4711,"importe":89.90}' | sha256sum
# 9013e1793608917bd50e762df534668acb21da5e3eb053fa03e26a1267c8043a
printf '{"importe":89.90,"pedido":4711}' | sha256sum
# 40400e5b7749305cc13c6b0bec6d117b126d801af386742101fb4b35d91b38d3
```

Mismo pedido, mismo importe, digests sin nada en común. Y el orden de las claves de un JSON no está garantizado: depende del serializador, de su versión y a veces del orden de los campos en la clase de C#. El segundo caso son los finales de línea y el **BOM** (*Byte Order Mark*, tres bytes invisibles `EF BB BF` que algunos editores de Windows meten al principio de un fichero UTF-8):

```bash
printf 'linea1\nlinea2\n'     | sha256sum   # LF (Linux, macOS)
# 8a8f4b9135d02a838d28255ffd49161fa434b4f0f82a2045104748ef56d7fd98
printf 'linea1\r\nlinea2\r\n' | sha256sum   # CRLF (Windows)
# 8feab959bcad6ea7f12d01bf426542b40599df9748da7a299be284610e93a899
# el mismo LF de la primera línea, pero con BOM delante:
# 9c2700a1705ff9eca9114cd3a63914e259d5dac8a94bf93da822c2088463b4d6
```

Tres digests para un contenido que nadie ha editado: solo ha pasado por un `git` con `core.autocrlf` activado o por el Notepad. La solución es tratar la serialización como **parte del contrato**, igual que los nombres de los campos: codificación fija (UTF-8 sin BOM), orden de campos fijo (alfabético o el que sea, pero escrito), formato numérico fijo (`89.90` y `89.9` son bytes distintos), finales de línea fijos, y nada de espacios de adorno. Para JSON existe la especificación **JCS** (RFC 8785, *JSON Canonicalization Scheme*) precisamente para esto. Si no puedes usarla, define tu propia cadena canónica y **no hashees nunca la salida de un serializador genérico**.

## Concatenar campos sin delimitar fabrica colisiones sin romper el algoritmo

Un caso de canonicalización tan concreto que merece su propio apartado. Si construyes la entrada del hash juntando campos, mira esto:

```bash
printf 'ana100' | sha256sum   # cliente="ana",  importe="100"
# 7df1247d52f33d31e7e286ffde5e08ac49296bc39eb7ad3373c9451a3af4bd81
printf 'ana100' | sha256sum   # cliente="ana1", importe="00"
# 7df1247d52f33d31e7e286ffde5e08ac49296bc39eb7ad3373c9451a3af4bd81
```

Es el mismo comando dos veces, y esa es la gracia: `hash("ana" + "100")` y `hash("ana1" + "00")` son **el mismo hash** porque los bytes que entran son los mismos. La frontera entre campos no existe para la función.

El escenario de explotación: la tienda firma sus reembolsos con `cliente ‖ importe`. Un atacante registra el usuario `ana1`, pide un reembolso de `00` euros y obtiene una firma válida... que es también la firma válida del reembolso de **100 euros** a `ana`. No ha roto SHA-256: ha desplazado la frontera. Prefijar cada campo con su longitud es la solución que no falla nunca:

```csharp
// ❌ La frontera entre campos es ambigua
var entrada = cliente + importe;                       // "ana" + "100" → "ana100"

// ✅ La longitud hace que solo haya una lectura posible
var entrada = $"{cliente.Length}:{cliente}|{importe.Length}:{importe}";
// "ana" + "100"  → "3:ana|3:100"
// "ana1" + "00"  → "4:ana1|2:00"     ← ahora sí son distintos
```

Un separador que sea imposible dentro de los datos también sirve, pero "imposible" es una afirmación fuerte: el día que alguien registre un nombre con ese carácter, la garantía desaparece en silencio. La longitud no depende de suposiciones sobre los datos.

## Cuándo NO usar un hash

**1. Cuando el dato hay que recuperarlo.** El hash es irreversible por diseño: si dentro de dos meses vas a necesitar el teléfono del cliente para llamarle, no lo hashees, cífralo. Un hash de un dato que necesitabas es un dato perdido.

**2. Cuando el espacio de entradas es pequeño y enumerable.** Este es el error con peores consecuencias, porque parece cumplimiento normativo y no lo es: hashear un teléfono, un DNI o un correo electrónico **no lo anonimiza**. Un teléfono español son 9 dígitos, y una GPU de gama alta calcula del orden de 10.000 millones de SHA-256 por segundo:

```
1.000.000.000 candidatos ÷ 10.000.000.000 hashes/s ≈ 0,1 segundos
```

Una décima de segundo para construir la tabla completa que revierte **cualquier** teléfono hasheado del planeta. Con un DNI (8 dígitos más una letra que se deriva de ellos) son 10⁸: aún más rápido. Con correos electrónicos no hay enumeración exhaustiva, pero existen listas públicas de miles de millones de direcciones filtradas y probarlas cuesta minutos. La unidireccionalidad **no protege entradas predecibles**, solo entradas con suficiente entropía; para seudonimizar identificadores personales la vía es cifrado con clave gestionada, o un HMAC cuya clave no se filtre junto a los datos.

**3. Cuando hace falta autenticidad y no solo integridad.** Si un atacante puede alterar el dato *y* el digest, el hash no aporta nada. Es la fila "autenticidad: No" de la tabla, y lleva a HMAC o a firma digital.

**4. Cuando lo que hasheas es una contraseña.** SHA-256 está diseñado para ser **rápido**, y eso es exactamente lo contrario de lo que quieres en `Clientes.PasswordHash`: velocidad es lo que un atacante necesita para probar millones de candidatos. Ahí van funciones deliberadamente lentas y con sal, que cubre [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md); la sal y sus confusiones habituales, [Sal vs seed](Sal-Vs-Seed.md).

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| Dos servicios calculan hashes distintos del "mismo" pedido | Falta canonicalización: orden de claves del JSON, formato numérico o codificación distintos |
| El hash de un fichero de texto cambia al pasarlo por un editor | Se añadió un BOM o se convirtieron los finales de línea de LF a CRLF |
| La comparación falla con dos hashes que a ojo son idénticos | Se compara hexadecimal con base64, o mayúsculas de `Convert.ToHexString` con minúsculas de `sha256sum` |
| Tras una auditoría de privacidad, un teléfono "anonimizado" aparece revertido | Se hasheó un dato de espacio enumerable: 10⁹ candidatos se agotan en menos de un segundo |
| El volcado de `Clientes` se convierte en una lista de contraseñas en horas | Se usó un hash rápido (SHA-256) en lugar de una función de derivación lenta con sal |
| Un enlace firmado acepta parámetros que el servidor nunca emitió | La firma es `hash(secreto ‖ mensaje)`: extensión de longitud. Hacía falta HMAC |
| Dos pedidos distintos comparten identificador corto a los pocos meses | Digest truncado a 32 bits: colisiones esperables hacia los 65.000 elementos |
| La validación de sesión responde antes con unos tokens que con otros | Comparación con `==` en lugar de `FixedTimeEquals`: fuga por *timing* |

## Buenas prácticas avanzadas

- **Hashea bytes, no conceptos: canonicaliza antes.** "El mismo dato" tiene mil representaciones —un JSON con las claves en otro orden, UTF-8 frente a UTF-16, CRLF frente a LF, un BOM invisible— y cada una da un digest distinto. Define los bytes exactos (codificación, orden de campos, formato numérico, separadores) **como parte del contrato entre servicios**, versiónalo, y prueba la canonicalización con un test que fije el digest esperado de un caso conocido. Ese test es lo que detecta el día que alguien cambia el serializador.
- **Concatenar campos sin delimitar crea colisiones triviales.** `hash("ana" + "100")` y `hash("ana1" + "00")` son idénticos, y quien pueda controlar dos campos adyacentes puede desplazar la frontera entre ellos para fabricar dos mensajes distintos con el mismo hash **sin tocar el algoritmo**. Prefija cada campo con su longitud. Un separador "imposible en los datos" funciona hasta que deja de serlo, y falla en silencio.
- **`SHA-256(secreto ‖ mensaje)` no es un MAC: ataque de extensión de longitud.** La construcción Merkle–Damgård publica su estado interno como digest, así que quien conoce `hash(secreto ‖ mensaje)` puede calcular `hash(secreto ‖ mensaje ‖ relleno ‖ extra)` **sin conocer el secreto**, probando unas pocas longitudes hasta acertar. Es el error clásico al firmar URLs o webhooks a mano. Regla mecánica: **si hay un secreto entre los bytes de entrada, usa HMAC**. SHA-3 y BLAKE2 no tienen esta debilidad, pero no confíes en recordar cuál usas.
- **Compara hashes de secretos en tiempo constante.** Un `==` devuelve en cuanto encuentra el primer byte distinto, y ese microtiempo permite reconstruir el valor correcto byte a byte: 256 × 32 intentos en lugar de 2²⁵⁶. Al verificar HMACs o `Sesiones.TokenHash`, usa `CryptographicOperations.FixedTimeEquals`. Aplícalo también a la respuesta completa: si el endpoint tarda distinto según *dónde* falle la validación, la fuga sigue ahí aunque la comparación sea constante.
- **Truncar un hash cuesta el doble de lo que parece.** La resistencia a colisiones es de n/2 bits por la paradoja del cumpleaños: recortar SHA-256 a 8 caracteres hexadecimales deja 32 bits, y las colisiones son esperables a partir de **~65.000 elementos**. Antes de derivar identificadores cortos de un digest, calcula cuántos elementos tendrás en el peor caso y deja tres órdenes de magnitud de margen. Y si el identificador es público, ten claro que estás publicando información sobre la entrada.

## Documentación oficial

- [FIPS 180-4](https://csrc.nist.gov/pubs/fips/180-4/upd1/final) — el estándar de la familia SHA-2. No se lee de principio a fin: se consulta para zanjar dudas sobre tamaños de salida, variantes aprobadas y el relleno del mensaje, que es la pieza que hace posible el ataque de extensión de longitud.
- [RFC 2104 — HMAC](https://www.rfc-editor.org/rfc/rfc2104) — la especificación del mecanismo que sustituye a `hash(secreto ‖ mensaje)`. Su sección 2 explica en pocas líneas por qué la construcción con dos pasadas cierra el agujero.
- [`System.Security.Cryptography` en la documentación de .NET](https://learn.microsoft.com/dotnet/api/system.security.cryptography) — la referencia de la API cuando necesites el detalle de un método. Los dos tipos que conviene mirar primero son `SHA256` y `CryptographicOperations`, que es donde vive la comparación en tiempo constante.

## Recursos didácticos

- [sha256algorithm.com](https://sha256algorithm.com/) — SHA-256 ejecutándose paso a paso y en animación, con el estado interno visible bloque a bloque. Es la forma más rápida de *ver* por qué el digest es el estado final del algoritmo, que es justo la pieza que explica la extensión de longitud.
- [SHAttered](https://shattered.io/) — el sitio de la primera colisión real de SHA-1, con los **dos PDF distintos y el mismo digest** descargables para comprobarlo con `sha1sum`. Traduce "resistencia a colisiones" de concepto abstracto a dos ficheros en tu carpeta de descargas.
- [corkami/collisions](https://github.com/corkami/collisions) — colisiones de MD5 ya calculadas: pares de imágenes JPG, PNG, PDF y ejecutables con el mismo hash y contenido totalmente distinto. Incluye *hashquines*, ficheros que muestran su propio MD5 dentro de sí mismos. Divertidísimo y demoledor.
- [hash_extender](https://github.com/iagox86/hash_extender) — la herramienta del ejemplo de esta ficha. Monta un endpoint de prueba que valide `sha256(secreto ‖ mensaje)`, rómpelo con esto y cámbialo a HMAC para verlo dejar de funcionar: media hora que fija el concepto para siempre.
- [CyberChef](https://gchq.github.io/CyberChef/) — navaja suiza en el navegador para convertir entre hexadecimal, base64 y bytes, y calcular digests sobre la marcha. El sitio donde se resuelven las discusiones sobre si dos hashes "son distintos".

---

*En resumen: un hash criptográfico es una huella digital de tamaño fijo —determinista, irreversible, hipersensible a cambios y sin colisiones prácticas— que permite comparar y verificar datos sin exponerlos; pero fija los bytes exactos antes de calcularlo, y en cuanto haya un secreto de por medio deja de ser un hash lo que necesitas y pasa a ser un HMAC.*
