# Sal vs seed: parecidos de nombre, nada que ver

## ¿Qué es?

Una desambiguación de tres términos que aparecen juntos en cualquier funcionalidad de autenticación y se confunden con facilidad: la **sal** (*salt*), el **seed de datos** y la **seed aleatoria**.

- La **sal** es un valor aleatorio distinto para cada contraseña que se mezcla con ella antes de hashearla.
- El **seed de datos** (*data seeding*, «sembrar datos») son los registros mínimos que se insertan en una base de datos recién creada para que sea utilizable.
- La **seed aleatoria** (*random seed*, «semilla») es el número con el que se arranca un generador de números pseudoaleatorios para que produzca siempre la misma secuencia.

Solo comparten la metáfora de «algo pequeño que se pone al principio». Resuelven problemas completamente distintos y viven en capas distintas del sistema.

## ¿Por qué existe?

Porque en la documentación de un mismo login puedes leer «hash + sal» y «seed del admin» en el mismo párrafo, y es razonable preguntarse si hablan de lo mismo. La confusión tiene dos causas concretas. La primera es la traducción: *salt* se traduce como sal y *seed* como semilla, y al mezclar español e inglés en el mismo documento técnico los dos acaban sonando a «ingrediente inicial». La segunda es que ambas cosas ocurren de verdad en el mismo sitio —la creación de la primera cuenta de la tienda— así que el contexto no ayuda a separarlas.

Confundirlas no es un problema estético: cada una tiene una regla de oro distinta —la sal debe ser única, el seed de datos debe ser idempotente, la seed aleatoria jamás debe usarse para un secreto— y aplicar la regla de una a otra produce fallos reales, que es lo que recorre esta ficha.

> Piensa en una cocina. La **sal** es un ingrediente que añades a cada masa antes de hornearla, y echas una pizca distinta en cada una. El **seed de datos** es dejar la despensa con provisiones básicas antes de estrenar la casa, para que quien llegue pueda cocinar algo. Y la **seed aleatoria** es el número de página por el que abres siempre el mismo libro de recetas: si lo anotas, mañana repites el plato exacto.

## ¿Cuándo y para qué se usa?

El ejemplo que atraviesa la ficha es una **tienda online con API en .NET** y una base de datos con dos tablas: `Clientes`, con la columna `PasswordHash`, y `Sesiones`, con la columna `TokenHash`.

- **Sal** — al guardar la contraseña de una clienta con una función de derivación de claves o **KDF** (*Key Derivation Function*: un hash deliberadamente lento y con sal, diseñado para contraseñas; su detalle está en [Funciones de derivación de claves](Funciones-De-Derivacion-De-Claves.md)). Evita que dos clientas con la misma contraseña compartan el mismo `PasswordHash`, y evita las *rainbow tables*: catálogos precalculados de contraseña → hash que dejan de servir en cuanto cada contraseña lleva su propia sal.
- **Seed de datos** — en una migración o un script de arranque: el `INSERT` que deja creada la cuenta de administración de la tienda, el catálogo de formas de pago o los datos de demostración.
- **Seed aleatoria** — en tests que necesitan datos «aleatorios» pero reproducibles, en simulaciones y en generación de imágenes por IA (misma seed + mismo prompt = misma imagen).

## La tabla que resuelve la confusión

Esta tabla es la razón de existir de la ficha. Si te llevas una sola cosa, que sea esta.

| | **Sal** (*salt*) | **Seed de datos** | **Seed aleatoria** |
|---|---|---|---|
| **Mundo al que pertenece** | Criptografía: hashing de contraseñas | Bases de datos: migraciones y arranque | Generadores de números pseudoaleatorios |
| **Problema que resuelve** | Que hashear la misma contraseña dos veces dé el mismo resultado | Que una base de datos nueva arranque inutilizable, sin ninguna cuenta con la que entrar | Que una secuencia «aleatoria» no se pueda repetir para depurar o comparar |
| **¿Es secreta?** | **No.** Se guarda en claro junto al hash | No: está escrita en el repositorio | No, y suele imprimirse a propósito |
| **¿Única o compartida?** | **Única** por contraseña y por rehash | Un juego de datos compartido por todos los entornos | Fija a propósito, para repetir la secuencia |
| **Quién la genera** | La librería de hashing, sola, en cada llamada | Tú, escribiendo el script o la migración | Tú, eligiendo un número (o el reloj, si no eliges) |
| **Dónde se guarda** | **Dentro del propio hash**, en la misma columna `PasswordHash` | En filas normales de las tablas de negocio | En el código del test, o en su salida por consola |

