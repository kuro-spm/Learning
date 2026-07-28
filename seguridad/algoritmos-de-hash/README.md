# Algoritmos de hash modernos — Guía de tecnologías

Qué hash usar en cada situación y por qué, que es una pregunta con más filo del que parece: el algoritmo correcto para verificar un fichero es exactamente el incorrecto para guardar una contraseña, y quien no tiene claro el criterio se equivoca en los dos sentidos.

La colección va de lo conceptual a la decisión: primero las propiedades del hash criptográfico y el vocabulario, luego las dos grandes familias —rápidos para datos, lentos para contraseñas—, después la desambiguación de tres términos que se confunden constantemente, y por último la práctica en .NET y el criterio que cierra el tema.

Todas las fichas comparten escenario: la API de una tienda online, con la tabla `Clientes` y su columna `PasswordHash`, la tabla `Sesiones` con `TokenHash`, y un `instalador.exe` publicado con su hash para los ejemplos de integridad.

---

## Orden de lectura recomendado

### 1. Fundamentos

Las propiedades en las que se apoya todo lo demás, y las cuatro herramientas que se confunden entre sí.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Hash criptográfico](Hash-Criptografico.md) | Las cinco propiedades (determinista, unidireccional, avalancha, resistencia a colisiones, salida fija), la paradoja del cumpleaños y por qué la resistencia real es de la mitad de los bits, la tabla que distingue hash de cifrado, MAC y firma, y el ataque de extensión de longitud que hace que `SHA-256(secreto ‖ mensaje)` no sea una firma. |

### 2. Las dos familias

Rápidos para datos, lentos para contraseñas. Entender por qué existen las dos es la mitad de la colección.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [Hashes de propósito general](Hashes-De-Proposito-General.md) | SHA-2, SHA-3 y BLAKE2/BLAKE3: integridad, `ETag`, deduplicación y firmas. Incluye por qué MD5 y SHA-1 están jubilados, y la distinción —que casi nadie hace— entre colisión, colisión con prefijo elegido y preimagen, que es la que decide si un algoritmo roto te afecta. |
| 3 | [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md) | La ficha más importante en la práctica: por qué un hash rápido no vale para contraseñas, con el cálculo de cuánto tarda una GPU en agotar un diccionario según el algoritmo. PBKDF2, bcrypt, scrypt y Argon2id, el factor de coste, qué significa *memory-hard*, y el login como vector de denegación de servicio. |
| 4 | [Sal vs seed](Sal-Vs-Seed.md) | Deshace la confusión entre la sal criptográfica, el sembrado de datos de una base de datos y la semilla de un generador. Incluye la trampa donde se cruzan: sembrar una contraseña hasheada no es determinista, y un seed que lo ignore reescribe la misma fila para siempre. |

### 3. En la práctica

Cómo se materializa todo lo anterior en código, y el criterio para decidir.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 5 | [Hashing en C#/.NET](Hashing-En-CSharp.md) | Qué trae la plataforma y qué no: `SHA256.HashData`, `IncrementalHash`, `PasswordHasher` con su re-hash automático, `RandomNumberGenerator`, y los paquetes de bcrypt y Argon2 con sus licencias. Incluye por qué `GetHashCode()` no es un hash y cambia en cada ejecución. |
| 6 | [Contraseñas vs tokens de sesión](Contrasenas-Vs-Tokens-De-Sesion.md) | El criterio que cierra la colección: lo que decide si toca un hash lento o uno rápido no es la moda, es la **entropía del secreto**. Con la tabla de decisión por tipo de secreto y el caso de los códigos cortos, donde no lo resuelve el algoritmo sino el límite de intentos. |

---

## El criterio, en una tabla

Si solo te llevas una cosa de la colección, que sea esta:

| Qué estás guardando | De dónde sale | Algoritmo | Por qué |
|---|---|---|---|
| Contraseña de una persona | La elige un humano | KDF lenta con sal (Argon2id) | El espacio es enumerable: cada intento tiene que costar caro |
| PIN o código de pocos dígitos | La elige un humano, o son pocos dígitos | KDF lenta **y** límite de intentos | Un millón de combinaciones se agotan; aquí el algoritmo no basta |
| Token de sesión o clave de API | Lo genera un CSPRNG, 256 bits | SHA-256 | Contra 2²⁵⁶ no hay diccionario, y se valida en cada petición |
| Fichero, contenido, `ETag` | No es un secreto | SHA-256, o BLAKE3 por volumen | Solo hace falta detectar cambios |
| Cualquier cosa con un secreto de por medio | — | HMAC, no un hash a secas | Un hash sin clave no autentica a nadie |

## Los errores que se pagan caros

| Síntoma | Causa habitual | Dónde se explica |
|---|---|---|
| Dos clientas con la misma contraseña tienen el mismo `PasswordHash` | Falta la sal, o es fija para todas | [Sal vs seed](Sal-Vs-Seed.md) |
| El login tarda dos segundos | El coste se calibró en otra máquina, o falta límite de concurrencia | [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md) |
| Validar la sesión consume toda la CPU | Hay una KDF en el camino de cada petición | [Contraseñas vs tokens de sesión](Contrasenas-Vs-Tokens-De-Sesion.md) |
| Una contraseña larguísima valida con solo sus primeras letras | bcrypt solo considera los primeros 72 bytes | [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md) |
| Dos servicios calculan hashes distintos del «mismo» dato | Nadie fijó los bytes: codificación, orden de campos o finales de línea | [Hash criptográfico](Hash-Criptografico.md) |
| Los identificadores dejan de coincidir tras un reinicio | Se derivaron de `GetHashCode()`, que está aleatorizado por proceso | [Hashing en C#/.NET](Hashing-En-CSharp.md) |
| Un token de sesión resulta predecible | Se generó con `Random` en lugar de `RandomNumberGenerator` | [Sal vs seed](Sal-Vs-Seed.md) |
| Alguien firma peticiones que tú no emitiste | Se usó `hash(secreto ‖ mensaje)` como firma en lugar de HMAC | [Hash criptográfico](Hash-Criptografico.md) |

## Comprobaciones rápidas

```bash
# ¿Qué digest tiene realmente este fichero?
sha256sum instalador.exe

# ¿La columna PasswordHash guarda blobs con sal distinta en cada fila?
# Dos filas idénticas con la misma contraseña son una señal de alarma.
```

```sql
SELECT PasswordHash, COUNT(*) FROM Clientes GROUP BY PasswordHash HAVING COUNT(*) > 1;
```

Si esa consulta devuelve algo, la sal falta o es fija, y el paso siguiente está en [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md).

---

> Buenos siguientes pasos fuera de esta colección: **HMAC y la firma de tokens** (HS256 frente a RS256) en [Autenticación y autorización](../autenticacion-y-autorizacion/README.md); el **cifrado reversible**, que es lo que toca cuando el secreto hay que recuperarlo y no solo verificarlo, en [Cifrado en reposo de credenciales](../gestion-de-secretos-en-desarrollo/Cifrado-En-Reposo-De-Credenciales.md); y el hashing **no** criptográfico (xxHash, las tablas hash de toda la vida) para los casos en los que ni siquiera hace falta seguridad.
