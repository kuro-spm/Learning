# Cifrado en reposo de credenciales

## ¿Qué es?

Guardar en tu base de datos las credenciales de terceros **cifradas en lugar de en claro**, de forma que solo se conviertan en texto legible en el instante exacto en que hay que usarlas. La aplicación tiene la capacidad de descifrarlas; la base de datos, por sí sola, no.

## ¿Por qué existe?

Porque hay secretos que tu aplicación no elige: los introduce alguien. En el panel de administración de la tienda, una persona administradora pega la clave de API de la transportista o de la pasarela de pagos `api.pasarela.ejemplo.com`, y esa clave tiene que sobrevivir al reinicio del proceso para poder usarla en cada envío. Es decir, acaba en una columna de `TiendaDB`. Si esa columna guarda el valor en claro, la credencial se escapa por cuatro vías que no tienen nada que ver entre sí:

- **Un *dump* de la base de datos.** El fichero `.bak` o `.sql` que alguien genera para depurar un problema y comparte por chat.
- **Un backup.** Vive más tiempo y en más sitios que la base de datos, a menudo en un bucket con permisos más laxos.
- **Una inyección SQL en cualquier otro módulo.** Un `SELECT` conseguido en el buscador de productos lee también la tabla de proveedores.
- **Los logs.** Una consulta lenta registrada, una excepción de Entity Framework o un volcado de la entidad completa escriben el valor en un fichero de texto que se replica al sistema de observabilidad.

Cifrar la columna corta las cuatro de golpe: lo que se filtra deja de ser una clave y pasa a ser una cadena inútil. Pero aparece a cambio un problema nuevo, y es el que ocupa la mitad de esta ficha: para descifrar hace falta una **clave maestra**, que es en sí misma un secreto — y de los importantes, porque con ella se descifran *todas* las credenciales guardadas.

> Cifrar las credenciales es meterlas en una caja fuerte, y la clave maestra es la llave de esa caja. Si la escribes en el código (*hardcoded*), es como pegarla con celo encima de la puerta: el candado ya no protege de nada, porque quien lee el repositorio ya tiene la llave. Por eso la clave maestra vive **fuera del código**, igual que cualquier otro secreto — ver [Por qué los secretos no van a git](Por-Que-Los-Secretos-No-Van-A-Git.md).

## ¿Cuándo y para qué se usa?

Cuando se cumplen tres condiciones a la vez: el dato es **una credencial**, tu aplicación necesita **recuperarlo en claro** más tarde, y no puedes delegar su almacenamiento a un servicio externo. Casos típicos: claves de API de proveedores, tokens de larga duración de integraciones, contraseñas de servidores SMTP o FTP que configura cada cliente.

El ejemplo que recorre la ficha es una API en .NET (`src/Tienda.Api`) que guarda la clave de la pasarela de pagos en la tabla `Proveedores` de `TiendaDB`, con una clave maestra en configuración bajo `Cifrado:MasterKey`.

---

## Cifrar no es hashear

Es la distinción de la que depende todo lo demás, y confundirla lleva a implementar la mitad de un sistema que nunca podrá funcionar.

| | Hashing | Cifrado |
|---|---|---|
| Dirección | Solo ida (irreversible) | Ida y vuelta (reversible) |
| Para qué | Comprobar si un valor coincide con el guardado | Recuperar el valor original para usarlo |
| Ejemplo | PBKDF2, Argon2, `PasswordHasher` | AES-256-GCM |

Dos casos concretos de la tienda, para que no quede duda:

- **La contraseña de una clienta que se registra** → *hash*. Al iniciar sesión vuelves a hashear lo que escribe y comparas. Nunca necesitas el texto original, y no poder recuperarlo es precisamente la propiedad que quieres. Lo cubre la colección de [algoritmos de hash](../algoritmos-de-hash/README.md), en particular las [funciones de derivación de claves](../algoritmos-de-hash/Funciones-De-Derivacion-De-Claves.md).
- **La clave de API de la pasarela de pagos** → *cifrado*. Tienes que enviarla literalmente en la cabecera `api-key` de cada petición. Un hash no sirve: la pasarela espera la clave, no su huella.