## Cómo distinguirlos al leer documentación

Esta regla resuelve el 95 % de los casos y cabe en tres líneas. Mira las palabras que rodean al término:

| Si aparece cerca de… | Es… |
|---|---|
| *hash*, *contraseña*, *KDF*, *bcrypt*, *Argon2*, *rainbow table* | la **sal** |
| *migración*, *datos iniciales*, *admin*, *fixture*, *bootstrap* | un **seed de datos** |
| *generador*, *reproducible*, *determinista*, *simulación*, *prompt* | una **seed aleatoria** |

Un segundo indicio, aún más rápido: si el texto habla de un valor que **no controlas ni escribes**, es la sal; si habla de algo que **tú redactas** y acaba en el repositorio, es un seed de datos; si habla de un **número que eliges para poder repetir**, es una seed aleatoria.

## La sal, a fondo

La sal es un valor aleatorio, típicamente de 16 bytes, que se mezcla con la contraseña antes de pasarla por la KDF. Su efecto es que la misma contraseña produce un hash distinto cada vez que se guarda.

Lo primero que sorprende es que **la sal no es secreta**: viaja en claro, en la misma columna, al lado del hash que protege. Y no pasa nada, porque su valor no está en ser oculta sino en ser **única**. Una sal distinta por cuenta obliga a atacar cada contraseña por separado: el atacante no puede precalcular un diccionario y compararlo contra el millón de filas de la tabla, tiene que rehacer el cálculo entero para cada fila.

Lo segundo que sorprende es que **no la gestionas tú**. Las librerías modernas la generan y la guardan *dentro* del propio resultado, junto al algoritmo y sus parámetros. Por eso la columna `PasswordHash` es un único string opaco:

```csharp
var hasher = new PasswordHasher<Cliente>();
string blob = hasher.HashPassword(cliente, "T!enda-2026#Admin");

// Al verificar, el hasher extrae del blob la sal y los parámetros y repite el cálculo
PasswordVerificationResult r = hasher.VerifyHashedPassword(cliente, blob, "T!enda-2026#Admin");
// r == PasswordVerificationResult.Success
```

El valor de `blob` es este, y no hace falta creerse que la sal está dentro: se puede desglosar. Son 61 bytes en base64, 84 caracteres:

```
AQAAAAIAAYagAAAAECPtaWvyGQhzvE1l7oUkq4wrqA4DF0TspLEVn2OIgG8fmyxhU4ZYuvVjQ7HK4tZyZQ==
```

| Bytes | Valor | Qué es |
|---|---|---|
| 1 | `0x01` | versión del formato (el «v3» de ASP.NET Core Identity) |
| 4 | `0x00000002` | función interna: HMAC-SHA512 |
| 4 + 4 | `0x000186A0`, `0x00000010` | 100 000 iteraciones (el factor de coste) y longitud de la sal: 16 bytes |
| 16 | `23ED696B F2190873 BC4D65EE 8524AB8C` | **la sal** |
| 32 | el resto | el hash derivado |

Esto explica por qué la tabla `Clientes` no tiene ninguna columna `Salt`, y por qué buscarla es señal de que se ha entendido mal el formato. También explica por qué la verificación funciona sin más información: todo lo necesario para repetir el cálculo —algoritmo, coste y sal— viaja con el hash. Si mañana subes las iteraciones, los hashes antiguos siguen validando porque cada uno lleva su propio coste escrito dentro.

Los dos errores de gestión manual de la sal son estos, y ambos vienen de intentar hacer a mano lo que la librería ya hace:

```sql
-- ❌ Sal en una columna aparte: dos sitios que hay que mantener sincronizados
ALTER TABLE "Clientes" ADD COLUMN "Salt" bytea;
-- Un backup parcial o un UPDATE mal filtrado rompen la correspondencia entre
-- "Salt" y "PasswordHash", y a partir de ahí NADIE puede iniciar sesión.
```

El segundo es peor porque no rompe nada visible y parece funcionar:

```csharp
// ❌ Una sal fija para todas las cuentas: se pierde justo lo único que aporta la sal
byte[] salFija = Encoding.ASCII.GetBytes("tienda-sal-fija\0");
```

La demostración es inmediata. Dos clientas distintas eligen la misma contraseña, `Verano2026!`:

