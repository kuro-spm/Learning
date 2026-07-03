# Funciones de derivación de claves (PBKDF2, bcrypt, scrypt, Argon2)

## ¿Qué es?

Son funciones de hash **deliberadamente lentas y configurables** diseñadas para proteger contraseñas. A partir de una contraseña, una sal aleatoria y un factor de coste, derivan un hash cuyo cálculo cuesta un tiempo apreciable — a propósito. También se las llama KDF (*key derivation functions*) o *password hashing functions*.

## ¿Por qué existe?

Un hash rápido como SHA-256 es pésimo para contraseñas. Cuando a una empresa le roban la base de datos, el atacante tiene los hashes en su máquina y puede probar contraseñas *offline* sin límite de intentos: una GPU doméstica calcula **miles de millones de SHA-256 por segundo**, así que un diccionario con las contraseñas humanas más comunes cae en minutos. El problema no es que el atacante "revierta" el hash — es que las contraseñas humanas son adivinables, y la velocidad del hash decide cuántos intentos por segundo puede hacer.

La defensa es encarecer cada intento. Si verificar una contraseña cuesta 100 ms en vez de 1 nanosegundo, el login legítimo ni lo nota (una verificación), pero el atacante pasa de miles de millones de intentos por segundo a unas decenas.

> Piensa en la diferencia entre una puerta normal y la puerta de una cámara acorazada con apertura retardada: abrirla una vez al día es asumible; intentar un millón de combinaciones se vuelve una condena.

## ¿Cuándo y para qué se usa?

- **Almacenar contraseñas de usuarias** en cualquier aplicación con registro propio (una tienda online, un SaaS, una intranet). Es su uso principal y casi único.
- **Derivar claves de cifrado a partir de una contraseña**: un gestor de contraseñas o un disco cifrado convierten tu contraseña maestra en la clave AES real usando una KDF.
- **No** para tokens aleatorios, ficheros ni firmas — ahí la lentitud no aporta nada (ver [contraseñas vs tokens de sesión](Contrasenas-Vs-Tokens-De-Sesion.md)).

## Lo mínimo que necesitas saber

**1. La sal: una por contraseña**

La *sal* es un valor aleatorio único que se mezcla con la contraseña antes de hashear y se guarda en claro junto al hash. Impide dos ataques: las *rainbow tables* (tablas precalculadas de hashes de contraseñas comunes) y atacar todas las cuentas a la vez — con sales distintas, dos usuarias con la misma contraseña tienen hashes distintos y cada cuenta hay que atacarla por separado. Todas las funciones de esta ficha la generan y gestionan por ti.

**2. El factor de coste: la lentitud es un parámetro**

Todas exponen un ajuste de coste (iteraciones, factor de trabajo, memoria...) para que la función pueda seguir siendo lenta aunque el hardware mejore. Se calibra para que verificar cueste del orden de decenas a centenas de milisegundos en tu servidor, y se sube con los años.

**3. Memoria dura (memory-hard): la defensa contra GPUs**

Una GPU tiene miles de núcleos pero poca memoria rápida por núcleo. Las funciones *memory-hard* (scrypt, Argon2) exigen decenas de megabytes de RAM por cada cálculo, de modo que la GPU no puede ejecutar miles en paralelo. Las que solo queman CPU (PBKDF2, y bcrypt en menor medida) se aceleran mucho mejor con hardware especializado.

**4. Las cuatro candidatas**

| Función | Año | ¿Memory-hard? | Factor de coste | Notas |
|---|---|---|---|---|
| **PBKDF2** | 2000 | No | Iteraciones | Estándar NIST/FIPS; disponible en cualquier plataforma. La más débil frente a GPUs, pero correcta con iteraciones altas. |
| **bcrypt** | 1999 | No (usa 4 KB, algo hostil a GPUs) | Factor de trabajo (exponencial: 2^n) | Veterana y sólida. Limitación: solo usa los primeros 72 bytes de la contraseña. |
| **scrypt** | 2009 | Sí | CPU + memoria (N, r, p) | Primera memory-hard popular. Correcta, aunque Argon2 la ha relevado. |
| **Argon2id** | 2015 | Sí | Memoria + iteraciones + paralelismo | Ganadora de la Password Hashing Competition. La recomendación actual. |