## Simétrico frente a asimétrico

El cifrado **simétrico** usa la misma clave para cifrar y para descifrar. Es rapidísimo, cabe en cincuenta líneas y el algoritmo estándar (AES) está acelerado por hardware en cualquier CPU de los últimos quince años. Su única debilidad es logística: quien puede cifrar puede descifrar, así que hay que proteger la clave en todos los sitios donde se use.

El **asimétrico** (RSA, curvas elípticas) usa un par de claves: la pública cifra, la privada descifra. Eso permite un reparto de papeles interesante —un servicio puede guardar datos cifrados sin poder leerlos, porque solo tiene la pública— pero es órdenes de magnitud más lento y tiene un límite de tamaño por operación (RSA-2048 cifra unos 190 bytes como máximo). Aquí el simétrico es la respuesta por un motivo simple: **el mismo proceso que guarda la credencial es el que luego la usa**. No hay dos partes con papeles distintos que justifiquen dos claves.

## AES-256-GCM, pieza por pieza

Es la opción por defecto razonable hoy. Son cuatro conceptos, y conviene tenerlos claros antes de ver una línea de código:

- **Clave simétrica de 32 bytes** (256 bits). Es la clave maestra. No es una contraseña ni una frase: son 32 bytes aleatorios.
- **Nonce de 12 bytes.** *Nonce* viene de *number used once*: un valor que acompaña a cada operación de cifrado y que **no se repite jamás con la misma clave**. No es secreto —se guarda junto al dato cifrado— pero tiene que ser único.
- **Tag de 16 bytes.** Un sello de autenticación que se calcula al cifrar y se verifica al descifrar.
- **Cifrado autenticado.** Es lo que aporta el modo GCM: no solo hace ilegible el dato (confidencialidad), también **detecta si alguien lo ha modificado** (integridad). Si un byte del texto cifrado cambia, el tag no cuadra y el descifrado falla con una excepción en lugar de devolver basura en silencio. Esa es la diferencia con modos antiguos como CBC, donde un atacante puede alterar el texto cifrado y tu código descifra tranquilamente un valor distinto sin enterarse.

## Por qué el nonce no se puede reutilizar (con números)

Esta es la regla que más se incumple, casi siempre con buena intención: alguien decide derivar el nonce del identificador de la fila «para no tener que guardarlo». Veamos qué se rompe exactamente. GCM cifra generando un *keystream* a partir de la clave y el nonce, y haciéndole XOR con el texto en claro:

```
C1 = P1 ⊕ keystream(clave, nonce)
C2 = P2 ⊕ keystream(clave, nonce)     ← mismo nonce → mismo keystream
─────────────────────────────────────
C1 ⊕ C2 = P1 ⊕ P2                     ← el keystream se cancela
```

Con dos textos cifrados bajo el mismo par (clave, nonce), cualquiera que los tenga obtiene **el XOR de los dos textos en claro** sin conocer la clave. Y eso, aquí, es devastador por lo concreto:

- Las claves de API tienen prefijos fijos. Si dos proveedores usan claves que empiezan por `sk_live_`, esos 8 bytes del XOR salen a cero, lo que confirma el formato y filtra la longitud.
- Y sobre todo: **quien conoce uno de los dos textos en claro obtiene el otro completo**, porque `P2 = C1 ⊕ C2 ⊕ P1`. Alguien con acceso al panel que introduce su propia clave de pruebas y luego lee la columna cifrada de otra fila recupera la credencial ajena byte a byte.

Hay un segundo daño, menos conocido y peor: la reutilización de nonce permite recuperar la **subclave de autenticación** de GCM (el llamado *forbidden attack*), y con ella **falsificar tags válidos** para textos cifrados inventados. El cifrado autenticado deja de autenticar nada.

La solución no es ingeniosa, es aritmética: **12 bytes aleatorios de un generador criptográfico en cada operación**. Un nonce de 96 bits da 2⁹⁶ valores posibles; por la paradoja del cumpleaños, la probabilidad de que se repita alguno tras 2³² cifrados (4.300 millones de operaciones) es de unos 2⁻³³, es decir **una entre 8.600 millones**. Guardar el nonce cuesta 12 bytes por fila; ahorrárselos cuesta el sistema entero.