```
Lucía → AQAAAAIAAYagAAAAEHRpZW5kYS1zYWwtZmlqYQDYhQ4Rwf5Gz5mcZeWWb/dhSuTaVBHk5B0NpnJiRNYYnw==
Marta → AQAAAAIAAYagAAAAEHRpZW5kYS1zYWwtZmlqYQDYhQ4Rwf5Gz5mcZeWWb/dhSuTaVBHk5B0NpnJiRNYYnw==
```

Idénticos, carácter a carácter. Con eso, quien consiga la tabla `Clientes` agrupa por `PasswordHash`, ataca una sola vez el grupo más numeroso y abre de golpe todas las cuentas que compartían contraseña. Y como la sal fija está en el código, sirve igual para precalcular un diccionario contra toda la tabla. Con sal aleatoria por cuenta, los dos valores no se parecen en nada más allá de la cabecera de parámetros.

## El seed de datos, a fondo

Una base de datos recién creada sin ninguna cuenta de administración es una base de datos en la que nadie puede entrar: no hay forma de crear la primera cuenta porque crear cuentas requiere estar dentro. El seed de datos rompe ese círculo insertando lo mínimo imprescindible.

El requisito que lo define es la **idempotencia**: poder ejecutarse mil veces con el mismo efecto que una. Hace falta porque el seed corre en cada despliegue, en cada CI, en cada máquina de desarrollo, y a veces dos veces seguidas porque alguien reintentó el pipeline.

En PostgreSQL el patrón es `ON CONFLICT`, que necesita un índice único sobre la columna en la que se apoya:

```sql
-- ✅ Idempotente: si el correo ya existe, no hace nada
INSERT INTO "Clientes" ("Email", "PasswordHash", "Rol")
VALUES ('admin@tienda-online.example', @hash, 'ADMIN')
ON CONFLICT ("Email") DO NOTHING;
```

Ejecutado dos veces devuelve `INSERT 0 1` la primera y `INSERT 0 0` la segunda: la base de datos informa de que no ha tocado nada. En SQL Server no existe `ON CONFLICT` y el equivalente se escribe con una comprobación previa:

```sql
-- ✅ Misma idea en SQL Server
IF NOT EXISTS (SELECT 1 FROM Clientes WHERE Email = 'admin@tienda-online.example')
BEGIN
    INSERT INTO Clientes (Email, PasswordHash, Rol)
    VALUES ('admin@tienda-online.example', @hash, 'ADMIN');
END
```

Y en EF Core hay un mecanismo declarativo, `HasData`, que traduce los datos del modelo a `INSERT` dentro de la migración:

```csharp
modelBuilder.Entity<Cliente>().HasData(new Cliente
{
    Id = 1,                                        // la clave debe ser fija y explícita
    Email = "admin@tienda-online.example",
    Rol = "ADMIN"
});
```

`HasData` compara el modelo con el *snapshot* de la última migración y genera solo las diferencias, así que la idempotencia la da el propio sistema de migraciones. El precio es que la clave primaria tiene que estar fijada a mano, porque EF necesita identificar la fila entre versiones. El funcionamiento de las migraciones y cuándo conviene sembrar dentro o fuera de ellas está en [Migraciones de esquema](../../bases-de-datos/migraciones-de-esquema/README.md).

## La trampa donde se cruzan: sembrar una contraseña hasheada

Aquí es donde las dos primeras «semillas» chocan, y es el motivo de fondo por el que merece la pena distinguirlas.

**Hashear una contraseña no es determinista.** La KDF genera una sal nueva en cada llamada, así que el mismo `HashPassword` con la misma contraseña devuelve un valor distinto cada vez:

```csharp
var hasher = new PasswordHasher<Cliente>();
string primero = hasher.HashPassword(admin, "T!enda-2026#Admin");
string segundo = hasher.HashPassword(admin, "T!enda-2026#Admin");
```

```
primero → AQAAAAIAAYagAAAAECPtaWvyGQhzvE1l7oUkq4wrqA4DF0TspLEVn2OIgG8fmyxhU4ZYuvVjQ7HK4tZyZQ==
segundo → AQAAAAIAAYagAAAAECMR6OkWgTX8M+L14TSjN6YmOmQWs0Vr6Bdi8zW5rrPORlfGgFvIJZS913SRZyQXVA==
primero == segundo  → False
```

Fíjate en que los 17 primeros caracteres coinciden —son los parámetros, idénticos— y a partir de ahí no se parecen en nada: ahí empieza la sal. Y **los dos validan** la misma contraseña, porque cada uno lleva la suya dentro.

