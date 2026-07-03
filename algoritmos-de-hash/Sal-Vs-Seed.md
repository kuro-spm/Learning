# Sal vs seed: parecidos de nombre, nada que ver

**¿Qué es?** Una desambiguación de tres términos que aparecen juntos en cualquier feature de autenticación y se confunden con facilidad: la **sal** (*salt*, criptografía), el **seed de datos** (sembrar registros iniciales en una base de datos) y la **seed aleatoria** (la semilla de un generador de números pseudoaleatorios). Solo comparten la metáfora de "algo pequeño que se pone al principio"; resuelven problemas completamente distintos.

---

## ¿Por qué existe?

Porque en la documentación de un mismo login puedes leer "hash+sal" y "Admin seed" en el mismo párrafo, y es natural preguntarse si hablan de lo mismo. No: *salt* se traduce como sal y *seed* como semilla, pero al mezclar español e inglés en los documentos técnicos los dos acaban sonando a "ingrediente inicial".

- La **sal** vive en el mundo criptográfico: existe para que hashear la misma contraseña dos veces no produzca el mismo resultado.
- El **seed de datos** vive en el mundo de las bases de datos: existe para que una base de datos recién creada no arranque vacía (sin al menos una cuenta de administración, nadie podría entrar jamás).
- La **seed aleatoria** vive en el mundo de los generadores: existe para poder *reproducir* una secuencia "aleatoria" (el mismo seed produce siempre la misma secuencia).

> Piensa en una cocina: la sal es un ingrediente que añades a la receta antes de hornear; el seed de datos es dejar la despensa con provisiones básicas antes de estrenar la casa; y la seed aleatoria es el número de página por el que abres siempre el mismo libro de recetas.

---

## ¿Cuándo y para qué se usa?

- **Sal** — al guardar contraseñas con una función de derivación ([ficha de KDF](Funciones-De-Derivacion-De-Claves.md)): un valor aleatorio único por contraseña, mezclado con ella antes de hashear. Evita las *rainbow tables* y que dos usuarias con la misma contraseña compartan hash.
- **Seed de datos** — en migraciones o scripts de arranque: el INSERT que deja creada la cuenta admin, el catálogo de roles o los datos de demo en una base de datos vacía.
- **Seed aleatoria** — en tests que necesitan datos "aleatorios" pero reproducibles, en simulaciones, y en generación de imágenes por IA (misma seed + mismo prompt = misma imagen).

---

## Lo mínimo que necesitas saber

**1. La sal no se gestiona a mano: va incrustada en el blob**

Las librerías modernas generan la sal por ti y la guardan *dentro* del propio resultado, junto al algoritmo y sus parámetros. Por eso la columna de la contraseña es un único string opaco:

```csharp
var hasher = new PasswordHasher<User>();
string blob = hasher.HashPassword(user, "MiContraseña$egura");
// blob = base64(formato + parámetros + SAL aleatoria de 128 bits + hash)
// Al verificar, el hasher extrae la sal del blob y repite el cálculo:
var resultado = hasher.VerifyHashedPassword(user, blob, "MiContraseña$egura");
```

**2. La sal no es secreta**

Se guarda en claro junto al hash y no pasa nada: su valor no está en ser oculta sino en ser **única por contraseña**, para que cada cuenta haya que atacarla por separado.

**3. El seed de datos es un INSERT idempotente**

Sembrar datos debe poder ejecutarse mil veces sin duplicar nada. El patrón típico en una migración:

```sql
INSERT INTO "User" ("Email", "PasswordHash", "Role")
VALUES ('admin@example.com', '<hash-con-su-sal-dentro>', 'ADMIN')
ON CONFLICT ("Email") DO NOTHING;   -- si ya existe, no hace nada
```

**4. La seed aleatoria hace reproducible lo aleatorio**

```csharp
var rng = new Random(42);       // misma seed...
rng.Next(100);                  // ...misma secuencia de "aleatorios", siempre
```

Útil en tests y simulaciones. Y justo por esa reproducibilidad, los generadores **criptográficos** (`RandomNumberGenerator`) no aceptan seed: que nadie pueda reproducir la secuencia es precisamente el requisito.

**5. Cómo distinguirlos al leer documentación**

Si la palabra aparece cerca de *hash*, *contraseña* o *KDF* → es la **sal**. Si aparece cerca de *migración*, *datos iniciales* o *admin* → es un **seed de datos**. Si aparece cerca de *generador*, *reproducible* o *prompt* → es una **seed aleatoria**.

---

## Lo que NO hace

- **La sal no fortalece una contraseña débil** — solo impide ataques precalculados y en masa; la lentitud la pone la KDF y la calidad la pone la contraseña.
- **El seed de datos no aporta seguridad** — es solo contenido inicial; si siembra credenciales temporales, hay que rotarlas antes de producción.
- **La seed aleatoria no vale para secretos** — un token generado con `new Random(seed)` es predecible; los secretos exigen un CSPRNG sin seed.

---

*En resumen: la sal es aleatoriedad única que se mezcla con cada contraseña antes de hashear; el seed de datos es lo que plantas en una base de datos vacía para que arranque; y la seed aleatoria hace repetible a un generador — tres "semillas" que solo comparten el nombre.*