## Cifrar: el payload y el método

Al cifrar se producen tres cosas —nonce, tag y texto cifrado— y las tres son necesarias para descifrar. La forma sensata de guardarlas es concatenarlas como `nonce ‖ tag ‖ textoCifrado` y codificar el conjunto en base64, para que quepa en una sola columna de texto. Van juntas porque van juntas toda la vida: un tag sin su texto cifrado no vale nada, y un texto cifrado sin su nonce es irrecuperable; separarlos en tres columnas solo multiplica las ocasiones de desincronizarlos en una migración.

```csharp
public sealed class CifradorDeCredenciales
{
    private const int TamanoNonce = 12;   // 96 bits, el tamaño recomendado para GCM
    private const int TamanoTag = 16;     // 128 bits, el máximo que admite GCM
    private readonly byte[] _clave;       // 32 bytes = AES-256

    public string Cifrar(string textoEnClaro)
    {
        var enClaro = Encoding.UTF8.GetBytes(textoEnClaro);
        var payload = new byte[TamanoNonce + TamanoTag + enClaro.Length];
        var nonce   = payload.AsSpan(0, TamanoNonce);
        var tag     = payload.AsSpan(TamanoNonce, TamanoTag);
        var cifrado = payload.AsSpan(TamanoNonce + TamanoTag);

        RandomNumberGenerator.Fill(nonce);        // nonce nuevo en CADA llamada
        using var aes = new AesGcm(_clave, TamanoTag);
        aes.Encrypt(nonce, enClaro, cifrado, tag);

        return Convert.ToBase64String(payload);
    }
}
```

Se escribe directamente sobre el buffer final con `AsSpan`, así que no quedan copias intermedias del texto en claro dando vueltas por el montón. Para la clave de ejemplo `sk_live_a1b2c3d4e5f60718` (24 bytes) el resultado son 52 bytes, o 72 caracteres en base64:

```
9Qr3TmXsPa7kLdN2vBhZ0eWyC4uJmR6tGnA8sKfD1xVpO5iEqM3bYzU7cHlW9jSrT2nFdX==
```

Detalle que sorprende la primera vez: **cifrar dos veces el mismo valor da dos cadenas distintas**, porque el nonce cambia. Es correcto y deseable —impide deducir que dos filas guardan la misma clave— pero implica que la columna cifrada no se puede comparar ni indexar por igualdad.

## Descifrar, y qué pasa si alguien manipula el dato

El descifrado deshace el troceado y deja que GCM verifique el tag:

```csharp
public string Descifrar(string payloadBase64)
{
    var payload = Convert.FromBase64String(payloadBase64);
    if (payload.Length < TamanoNonce + TamanoTag)
        throw new CryptographicException("El payload cifrado tiene un formato inválido.");

    var nonce   = payload.AsSpan(0, TamanoNonce);
    var tag     = payload.AsSpan(TamanoNonce, TamanoTag);
    var cifrado = payload.AsSpan(TamanoNonce + TamanoTag);
    var enClaro = new byte[cifrado.Length];

    using var aes = new AesGcm(_clave, TamanoTag);
    aes.Decrypt(nonce, cifrado, tag, enClaro);   // verifica el tag; lanza si no cuadra
    return Encoding.UTF8.GetString(enClaro);
}
```

Si alguien cambia un solo carácter del base64 guardado en `TiendaDB`, `Decrypt` no devuelve un valor incorrecto: aborta.

```
System.Security.Cryptography.AuthenticationTagMismatchException:
   The computed authentication tag did not match the input authentication tag.
```

Esa excepción (que hereda de `CryptographicException`, por si capturas el tipo base) **es exactamente lo que se quiere**. La alternativa —devolver una cadena corrupta— haría que tu API enviase a la pasarela una credencial alterada, y el fallo aparecería tres capas más arriba como un `401` incomprensible. Así falla en el sitio donde está el problema. Nunca captures esta excepción para devolver `null` o el payload tal cual: el fallo ruidoso **es** la funcionalidad.

## Fail-fast al arrancar