La consecuencia es un fallo silencioso y eterno. Un seed que hashea la contraseña en cada arranque y compara el resultado con lo almacenado para decidir si «hay cambios» encontrará siempre una diferencia, porque comparar dos hashes con sal distinta nunca da igualdad:

```csharp
// ❌ Reescribe la fila en cada arranque, para siempre
var hashNuevo = hasher.HashPassword(admin, contraseñaConfigurada);
if (clienteExistente.PasswordHash != hashNuevo)
    clienteExistente.PasswordHash = hashNuevo;   // esta condición es SIEMPRE verdadera
```

Con `HasData` la variante es peor, porque la reescritura se materializa en el repositorio: cada `dotnet ef migrations add` detecta que el valor sembrado ha cambiado y genera una migración nueva con un `UpdateData` de la columna `PasswordHash`. El historial se llena de migraciones que no significan nada.

La conclusión es doble.

**Primero: la idempotencia se apoya en la clave natural, nunca en el valor hasheado.** Existe o no existe una fila con ese correo; el contenido de `PasswordHash` no se compara jamás. Si de verdad quieres saber si la contraseña sembrada sigue siendo válida, no compares: verifica.

```csharp
// ✅ La decisión se toma sobre el correo, y la validez se comprueba verificando
var admin = await db.Clientes.SingleOrDefaultAsync(c => c.Email == correoAdmin);
if (admin is null)
{
    admin = new Cliente { Email = correoAdmin, Rol = "ADMIN" };
    admin.PasswordHash = hasher.HashPassword(admin, contraseñaConfigurada);
    db.Clientes.Add(admin);
    await db.SaveChangesAsync();
}
```

**Segundo: la contraseña de la cuenta sembrada llega por variable de entorno o gestor de secretos.** Un hash escrito a mano en la migración es una contraseña publicada en el repositorio: el hash es del algoritmo público más conocido, la contraseña casi siempre es del estilo `Admin123!`, y romperla ofrece la cuenta de administración de todas las instalaciones que hayan corrido ese seed.

```csharp
// ❌ Un hash literal en el código fuente
const string HashAdmin = "AQAAAAIAAYagAAAAECPtaWvyGQhzvE1l7oUkq4wrqA4DF0Tsp...";

// ✅ La contraseña entra desde fuera, y si no está, el arranque falla
string contraseñaConfigurada = config["Seed:AdminPassword"]
    ?? throw new InvalidOperationException("Falta Seed:AdminPassword");
```

Que falle el arranque es deliberado: una contraseña por defecto silenciosa es la que acaba en producción. Dónde poner ese valor en local y en despliegue lo cubre [Gestión de secretos en desarrollo](../gestion-de-secretos-en-desarrollo/README.md).

## La seed aleatoria, a fondo

Un generador de números pseudoaleatorios (**PRNG**) no produce azar: produce una secuencia determinista que *parece* aleatoria, calculada a partir de un estado interno. La seed es el valor inicial de ese estado. Misma seed, misma secuencia, siempre y en cualquier máquina:

```csharp
var rng = new Random(42);
for (int i = 0; i < 5; i++) Console.Write($"{rng.Next(100)} ");
```

```
66 14 12 52 16
```

Ejecútalo mil veces y saldrá exactamente lo mismo. Cambia el 42 por un 7 y saldrá `38 87 66 5 36`, también siempre igual. Esa reproducibilidad es toda la utilidad: un test que genera 10 000 pedidos con datos variados y falla puede repetirse tal cual, y una simulación se puede comparar entre dos versiones del código sabiendo que la entrada fue idéntica.

Y aquí está la línea que no se cruza nunca: **un generador con seed no sirve para generar secretos.** El token que guardas hasheado en `Sesiones.TokenHash` es exactamente eso, un secreto.

```csharp
// ❌ Token de sesión predecible
var rng = new Random();
var bytes = new byte[32];
rng.NextBytes(bytes);
string token = Convert.ToBase64String(bytes);
```

Ese código compila, pasa los tests y produce cadenas que parecen aleatorias. Es inseguro por dos razones distintas:

