# Microsoft.Extensions.Options

## ¿Qué es?

`Microsoft.Extensions.Options` es la librería de .NET que implementa el **Options pattern**: coger una sección de tu configuración (por ejemplo, de `appsettings.json`) y exponerla al resto de la aplicación como un objeto C# fuertemente tipado, inyectable por DI, en lugar de leer claves sueltas por su nombre.

## ¿Por qué existe?

En una aplicación .NET la configuración vive en `IConfiguration`: un diccionario de claves y valores en forma de texto (`"Smtp:Host"`, `"Smtp:Port"`...). Puedes leerlo directamente, pero acabas con *magic strings* repartidos por todo el código, sin tipado, sin autocompletado y sin ninguna garantía de que la clave exista o de que el valor sea convertible:

```csharp
// Sin Options: frágil y repetido en cada sitio donde haga falta
var host = configuration["Smtp:Host"];
var port = int.Parse(configuration["Smtp:Port"]); // revienta en ejecución si falta o no es número
```

El Options pattern resuelve esto: defines una clase POCO con las propiedades que esperas, le dices a .NET a qué sección de configuración corresponde, y a partir de ahí pides ese objeto ya montado y tipado allí donde lo necesites. Un único punto de *binding*, tipado real y validación opcional.

> Si vienes de Java con Spring, el Options pattern es el equivalente a `@ConfigurationProperties`: mapear una sección del fichero de configuración a un bean tipado.

## ¿Cuándo y para qué se usa?

Cada vez que un componente necesita parámetros externos que no quieres tener hardcodeados: los datos del servidor SMTP para enviar correos, la URL y la API key de un servicio de pagos, los márgenes de una política de precios de una tienda online, el tamaño de página por defecto de un listado de productos, o los *feature flags* que encienden y apagan funcionalidades.

En todos esos casos creas una clase de *settings* (`SmtpSettings`, `PricingOptions`...), la enlazas a su sección de configuración una sola vez al arrancar, y la inyectas en los servicios que la usan. Así el mismo binario funciona en local, en staging y en producción cambiando solo el `appsettings.json` (o las variables de entorno), sin tocar código.

## Lo mínimo que necesitas saber

**1. La clase POCO de settings**

Una clase normal, con propiedades públicas y, por convención, el nombre de su sección como constante:

```csharp
public class SmtpSettings
{
    public const string SectionName = "Smtp";

    public string Host { get; set; } = string.Empty;
    public int Port { get; set; }
    public bool UseSsl { get; set; }
}
```

**2. La sección en `appsettings.json`**

```json
{
  "Smtp": {
    "Host": "smtp.example.com",
    "Port": 587,
    "UseSsl": true
  }
}
```

**3. Registrar el binding en el arranque (`Program.cs`)**

`Configure<T>` vincula la sección con la clase y la deja disponible en el contenedor de DI:

```csharp
builder.Services.Configure<SmtpSettings>(
    builder.Configuration.GetSection(SmtpSettings.SectionName));
```

**4. Consumir la configuración: `IOptions<T>`**

Inyectas `IOptions<T>` y accedes al objeto real con `.Value`:

```csharp
public class EmailSender
{
    private readonly SmtpSettings _settings;

    public EmailSender(IOptions<SmtpSettings> options)
    {
        _settings = options.Value; // el objeto ya bindeado y tipado
    }
}
```

**5. `IOptions<T>` vs `IOptionsSnapshot<T>` vs `IOptionsMonitor<T>`**

Las tres interfaces dan acceso a los mismos datos, pero se diferencian en cuándo se calculan y si detectan cambios en la configuración en caliente:

- **`IOptions<T>`** — *singleton*. Se resuelve **una sola vez** para toda la vida de la aplicación y se cachea. No refleja cambios en `appsettings.json` posteriores al arranque. Es la opción por defecto y la más simple.
- **`IOptionsSnapshot<T>`** — *scoped*. Se recalcula **una vez por petición** (scope). En una API web, cada request ve el valor más reciente. Ideal para servicios registrados como *scoped* que deben respetar cambios de configuración sin reiniciar. No se puede inyectar en un *singleton*.
- **`IOptionsMonitor<T>`** — *singleton* con **recarga en caliente**. Expone `.CurrentValue` (siempre actualizado) y `OnChange(...)` para reaccionar a cambios. Es la única opción válida cuando un *singleton* necesita ver configuración que cambia en tiempo de ejecución.

Regla práctica: en servicios *scoped* de una web usa `IOptionsSnapshot<T>`; en *singletons* que necesiten reaccionar a cambios usa `IOptionsMonitor<T>`; para todo lo demás, `IOptions<T>`.

**6. Validación**

Puedes exigir que la configuración cumpla reglas antes de dejar que se use:

```csharp
public class SmtpSettings
{
    [Required] public string Host { get; set; } = string.Empty;
    [Range(1, 65535)] public int Port { get; set; }
}
```