Una clave maestra ausente o del tamaño equivocado no debe descubrirse en la primera petición que intente cifrar, sino al arrancar. Se valida en el constructor:

```csharp
public CifradorDeCredenciales(IConfiguration configuracion)
{
    var enBase64 = configuracion["Cifrado:MasterKey"]
        ?? throw new InvalidOperationException("Falta la configuración 'Cifrado:MasterKey'. " +
            "Genérala con 'openssl rand -base64 32' y guárdala en user-secrets.");

    var clave = Convert.FromBase64String(enBase64);   // FormatException si no es base64 válido
    if (clave.Length != 32)
        throw new InvalidOperationException(
            $"'Cifrado:MasterKey' debe medir 32 bytes (AES-256) y mide {clave.Length}. " +
            "Regenérala con 'openssl rand -base64 32'.");

    _clave = clave;
}
```

Como un singleton no se construye hasta que alguien lo pide, hay que forzar su resolución en el arranque para que el error salga antes de aceptar tráfico:

```csharp
builder.Services.AddSingleton<CifradorDeCredenciales>();
var app = builder.Build();
app.Services.GetRequiredService<CifradorDeCredenciales>();   // valida ahora, no en la primera petición
```

Con una clave generada por error con `openssl rand -base64 24`, la aplicación no arranca:

```
Unhandled exception. System.InvalidOperationException: 'Cifrado:MasterKey' debe medir
32 bytes (AES-256) y mide 24. Regenérala con 'openssl rand -base64 32'.
```

Con eso, el uso queda trivial en los dos extremos: `proveedor.ClaveApiCifrada = cifrador.Cifrar(peticion.ClaveApi)` al guardar lo que introduce la persona administradora, y `client.DefaultRequestHeaders.Add("api-key", cifrador.Descifrar(proveedor.ClaveApiCifrada))` al llamar a la pasarela.

## Generar y guardar la clave maestra

Los 32 bytes aleatorios se generan con cualquier generador criptográfico y se manejan en base64 para poder escribirlos en configuración:

```bash
openssl rand -base64 32
# -> i2qYrfRK6OekOcjJ8OBsRPKKMBG2CE7RkIymnl/UCwo=
```

Sin salir de .NET, el equivalente es `Convert.ToBase64String(RandomNumberGenerator.GetBytes(32))`. En desarrollo la clave se guarda en user-secrets, que la deja fuera del árbol del proyecto y por tanto fuera de git:

```bash
dotnet user-secrets set "Cifrado:MasterKey" "i2qYrfRK6OekOcjJ8OBsRPKKMBG2CE7RkIymnl/UCwo=" --project src/Tienda.Api
```

La clave de desarrollo es de usar y tirar: si se pierde, se regenera y se vuelven a introducir las credenciales de prueba. El mecanismo completo —el `UserSecretsId`, la precedencia de fuentes de configuración— está en [User-secrets en .NET](User-Secrets-En-Dotnet.md). En producción la provee la plataforma como variable de entorno o, mejor, un gestor de secretos.

## `IDataProtector`, la alternativa que hace el trabajo por ti

ASP.NET Core trae una API de protección de datos que ya resuelve la generación, el almacenamiento y la rotación de claves:

```csharp
public sealed class ClavesDeProveedor(IDataProtectionProvider proveedor)
{
    private readonly IDataProtector _protector =
        proveedor.CreateProtector("Tienda.Api.ClavesDeProveedor.v1");

    public string Cifrar(string clave) => _protector.Protect(clave);
    public string Descifrar(string cifrada) => _protector.Unprotect(cifrada);
}
```

La cadena que se pasa a `CreateProtector` es el **propósito**, y aporta algo valioso gratis: dos protectores con propósitos distintos no pueden descifrar lo del otro, aunque compartan el llavero. Un payload de claves de proveedor no se puede reinyectar donde se esperaba un token de recuperación de contraseña. `Unprotect` sobre un dato manipulado lanza `CryptographicException: The payload was invalid.`, el equivalente al tag que no cuadra.

