# Hash criptográfico

## ¿Qué es?

Un hash criptográfico es una función matemática que recibe datos de cualquier tamaño (un texto, un fichero, una contraseña) y devuelve siempre una "huella digital" de tamaño fijo, aparentemente aleatoria, llamada *digest* o simplemente *hash*.

## ¿Por qué existe?

Muchos problemas de seguridad e integridad se reducen a la misma pregunta: "¿estos datos son exactamente los que espero, sin haberlos guardado ni transmitido enteros?". Comparar dos ficheros de 4 GB byte a byte es caro; comparar sus huellas de 32 bytes es instantáneo. Y guardar una contraseña tal cual es un desastre esperando a ocurrir; guardar solo su huella permite verificarla sin conocerla.

> Piensa en el hash como la huella dactilar de una persona: ocupa poquísimo comparado con la persona entera, identifica de forma prácticamente única, y a partir de la huella no puedes reconstruir a la persona.

## ¿Cuándo y para qué se usa?

Aparece en casi cualquier sistema:

- **Verificar descargas**: la web publica el hash del fichero; tú lo recalculas tras descargarlo y compruebas que coincide (nadie lo alteró por el camino).
- **Almacenar credenciales**: una app de registro no guarda tu contraseña, guarda un derivado de ella (ver la ficha de [funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md)).
- **Detectar duplicados**: un servicio de fotos calcula el hash de cada imagen subida; si dos usuarias suben la misma foto, solo se almacena una vez.
- **Firmas digitales y Git**: no se firma el documento entero, se firma su hash. Git identifica cada commit por el hash de su contenido.

## Lo mínimo que necesitas saber

Las cuatro propiedades que convierten una función de resumen en un hash *criptográfico*:

**1. Determinista**

La misma entrada produce siempre la misma salida, en cualquier máquina y en cualquier momento. Sin esto no podrías comparar huellas.

```bash
echo -n "hola" | sha256sum
# b221d9dbb083a7f33428d7c2a3c3198ae925614d70210e28716ccaa7cd4ddb79  (siempre)
```

**2. Unidireccional (resistencia a preimagen)**

Del hash no se puede volver a la entrada original. La única vía es probar entradas una a una hasta encontrar una que produzca ese hash — computacionalmente inviable si la entrada tiene suficiente entropía.

**3. Efecto avalancha**

Cambiar un solo bit de la entrada cambia, de media, la mitad de los bits de la salida. Dos entradas casi idénticas producen hashes que no se parecen en nada, así que el hash no filtra "cuánto se parecen" dos datos.

```bash
echo -n "hola" | sha256sum   # b221d9db...
echo -n "holA" | sha256sum   # 6ee54981...  (irreconocible respecto al anterior)
```

**4. Resistencia a colisiones**

Debe ser inviable encontrar dos entradas distintas con el mismo hash. Las colisiones existen (hay infinitas entradas y salidas finitas), pero encontrarlas a propósito debe costar más de lo que ningún atacante puede pagar. Cuando alguien aprende a fabricar colisiones de un algoritmo (le pasó a MD5 y a SHA-1), ese algoritmo queda roto para usos de seguridad.

**5. El tamaño de salida es fijo y se mide en bits**

SHA-256 produce 256 bits (32 bytes, 64 caracteres en hexadecimal) sin importar si la entrada es una letra o una película. Más bits ⇒ más resistencia: con 256 bits, la probabilidad de colisión accidental es despreciable a cualquier escala práctica.

## Lo que NO hace

- **No cifra** — el cifrado es reversible con una clave; el hash no es reversible por diseño. Son herramientas distintas para problemas distintos.
- **No garantiza autenticidad por sí solo** — cualquiera puede recalcular el hash de un dato alterado. Para autenticar hace falta combinarlo con un secreto (HMAC) o una firma digital.
- **No oculta entradas predecibles** — si la entrada es "1234", un atacante la adivina probando hashes de entradas comunes. Por eso las contraseñas necesitan un tratamiento especial (ver ficha de derivación de claves).

---

*En resumen: un hash criptográfico es una huella digital de tamaño fijo — determinista, irreversible, hipersensible a cambios y sin colisiones prácticas — que permite comparar y verificar datos sin exponerlos.*