- **El estado es reconstruible.** Los PRNG de propósito general están diseñados para ser rápidos y estadísticamente uniformes, no para ocultar su estado. Con unas pocas salidas observadas se recupera el estado interno y a partir de ahí se predicen todas las siguientes. Es un resultado conocido para los generadores clásicos: con el Mersenne Twister bastan 624 salidas consecutivas. En una tienda, «unas pocas salidas» es trivial de conseguir: crea varias sesiones tú, mira tus propios tokens y calcula el siguiente, que será de otra persona.
- **La semilla puede ser adivinable.** En .NET Framework y en .NET hasta la versión 5, `new Random()` sin argumentos se sembraba con `Environment.TickCount`, es decir con el reloj: quien sepa aproximadamente cuándo arrancó el proceso tiene unos pocos miles de semillas candidatas que probar. Las versiones modernas de .NET siembran el constructor sin argumentos con entropía del sistema, lo que elimina *este* problema pero no el anterior: el algoritmo sigue siendo reconstruible.

Lo correcto es un **CSPRNG** (*Cryptographically Secure PRNG*), diseñado precisamente para que su estado no se pueda deducir de sus salidas:

```csharp
// ✅ Token de sesión imprevisible
byte[] bytes = RandomNumberGenerator.GetBytes(32);
string token = Convert.ToBase64String(bytes);
```

El detalle que lo explica todo: `RandomNumberGenerator` **no acepta seed**. No es un olvido de la API ni una carencia, es el requisito al revés. Que nadie pueda reproducir la secuencia es exactamente lo que se le pide, así que ofrecer una forma de reproducirla sería el fallo. Cuando una API de aleatoriedad te deja fijar la semilla, te está diciendo para qué sirve; y cuando no te deja, también.

## La cuarta prima: el pepper

Existe un cuarto término de la misma familia semántica, el **pepper**: un secreto de servidor que se mezcla con la contraseña *además* de la sal, guardado fuera de la base de datos (variable de entorno, gestor de claves, KMS). Protege en el escenario más frecuente de todos, el que solo se filtra la base de datos: sin el pepper, los hashes robados no se pueden atacar ni por fuerza bruta.

Se confunde con la sal porque las dos se «añaden» a la contraseña, y son opuestas en las dos dimensiones que importan:

| | **Sal** | **Pepper** |
|---|---|---|
| ¿Secreto? | No, se guarda en claro con el hash | **Sí**, y nunca se guarda con el hash |
| ¿Único o compartido? | **Único** por contraseña | **Compartido** por todas las cuentas |
| Dónde vive | Dentro de la columna `PasswordHash` | Fuera de la base de datos |
| Si se pierde | No pasa nada: no se pierde, va con el hash | **Nadie puede volver a iniciar sesión** |

Esa última fila es la razón de que el pepper sea una decisión y no un valor por defecto: añade una dependencia crítica de operaciones y rotarlo obliga a rehashear en el siguiente login de cada cuenta.

## Cuándo NO usar cada cosa

- **No implementes la sal a mano.** Si estás generando bytes aleatorios, concatenándolos a la contraseña y decidiendo dónde guardarlos, estás reescribiendo lo que la KDF ya hace mejor. La única sal que deberías tocar es la que lees en un blob al depurar.
- **No siembres datos que sean configuración, ni credenciales que sobrevivan al primer día.** El seed es para filas sin las cuales el sistema no arranca: la cuenta de administración, un catálogo cerrado. Los datos de demostración no deben compartir mecanismo con las migraciones de producción, o acabarás con pedidos de prueba en la tienda real. Y si la cuenta sembrada usa una contraseña provisional, hay que forzar su cambio en el primer login: un seed no es un mecanismo de seguridad, es contenido inicial.
- **No uses una seed aleatoria fija para nada que deba ser imprevisible.** Tokens de sesión, códigos de recuperación, identificadores de invitación, contraseñas temporales: todo eso es `RandomNumberGenerator`. La regla operativa es sencilla: si el valor se va a guardar hasheado, es un secreto, y si es un secreto no sale de un PRNG con seed.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| Dos clientas con la misma contraseña tienen el mismo `PasswordHash` | Sal fija, compartida por todas las cuentas, o hash sin sal (SHA-256 a secas) |
| El seed reescribe la misma fila en cada arranque | Compara el hash almacenado con uno recién calculado; la sal nueva hace que nunca coincidan |
| Cada `migrations add` genera un `UpdateData` de `PasswordHash` | Un `HasData` con la contraseña hasheada en tiempo de diseño |
| Un token de sesión resulta predecible | Generado con `new Random()` en lugar de `RandomNumberGenerator` |
| Se busca la columna `Salt` en `Clientes` y no existe | La sal va dentro del blob de `PasswordHash`, con el algoritmo y los parámetros |
| Un hash está escrito literal en una migración | Es una contraseña publicada en el repositorio; debe venir de configuración |
| Se fija `new Random(42)` en los tests y los fallos de datos límite dejan de aparecer | La suite explora siempre el mismo camino: es un test determinista disfrazado de aleatorio |