| | `AesGcm` a mano | `IDataProtector` |
|---|---|---|
| Control del formato | Total: tú decides el payload | Opaco, es un detalle de implementación |
| Curva de aprendizaje | Hay que entender nonce, tag y sus reglas | Dos métodos, `Protect` y `Unprotect` |
| Rotación de claves | La implementas tú (sección siguiente) | Automática, con llavero y expiración de 90 días |
| Descifrar fuera de la app | Sí: cualquier lenguaje con AES-GCM lo lee | No en la práctica: dependes del llavero de ASP.NET Core |
| Riesgo de usarlo mal | Real: un nonce fijo lo arruina en silencio | Bajo: no hay parámetros que equivocar |

El juicio es sencillo: **hacerlo a mano da más control, `IDataProtector` es más difícil de usar mal**, y ambas son válidas. Si el dato cifrado solo lo va a leer esta aplicación, `IDataProtector` es la opción de menos riesgo. Si otro proceso, un script de migración o un servicio en otro lenguaje tienen que descifrarlo, o si necesitas saber exactamente qué bytes hay en la columna, AES-GCM a mano.

## Rotación: versiona la clave dentro del payload

Una clave maestra se rota porque ha caducado por política, porque alguien que la conocía ha dejado el equipo o porque se ha filtrado. Rotar significa **descifrar todo con la vieja y recifrarlo con la nueva**, lo que exige tener las dos disponibles a la vez. Y si el payload no dice con qué clave se cifró, hay que adivinarlo probando —un truco frágil que además no sabe distinguir «clave equivocada» de «dato corrupto»—. La solución es un prefijo de versión en el propio valor guardado:

```
v2:9Qr3TmXsPa7kLdN2vBhZ0eWyC4uJmR6tGnA8sKfD1xVpO5iEqM3bYzU7cHlW9jSrT2nFdX==
└┬┘└──────────────────────── nonce ‖ tag ‖ textoCifrado, en base64 ─────────┘
 └─ qué clave maestra hay que usar
```

El servicio pasa a tener un diccionario de claves y una versión «actual» con la que escribe:

```csharp
private const string VersionActual = "v2";
private readonly Dictionary<string, byte[]> _claves;   // "v1" y "v2", de configuración

public string Cifrar(string textoEnClaro)
    => $"{VersionActual}:{CifrarCon(_claves[VersionActual], textoEnClaro)}";

public string Descifrar(string payload)
{
    var version = payload[..payload.IndexOf(':')];
    if (!_claves.TryGetValue(version, out var clave))
        throw new InvalidOperationException($"No hay clave maestra configurada para la versión '{version}'.");

    return DescifrarCon(clave, payload[(version.Length + 1)..]);
}
```

Con eso, el recifrado es un bucle que se puede ejecutar con la aplicación en marcha, sin parada ni ventana de mantenimiento:

```csharp
await foreach (var proveedor in db.Proveedores
    .Where(p => p.ClaveApiCifrada != null && !p.ClaveApiCifrada.StartsWith("v2:"))
    .AsAsyncEnumerable())
{
    var enClaro = cifrador.Descifrar(proveedor.ClaveApiCifrada!);  // entra por v1
    proveedor.ClaveApiCifrada = cifrador.Cifrar(enClaro);          // sale como v2
}
await db.SaveChangesAsync();
```

El filtro `!StartsWith("v2:")` hace el proceso **idempotente y reanudable**: si se corta a mitad, la siguiente ejecución retoma solo las filas que faltan. Cuando el `WHERE` no devuelve nada, `v1` puede retirarse de la configuración. Dos consecuencias operativas más, del mismo asunto:

- **Si se pierde la clave maestra de producción, las credenciales cifradas son ilegibles para siempre.** No hay recuperación ni soporte que la restaure: es la propiedad que hace que el cifrado funcione. Un backup de `TiendaDB` sin su clave maestra es papel mojado, así que el procedimiento de recuperación tiene que incluir las dos cosas.
- **Todas las réplicas necesitan la misma clave, desde un origen compartido.** En cuanto hay dos instancias tras un balanceador, una clave por máquina significa que lo que cifró la instancia A no lo puede descifrar la B. El síntoma es intermitente y desconcertante: funciona la mitad de las veces.

## Las capas de cifrado no son la misma cosa