**5. Recomendaciones OWASP actuales**

La *Password Storage Cheat Sheet* de OWASP recomienda, por orden:

1. **Argon2id** — mínimo orientativo: 19 MiB de memoria, 2 iteraciones, paralelismo 1.
2. **scrypt** si Argon2 no está disponible — N=2^17, r=8, p=1.
3. **bcrypt** en sistemas legacy — factor de trabajo 10 o más.
4. **PBKDF2** cuando se exige cumplimiento FIPS-140 — con HMAC-SHA256, unas 600.000 iteraciones.

Y siempre: sal única por contraseña (las librerías lo hacen solas) y coste recalibrado periódicamente.

**6. El formato de salida se autodescribe**

Estas funciones no devuelven solo el hash: devuelven una cadena con algoritmo, versión, parámetros, sal y hash, todo junto. Así puedes verificar contraseñas antiguas y subir el coste para las nuevas sin migraciones traumáticas.

```text
$argon2id$v=19$m=19456,t=2,p=1$C7RPZ4qRolZ5...$5d6P0PU3q2kX...
```

## Lo que NO hace

- **No hace segura una contraseña débil** — "123456" cae en el primer intento del diccionario por lenta que sea la función. La KDF compra tiempo, no milagros.
- **No sustituye al control de intentos online** — el rate-limiting del login y el bloqueo de cuenta siguen haciendo falta; la KDF protege sobre todo cuando la base de datos ya se ha filtrado.
- **No sirve para datos que no sean secretos adivinables** — hashear un fichero o un token de 256 bits con Argon2 solo desperdicia CPU y memoria.

## Buenas prácticas avanzadas

- **Calibra en el hardware de producción y trata el login como vector de DoS** — el coste que en tu portátil da 50 ms puede dar 400 ms en el contenedor pequeño de producción. Y una KDF cara convierte el endpoint de login en un objetivo: unas decenas de peticiones simultáneas con contraseñas inventadas bastan para saturar la CPU. Además del rate-limiting por IP y cuenta, limita cuántas verificaciones se ejecutan a la vez (una cola o un semáforo).
- **Pepper: el secreto que la sal no es** — sobre la sal (pública, en la BD) puedes añadir un HMAC con una clave secreta guardada *fuera* de la base de datos (variable de entorno, KMS) antes de pasar por la KDF. En el escenario típico de filtración — inyección SQL, backup perdido — el atacante tiene los hashes pero no el pepper, y no puede probar ni una sola contraseña. A cambio, asumes gestionar y rotar esa clave.
- **Pre-hashear para esquivar el límite de bcrypt tiene trampa** — el apaño `bcrypt(sha256(contraseña))` para superar los 72 bytes introduce dos fallos sutiles: el digest binario puede contener bytes nulos que bcrypt trunca, y habilita el *password shucking* — si el atacante tiene SHA-256 filtrados de otra brecha, los prueba directamente como "contraseñas" contra tus bcrypt. Si pre-hasheas, codifica el resultado en Base64 y usa HMAC con clave, no un hash a pelo.
- **Migra hashes legacy envolviéndolos, no esperando logins** — si heredas una tabla con MD5 o SHA-1, no esperes a que cada usuaria vuelva a entrar para re-hashear: aplica ya `argon2id(hash_viejo)` a todos los registros y anota el esquema en el formato. La base queda protegida hoy mismo, y en el siguiente login de cada cuenta sustituyes por el Argon2id de la contraseña original.
- **De las tres variantes de Argon2, siempre la id** — Argon2d resiste mejor las GPUs pero es vulnerable a ataques de canal lateral; Argon2i, al revés. Argon2**id** combina ambas y es la única que recomiendan OWASP y el RFC 9106 para contraseñas: si una librería te ofrece las tres, no hay decisión que tomar.

---

*En resumen: las contraseñas se protegen con funciones lentas a propósito — Argon2id como primera opción según OWASP — porque contra un ladrón de bases de datos la única defensa real es que cada intento le cueste caro.*