```csharp
builder.Services.AddOptions<SmtpSettings>()
    .Bind(builder.Configuration.GetSection(SmtpSettings.SectionName))
    .ValidateDataAnnotations()                       // valida los [Required], [Range]...
    .Validate(s => s.Port != 25, "El puerto 25 no está permitido") // regla propia
    .ValidateOnStart();                              // falla al arrancar, no en la primera request
```

**7. Named options**

Cuando necesitas varias configuraciones del mismo tipo (por ejemplo, dos proveedores de correo), las registras con nombre y las recuperas por nombre con `IOptionsSnapshot<T>` o `IOptionsMonitor<T>`:

```csharp
builder.Services.Configure<SmtpSettings>("Primary", builder.Configuration.GetSection("Smtp:Primary"));
builder.Services.Configure<SmtpSettings>("Backup",  builder.Configuration.GetSection("Smtp:Backup"));

// Consumo:
var primary = monitor.Get("Primary");
```

## Lo que NO hace

- **No es un sistema de configuración por sí solo** — no lee ficheros ni variables de entorno. Eso lo hace `IConfiguration` y sus *providers* (`appsettings.json`, variables de entorno, *user secrets*, Azure Key Vault...). Options solo coge lo que `IConfiguration` ya cargó y lo tipa.
- **No recarga automáticamente** — con `IOptions<T>` el valor se congela al arrancar. Solo `IOptionsSnapshot<T>` (por petición) e `IOptionsMonitor<T>` (en caliente) reflejan cambios posteriores.
- **No valida por defecto** — si no llamas a `ValidateDataAnnotations`, `Validate` o `ValidateOnStart`, una sección incompleta o con un tipo incorrecto pasará sin avisar y fallará más tarde, en tiempo de ejecución.
- **No inventa valores** — si una clave no está en la configuración, la propiedad se queda con su valor por defecto de C# (`null`, `0`, `false`), no con un error, salvo que lo valides.

## Buenas prácticas avanzadas

- **`ValidateOnStart()` para fallar al arrancar, no en la primera petición.** Sin él, la validación se ejecuta de forma perezosa la primera vez que alguien pide el `IOptions<T>`, así que una configuración mal puesta no explota hasta que un usuario real toca el endpoint afectado. Con `ValidateOnStart()` el proceso ni siquiera termina de arrancar si la configuración es inválida: el fallo aparece en el arranque (y en el despliegue), donde lo quieres ver.
- **Elige la interfaz por el ciclo de vida del consumidor, no por costumbre.** `IOptions<T>` es *singleton* y cachea el valor una única vez; `IOptionsSnapshot<T>` es *scoped* y no puede inyectarse en un *singleton* (lanza excepción). Si un servicio *singleton* inyecta `IOptions<T>` esperando ver cambios de `appsettings.json` en caliente, nunca los verá: para eso necesita `IOptionsMonitor<T>` y su `.CurrentValue`. Este es el error sutil más común del patrón.
- **Reserva `IOptionsSnapshot<T>` para lo *scoped* y `IOptionsMonitor<T>` para lo *singleton*.** `IOptionsSnapshot<T>` recalcula el objeto en cada petición, lo que añade un coste pequeño pero real; no lo uses si no necesitas recarga. Y no uses `IOptionsMonitor<T>` "por si acaso" en servicios *scoped*: el *snapshot* ya te da el valor fresco por request de forma más barata y directa.
- **Registra la sección con un nombre constante (`const string SectionName`) en la propia clase.** Evita que el nombre de la sección viva como *magic string* duplicado entre el `appsettings.json`, el `Program.cs` y los tests. Un solo sitio de verdad reduce los errores de tecleo que, además, no da la compilación.
- **Separa los settings por *bounded concern*, no un mega-objeto de configuración.** Una clase `SmtpSettings`, otra `PricingOptions`, otra para el cliente de pagos... Cada servicio inyecta solo lo suyo y no arrastra dependencia sobre configuración que no le incumbe; además, la validación queda acotada a cada área.
- **Recuerda que `IOptions<T>` es un *singleton* cacheado.** No es un "lector de configuración en vivo": se resuelve una vez y devuelve siempre el mismo objeto. Si mutas `options.Value` en tiempo de ejecución, ese cambio es global y compartido por toda la app (normalmente no es lo que quieres). Trata los settings como inmutables una vez bindeados.

## Recursos didácticos

- [Options pattern in .NET](https://learn.microsoft.com/dotnet/core/extensions/options) — la guía oficial de Microsoft Learn, con el detalle de binding, validación y named options.
- [Options pattern in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/options) — la variante centrada en web, con las diferencias entre `IOptions`, `IOptionsSnapshot` e `IOptionsMonitor` bien explicadas.

---

*En resumen: Microsoft.Extensions.Options convierte una sección de tu configuración en un objeto C# tipado, validado e inyectable — dejas de leer claves sueltas por su nombre y empiezas a pedir settings de verdad.*