Aquí está el malentendido más caro de este tema: «ya tenemos TDE activado en la base de datos, esto no hace falta». Son protecciones distintas contra amenazas distintas.

| Capa | De qué protege | De qué NO protege |
|---|---|---|
| **Cifrado en la aplicación** (esta ficha) | Dump, backup, log de consultas, inyección SQL en otro módulo, acceso administrativo a la base de datos | Compromiso del proceso de la API, que tiene la clave |
| **TDE** (cifrado transparente) | Robo del fichero de datos o del backup: sin la clave del motor no se pueden montar | Cualquiera con permiso de `SELECT`. El motor descifra al leer, así que el valor sale en claro en la consulta |
| **Always Encrypted** | El motor nunca ve el valor en claro; cifra y descifra el driver, en el cliente | Requiere que el cliente tenga la clave, y limita mucho las consultas (sin `LIKE` ni `ORDER BY` sobre columnas con cifrado aleatorizado) |
| **Cifrado de disco** (BitLocker, LUKS) | Robo físico del disco, retirada de hardware sin borrar | Todo lo demás: con la máquina arrancada, el volumen está montado y descifrado |

La conclusión es que **TDE y el cifrado de disco no sustituyen a esta ficha**, porque los vectores de la sección «¿Por qué existe?» —un `SELECT` de más, un dump compartido, un log— ocurren con la base de datos arrancada y funcionando, que es justo cuando esas dos capas están descifrando de forma transparente.

## Alcance honesto de la protección

Cifrar en reposo protege frente a **fuga de la base de datos, backups, logs y accesos casuales**, que es la inmensa mayoría de incidentes reales. **No** protege frente a un compromiso total del proceso en ejecución: quien controle el proceso puede leer en memoria tanto la clave maestra (que está en la configuración cargada) como el secreto ya descifrado. Aun así merece mucho la pena: sube el listón enormemente por un coste bajo. Un *vault* gestionado lo sube todavía más.

## Cuándo NO usar esta técnica

- **Si no necesitas recuperar el valor.** Contraseñas de personas, tokens que solo hay que verificar: usa un hash. Cifrar algo que podrías hashear añade una clave maestra que custodiar sin ninguna ganancia.
- **Si la plataforma te da un vault gestionado al que puedes delegar el almacenamiento entero.** Azure Key Vault, AWS Secrets Manager o HashiCorp Vault guardan el secreto ellos, con auditoría de accesos, rotación y permisos por identidad. Si la credencial cabe ahí, guardar en tu base de datos una versión cifrada es duplicar la superficie y la responsabilidad.
- **Si el dato no es una credencial.** Cifrar campos «por si acaso» (un nombre, una dirección) los vuelve imposibles de buscar, ordenar y filtrar, y suele acabar en un descifrado masivo en memoria para poder hacer un `WHERE`. Los datos personales tienen sus propias herramientas —minimización, seudonimización, control de acceso— y esta no es la primera.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| Dos filas con la misma credencial tienen el mismo texto cifrado | Nonce fijo o derivado del identificador de la fila: se filtra el XOR de los textos en claro y se pueden falsificar tags |
| La clave maestra aparece en un *pull request* | Está en `appsettings.json`, que va a git; tiene que ir a user-secrets o al entorno |
| Un dato cifrado alterado descifra sin error y produce un `401` en la pasarela | Se usa AES-CBC sin MAC: no hay integridad, y encima abre la puerta a un *padding oracle* |
| El código captura `CryptographicException` y devuelve `null` | El tag se verifica pero no se actúa: la manipulación se convierte en un fallo silencioso |
| `String or binary data would be truncated in table 'TiendaDB.dbo.Proveedores', column 'ClaveApiCifrada'` | La columna es demasiado corta. Una clave de 40 caracteres ocupa 92 caracteres en base64 (12 + 16 + 40 bytes); dimensiona con margen |
| `CryptographicException: Specified key is not a valid size for this algorithm.` | La clave viene de `Encoding.UTF8.GetBytes("una-frase")`, que no mide 32 bytes |
| Todo funciona, pero la clave es `Encoding.UTF8.GetBytes("clave-maestra-de-la-tienda-2026!")` | Son 32 bytes exactos, así que AES los acepta — con la entropía de una frase adivinable en lugar de 256 bits. El peor caso: falla en silencio |

