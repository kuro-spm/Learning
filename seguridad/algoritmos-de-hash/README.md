# Algoritmos de hash modernos — Guía práctica

Tutorial introductorio sobre **hashing criptográfico**, pensado para desarrolladoras full-stack que quieren entender qué hash usar en cada situación: verificar integridad, guardar contraseñas o proteger tokens de sesión. Cada ficha explica qué es el concepto, por qué existe, cuándo se usa y lo mínimo que necesitas saber, con ejemplos genéricos que se entienden sin conocer ningún proyecto concreto.

Sigue el orden: primero el concepto, luego las dos grandes familias (rápidos y lentos), y por último la práctica en C#/.NET y cómo decidir en un caso real.

---

## Orden de lectura recomendado

### 1. Fundamentos

Qué es un hash criptográfico y las propiedades que lo hacen útil.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Hash criptográfico](Hash-Criptografico.md) | Las cuatro propiedades (determinista, unidireccional, avalancha, colisiones) en las que se apoya todo lo demás. |

### 2. Las dos familias

Hashes rápidos para datos, hashes lentos para contraseñas. Entender por qué existen ambas es la mitad del tutorial.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [Hashes de propósito general](Hashes-De-Proposito-General.md) | SHA-2, SHA-3 y BLAKE2/BLAKE3: integridad, firmas, ETags, deduplicación — y por qué MD5 y SHA-1 están jubilados. |
| 3 | [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md) | Por qué un hash rápido no vale para contraseñas: PBKDF2, bcrypt, scrypt y Argon2id, con las recomendaciones OWASP. |
| 4 | [Sal vs seed](Sal-Vs-Seed.md) | Deshace la confusión entre la sal de las KDF y los dos "seeds" que verás cerca (datos iniciales y semilla aleatoria). |

### 3. En la práctica

Cómo se materializa todo lo anterior en código y en decisiones de diseño.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 5 | [Hashing en C#/.NET](Hashing-En-CSharp.md) | `SHA256.HashData`, `PasswordHasher` (PBKDF2), BCrypt.Net-Next y los paquetes Argon2 con sus licencias. |
| 6 | [Contraseñas vs tokens de sesión](Contrasenas-Vs-Tokens-De-Sesion.md) | El criterio para elegir hash lento o rápido según la entropía del secreto, con un caso real de referencia. |

---

> ¿Te quedas con ganas de más? Buenos siguientes pasos: **HMAC** (hash con secreto para autenticar mensajes), la **firma de JWT** (HS256 vs RS256), y el hashing *no* criptográfico (xxHash, las hash tables de toda la vida) para entender cuándo ni siquiera hace falta seguridad.
