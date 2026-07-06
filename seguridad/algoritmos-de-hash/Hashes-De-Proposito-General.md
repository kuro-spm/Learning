# Hashes de propósito general: SHA-2, SHA-3 y BLAKE

## ¿Qué es?

Son las familias de hash criptográfico modernas y rápidas que se usan para todo lo que no sea contraseñas: verificar integridad, firmar documentos, generar identificadores de contenido o deduplicar datos. Las tres consideradas seguras hoy son **SHA-2**, **SHA-3** y **BLAKE2/BLAKE3**.

## ¿Por qué existe?

Los algoritmos de hash envejecen: la potencia de cálculo crece y la criptoanálisis avanza, y lo que era inviable en 1995 es un ataque de fin de semana en 2020. **MD5** (1992) y **SHA-1** (1995) fueron el estándar durante años, pero hoy se sabe fabricar colisiones de ambos — con MD5 en segundos, y con SHA-1 desde el ataque *SHAttered* (2017), que produjo dos PDF distintos con el mismo hash. Las familias modernas existen para reemplazarlos con márgenes de seguridad amplios y, en el caso de BLAKE, además con mucha más velocidad.

> Piensa en las familias de hash como en las cerraduras de una puerta: las antiguas no eran "malas" cuando se instalaron, pero hoy cualquier cerrajero abre una cerradura de los 90 en segundos. Se cambia la cerradura, no la puerta.

## ¿Cuándo y para qué se usa?

- **Integridad de ficheros y descargas**: una web de software publica el SHA-256 de cada instalador; la usuaria lo recalcula y compara.
- **Firmas digitales y certificados**: lo que se firma es el hash del documento, no el documento. TLS y las firmas de código usan SHA-2 de forma masiva.
- **ETags y caché HTTP**: un servidor puede usar el hash del contenido como `ETag`; si el hash no cambió, responde `304 Not Modified` y se ahorra reenviar el cuerpo.
- **Deduplicación y direccionamiento por contenido**: sistemas de backup, Git o almacenes de blobs identifican cada bloque por su hash; dos bloques con el mismo hash se guardan una sola vez.

## Lo mínimo que necesitas saber

**1. SHA-2: el estándar por defecto**

Familia publicada por el NIST en 2001: **SHA-256** y **SHA-512** son sus miembros más usados (el número es el tamaño de salida en bits). Es el algoritmo con más soporte del mundo — hardware, librerías estándar, normativas — y no tiene ataques prácticos conocidos. Si dudas, SHA-256.

```bash
sha256sum instalador.exe
# 3f79bb7b...  instalador.exe
```

**2. SHA-3: el sustituto de reserva**

Ganador de un concurso del NIST (2015), basado en una construcción interna (*sponge*, del algoritmo Keccak) completamente distinta a la de SHA-2. No existe porque SHA-2 esté roto, sino como **seguro de vida**: si algún día aparece un ataque contra la construcción de SHA-2, SHA-3 no se vería afectado. En la práctica se usa poco; su variante derivada SHAKE permite salidas de longitud arbitraria.

**3. BLAKE2 y BLAKE3: los rápidos**

**BLAKE2** (2012) ofrece seguridad comparable a SHA-3 siendo más rápido que MD5; es el hash de librerías como libsodium. **BLAKE3** (2020) va más allá: se paraleliza internamente (árbol de hashes) y en CPUs modernas es varias veces más rápido que SHA-256. Ideales cuando el volumen importa: deduplicar terabytes, hashear en cada request, sincronizar ficheros.

```bash
b3sum video-4gb.mp4
# af1349b9...  video-4gb.mp4  (mucho más rápido que sha256sum en ficheros grandes)
```

**4. Los que hay que evitar: MD5 y SHA-1**

Ambos tienen **colisiones fabricables**: un atacante puede crear dos contenidos distintos con el mismo hash, lo que rompe firmas, certificados y cualquier verificación de integridad frente a un adversario. Git migró de SHA-1 precisamente por esto, y los navegadores rechazan certificados firmados con SHA-1 desde 2017. Solo son tolerables en usos sin adversario (p. ej. un checksum interno de caché), y aun así no hay motivo para elegirlos en código nuevo.

**5. Cómo elegir**

| Necesidad | Elección |
|---|---|
| Compatibilidad máxima, normativa, "no quiero pensar" | SHA-256 |
| Diversificación frente a SHA-2 o requisito explícito | SHA-3 |
| Rendimiento en grandes volúmenes | BLAKE3 (o BLAKE2) |
| Contraseñas | **Ninguno de estos** — ver [funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md) |

## Lo que NO hace

- **No protegen contraseñas** — son deliberadamente rápidos, y esa velocidad es justo lo que un atacante necesita para probar miles de millones de contraseñas por segundo.
- **No autentican** — un hash sin secreto solo detecta corrupción accidental; contra un atacante que puede sustituir dato y hash a la vez hace falta HMAC o firma.
- **No anonimizan datos predecibles** — hashear un DNI o un teléfono no lo protege: el espacio de entradas es tan pequeño que se revierte por fuerza bruta.

## Buenas prácticas avanzadas

- **La resistencia real a colisiones es la mitad de los bits** — por la paradoja del cumpleaños, SHA-256 ofrece ~128 bits contra colisiones, no 256. Importa sobre todo al truncar: usar los primeros 64 bits de un hash como clave de deduplicación empieza a dar colisiones esperables hacia los ~5.000 millones de elementos — lejísimos si deduplicas fotos, incómodamente cerca si troceas backups en bloques de 4 KB.
- **SHA-2 sufre extensión de longitud; SHA-3 y BLAKE no** — con SHA-256, conocer `hash(secreto ∥ dato)` permite calcular hashes de mensajes extendidos sin saber el secreto, así que jamás sirve como MAC casero (para eso, HMAC). SHA-3 y BLAKE2/3 son inmunes por construcción, y BLAKE2/3 además traen modo *keyed* nativo: sustituye a HMAC con una sola llamada y más velocidad.
- **En CPUs de 64 bits, SHA-512 suele ganar a SHA-256** — trabaja con palabras de 64 bits y procesa más datos por ronda. La variante **SHA-512/256** (SHA-512 truncado a 32 bytes de salida) combina ese rendimiento con inmunidad a la extensión de longitud. Eso sí: las instrucciones SHA-NI de los procesadores modernos aceleran SHA-256 por hardware e invierten la tortilla — mide en tu máquina de producción, no en la de desarrollo.
- **Hashea en streaming, no cargando el fichero en memoria** — toda librería seria ofrece API incremental (init/update/final) que consume el dato a trozos con memoria constante; es la diferencia entre hashear un vídeo de 4 GB y tumbar el proceso. Y matiz sobre BLAKE3: su paralelismo interno solo luce con entradas grandes o varios hilos; para millones de entradas pequeñas la ventaja sobre BLAKE2 o un SHA-256 acelerado se difumina.

---

*En resumen: SHA-256 como opción por defecto, BLAKE3 cuando la velocidad importa, SHA-3 como reserva estratégica — y MD5 y SHA-1 jubilados para cualquier uso con adversario.*