## Buenas prácticas avanzadas

- **Ata el payload a su ubicación con datos asociados (AAD).** `AesGcm.Encrypt` acepta un parámetro `associatedData` que no se cifra pero sí entra en el cálculo del tag. Pasando ahí el identificador del proveedor, un payload copiado de la fila de la transportista a la fila de la pasarela deja de descifrar. Sin AAD, el cifrado autenticado garantiza que el valor no se ha modificado, pero no que esté en la fila donde se cifró — y mover filas es algo que cualquier inyección SQL de escritura permite.
- **Versiona el payload desde el primer día, aunque solo tengas una clave.** El prefijo `v1:` cuesta tres bytes y convierte la rotación en un bucle. Añadirlo después obliga a mantener para siempre un caso especial «sin prefijo significa v1» y a hacer un *backfill* sobre datos de producción. Es la decisión de diseño más barata de la ficha y la que más agradece el yo del futuro.
- **Que el valor descifrado no cruce la frontera del controlador.** Descífralo lo más tarde posible, úsalo y déjalo ir; nunca lo pongas en un DTO, en una respuesta JSON ni en un log. Para el panel de administración, expón `configurada: true` y los últimos cuatro caracteres: suficiente para que una persona reconozca qué clave puso, inútil para quien lea la respuesta.
- **Asume que la columna cifrada no se puede consultar.** No hay `WHERE ClaveApiCifrada = @valor`, ni `DISTINCT`, ni detección de duplicados: cada cifrado produce bytes distintos. Si de verdad necesitas buscar por el valor, la técnica es un **índice ciego** —guardar además un HMAC determinista del valor con una clave *distinta* de la de cifrado— entendiendo lo que concede: ese índice revela qué filas comparten el mismo valor.
- **Ensaya la restauración completa, no solo el backup.** Restaura una copia de `TiendaDB` en un entorno limpio y comprueba que las credenciales se descifran. Es el ejercicio que destapa que la clave maestra la conoce una sola persona, que no está en el gestor de secretos que creías o que el plan de recuperación ni la menciona. Descubrirlo un martes por la tarde cuesta una hora; descubrirlo durante un incidente cuesta las credenciales de todos los proveedores.
- **Trata la rotación como un evento con fecha, no como una capacidad teórica.** Tener el código de recifrado no es tener rotación: hay que haberla ejecutado al menos una vez en producción. Un `v1` que sigue en la configuración tres años después significa que la clave original nunca se ha cambiado, y que el número de personas que la han visto solo crece.

## Recursos didácticos

- [Documentación de `AesGcm` en .NET](https://learn.microsoft.com/dotnet/api/system.security.cryptography.aesgcm) — la referencia de la API, con los tamaños admitidos de nonce y tag y las sobrecargas con `associatedData`. Es donde comprobar por qué el constructor exige el tamaño de tag explícito a partir de .NET 8.
- [Protección de datos en ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/data-protection/introduction) — la introducción a `IDataProtector` y, más importante, la configuración del llavero: dónde se persisten las claves, cómo se comparten entre instancias y cómo funciona la expiración a los 90 días.
- [CryptoHack](https://cryptohack.org/) — retos interactivos de criptografía, con una sección dedicada a AES y a los modos de operación. Incluye el ataque de reutilización de nonce en GCM, donde recuperas el texto en claro con tus propias manos: leerlo convence a medias, hacerlo convence del todo.
- [Cryptopals](https://cryptopals.com/) — los sets clásicos, resolubles en cualquier lenguaje. El set 2 tiene el *padding oracle* de CBC, que es la mejor explicación práctica de por qué el cifrado sin autenticar no sirve.

---

*En resumen: cifra la credencial que necesitas recuperar con AES-256-GCM, un nonce aleatorio nuevo en cada operación y el nonce y el tag guardados junto al dato — y trata la clave maestra como lo que es: el secreto del que dependen todos los demás, versionado para poder rotarlo y respaldado para no perderlo.*