## Buenas prácticas avanzadas

- **Sembrar una contraseña hasheada nunca es idempotente por comparación de valores.** La sal nueva de cada llamada garantiza un blob distinto, así que la condición «¿ha cambiado el hash?» es siempre verdadera y la fila se reescribe eternamente. Apoya la idempotencia en la clave natural —el correo— y si necesitas comprobar si la contraseña sembrada sigue vigente, usa `VerifyHashedPassword` contra el hash almacenado en lugar de comparar cadenas. La contraseña llega por configuración externa, y su ausencia debe romper el arranque.
- **Hay una cuarta prima, el pepper, y su coste real es operativo, no criptográfico.** Añade una defensa genuina para el caso de fuga solo de la base de datos, pero convierte un secreto en un punto único de fallo: perderlo equivale a perder todas las contraseñas, y rotarlo exige un mecanismo de rehash progresivo en el login. Decídelo con esa factura delante, no por completitud.
- **En los tests, seed nueva en cada ejecución… pero registrada en la salida.** Fijar siempre `new Random(42)` esconde los fallos que solo aparecen con otros datos; la aleatoriedad pura hace irreproducibles los que aparecen. El punto medio experto es generar una seed distinta por ejecución, imprimirla siempre en la salida del test, y cuando algo falle, repetir esa ejecución exacta pasándole la misma seed. Los frameworks de *property-based testing* traen esto de serie precisamente porque es el patrón correcto.
- **Trata el uso de `System.Random` como algo que se revisa, no como una elección de estilo.** Es indistinguible de `RandomNumberGenerator` en el resultado visible y radicalmente distinto en garantías, así que la revisión de código no lo pilla leyendo. Una regla de análisis estático que marque `new Random` en el código que genera tokens, códigos o identificadores lo caza antes de que exista.
- **Fija los parámetros del hasher explícitamente y deja que el blob haga la migración.** El coste por defecto depende de la versión del *runtime*, así que declararlo evita que una actualización cambie el comportamiento sin avisar. Como cada hash lleva sus propios parámetros dentro, subir el coste no invalida nada: los hashes antiguos siguen validando y se van rehasheando en el siguiente login de cada cuenta.

## Documentación oficial

- [`RandomNumberGenerator`](https://learn.microsoft.com/dotnet/api/system.security.cryptography.randomnumbergenerator) y [`System.Random`](https://learn.microsoft.com/dotnet/api/system.random) — léelas una al lado de la otra: la primera no expone ninguna forma de fijar una semilla, y la segunda advierte en su propia página de que no sirve para fines de seguridad. Esa asimetría es el resumen de media ficha.
- [Código fuente de `PasswordHasher`](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/Extensions.Core/src/PasswordHasher.cs) — el sitio donde comprobar que el desglose del blob de esta ficha es exacto: versión, función interna, iteraciones, longitud de la sal, sal y hash. Útil cuando necesites leer un blob a mano para depurar.
- [Datos iniciales en EF Core (`HasData`)](https://learn.microsoft.com/ef/core/modeling/data-seeding) — la referencia del sembrado de datos, con la distinción entre los datos que forman parte del modelo y los que se insertan al arrancar. Ve ahí antes de decidir dónde poner tu cuenta de administración.

## Recursos didácticos

- [Introduction to Randomness and Random Numbers](https://www.random.org/randomness/) y su [análisis visual](https://www.random.org/analysis/), ambos en random.org — el primero explica sin matemáticas por qué un PRNG es determinista por construcción; el segundo lo hace ver, comparando mapas de bits generados con un PRNG frente a los generados con ruido físico. Los patrones del PRNG se distinguen a simple vista, y es la forma más rápida de interiorizar que «parece aleatorio» y «es imprevisible» no son lo mismo.
- [Mersenne Twister — limitaciones criptográficas (Wikipedia)](https://en.wikipedia.org/wiki/Mersenne_Twister) — el caso concreto y numérico: observando 624 salidas consecutivas se reconstruye el estado interno y se predice todo lo siguiente. Es el argumento del apartado de la seed aleatoria con cifras.

---

*En resumen: la sal es aleatoriedad única que se mezcla con cada contraseña antes de hashear; el seed de datos es lo que plantas en una base de datos vacía para que arranque; y la seed aleatoria hace repetible a un generador — tres «semillas» que solo comparten el nombre.*
