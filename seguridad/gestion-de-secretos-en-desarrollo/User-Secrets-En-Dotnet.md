# User-secrets en .NET

## ¿Qué es?

User-secrets (*Secret Manager*) es el mecanismo oficial de .NET para guardar los secretos de **desarrollo** fuera del árbol del proyecto: en lugar de escribir la clave en `appsettings.json`, vive en un JSON dentro de tu perfil de usuario del sistema operativo, y la aplicación lo lee sola al arrancar.

Conviene decir desde el principio qué **no** es: no es un almacén cifrado. Ese fichero está en texto plano en tu carpeta de usuario. Su valor es *sacar el secreto del repositorio*, no protegerlo criptográficamente.

## ¿Por qué existe?

Toda API necesita valores que no pueden estar en el código: la clave de la pasarela de pagos, la contraseña de la base de datos. El sitio natural para configurar cosas en .NET es `appsettings.json`, pero ese fichero está versionado, así que en el momento en que escribes la clave real dentro, la clave está en git — y de git no se va sola. El detalle de por qué eso es tan difícil de deshacer lo cuenta [Por qué los secretos no van a git](Por-Que-Los-Secretos-No-Van-A-Git.md).

Las salidas ingenuas a ese problema fallan todas. Poner el valor y añadir el fichero al `.gitignore` rompe el proyecto para el resto del equipo. Comentar la línea antes de cada commit funciona hasta el primer despiste. Exportar variables de entorno a mano en cada terminal es incómodo y se pierde al reiniciar.

User-secrets resuelve exactamente ese hueco: un almacén **por proyecto y por usuario**, físicamente fuera del repositorio, que la aplicación encuentra sin que tengas que cambiar una línea de código.

> Si ya conoces los `.env` de Node o Python, piensa en user-secrets como un `.env` que no puede acabar en git porque no está en la carpeta del proyecto: no depende de que alguien se acuerde de ignorarlo.

## ¿Cuándo y para qué se usa?

En tu máquina, mientras desarrollas, para cualquier valor que no quieras que se lea en el repositorio. El ejemplo que recorre toda la ficha es una tienda online con la API en `src/Tienda.Api` y dos secretos:

- `Pasarela:ApiKey` — la clave de la pasarela de pagos (`api.pasarela.ejemplo.com`).
- `ConnectionStrings:TiendaDb` — la cadena de conexión a `TiendaDB`.

No se usa para nada más. No es un mecanismo de despliegue, no protege frente a alguien que ya tiene acceso a tu equipo y no sirve para compartir credenciales con el resto del equipo. En producción los mismos valores llegan por variables de entorno o por un *vault* gestionado, y esa transición es la mitad de esta ficha.

---

## El modelo de configuración de .NET: gana el último que habla

Este apartado explica por qué user-secrets «funciona solo», sin que escribas una línea para leerlo, y sin él no se entiende el resto.

`IConfiguration` no es un fichero: es una **pila de proveedores**. Cada proveedor aporta pares clave/valor, se apilan en un orden fijo y, cuando dos aportan la misma clave, **el último sobreescribe al anterior**. Este es el orden que monta `WebApplication.CreateBuilder(args)`, de menor a mayor prioridad:

| Orden | Proveedor | Dónde vive | ¿Va a git? |
|---|---|---|---|
| 1 | `appsettings.json` | En el proyecto | Sí |
| 2 | `appsettings.{Environment}.json` | En el proyecto | Sí |
| 3 | **User-secrets** (solo si el entorno es `Development`) | Perfil de usuario | No |
| 4 | Variables de entorno | Proceso / contenedor / systemd | No |
| 5 | Argumentos de línea de comandos | El comando que lanzas | No |

Los dos primeros están en el repositorio y los tres últimos no: la pila está ordenada, y no por casualidad, de «público y compartido» a «privado y local». Se ve mejor con la misma clave en dos sitios. `appsettings.json`, versionado, declara la forma de la configuración con el valor vacío:

```json
{
  "Pasarela": {
    "BaseUrl": "https://api.pasarela.ejemplo.com",
    "ApiKey": ""
  }
}
```

Y en user-secrets está el valor real. Un endpoint mínimo que enseñe de dónde sale cada cosa:

```csharp
app.MapGet("/config", (IConfiguration config) => new
{
    baseUrl = config["Pasarela:BaseUrl"],
    apiKey  = config["Pasarela:ApiKey"]
});
```

Con `dotnet run` en Development, la respuesta es:

```json
{"baseUrl":"https://api.pasarela.ejemplo.com","apiKey":"sk_test_4f8b21c9d05e"}
```

`BaseUrl` viene de `appsettings.json` porque nadie más la define; `ApiKey` viene de user-secrets, que está más arriba en la pila y gana. Y si lanzas la misma aplicación pisando el valor desde arriba:

```bash
Pasarela__ApiKey=sk_env_desde_entorno dotnet run
# → {"baseUrl":"...","apiKey":"sk_env_desde_entorno"}

dotnet run -- --Pasarela:ApiKey=sk_desde_cli
# → {"baseUrl":"...","apiKey":"sk_desde_cli"}
```

Las variables de entorno pisan a user-secrets, y los argumentos pisan a las variables de entorno. Es exactamente el mecanismo que hará que el paso a producción no requiera tocar el código.

## `UserSecretsId`: el identificador que sí va a git

Cada proyecto que use user-secrets declara un identificador en su `.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <UserSecretsId>8f4e2b1a-6c37-4d90-b2e5-71ac0f3d9e42</UserSecretsId>
  </PropertyGroup>
</Project>
```

Ese GUID es el nombre de la **carpeta** donde se guardan los secretos de este proyecto, y nada más. No es una clave, no cifra nada y no da acceso a nada: quien lo lea solo sabrá el nombre de un directorio que existe en *tu* máquina y no en la suya. Por eso puede —y debe— estar versionado: es lo que hace que todo el equipo apunte al mismo cajón, cada persona con su contenido.

No tiene que ser un GUID: cualquier cadena vale, `tienda-api-dev` incluida. El GUID es solo la garantía de que dos proyectos distintos no acaben compartiendo cajón por accidente.

## Dónde viven físicamente los secretos

Los ficheros están en el perfil del usuario del sistema operativo, nunca dentro del repositorio:

- **Windows:** `%APPDATA%\Microsoft\UserSecrets\<UserSecretsId>\secrets.json`
- **Linux / macOS:** `~/.microsoft/usersecrets/<UserSecretsId>/secrets.json`

Vale la pena abrirlo una vez para perderle el misterio. Con `cat ~/.microsoft/usersecrets/8f4e2b1a-6c37-4d90-b2e5-71ac0f3d9e42/secrets.json`, esto es todo lo que hay:

```json
{
  "Pasarela:ApiKey": "sk_test_4f8b21c9d05e",
  "ConnectionStrings:TiendaDb": "Host=localhost;Port=5432;Database=TiendaDB;Username=tienda;Password=local-dev"
}
```

Dos cosas que enseña esa salida. La primera: las claves están **planas**, con `:` dentro del nombre, no anidadas como en `appsettings.json`; ambas formas son válidas para el proveedor de JSON, y la herramienta escribe la plana porque corresponde una a una con lo que le pasas por línea de comandos. La segunda, y más importante: `sk_test_4f8b21c9d05e` se lee ahí, en claro, sin contraseña ni descifrado. Cualquier proceso que corra con tu usuario —un script `postinstall` de una dependencia, una extensión del editor, una copia de seguridad de tu perfil que suba a la nube— puede leer ese fichero igual que tú. Es la razón por la que la única regla real de esta herramienta es: **claves de prueba, nunca de producción**.

## Los comandos, uno a uno

Todos aceptan `--project <ruta al .csproj o a su carpeta>`. Si ya estás en la carpeta del proyecto, se puede omitir.

**`init`** — se ejecuta una vez por proyecto y su único efecto es escribir el `UserSecretsId` en el `.csproj`:

```console
$ dotnet user-secrets init --project src/Tienda.Api
Set UserSecretsId to '8f4e2b1a-6c37-4d90-b2e5-71ac0f3d9e42' for MSBuild project
'C:\repos\tienda\src\Tienda.Api\Tienda.Api.csproj'.
```

Si el proyecto ya tenía identificador, `init` lo respeta y no lo cambia, así que es seguro volver a ejecutarlo. Y si te olvidas de este paso, el siguiente comando falla con un mensaje que dice exactamente qué falta:

```
Could not find the global property 'UserSecretsId' in MSBuild project
'C:\repos\tienda\src\Tienda.Api\Tienda.Api.csproj'.
Ensure this property is set in the project or use the '--id' command line option.
```

**`set`** — guarda un valor. La clave usa `:` para bajar de nivel, igual que las secciones de `appsettings.json`:

```console
$ dotnet user-secrets set "Pasarela:ApiKey" "sk_test_4f8b21c9d05e" --project src/Tienda.Api
Successfully saved Pasarela:ApiKey to the secret store.
```

Los elementos de una lista se direccionan por su índice, empezando en cero, como si el índice fuera otro nivel de anidación:

```bash
dotnet user-secrets set "Pasarela:IpsPermitidas:0" "198.51.100.25" --project src/Tienda.Api
dotnet user-secrets set "Pasarela:IpsPermitidas:1" "198.51.100.26" --project src/Tienda.Api
```

Eso equivale a `"IpsPermitidas": ["198.51.100.25", "198.51.100.26"]` en `appsettings.json`, y se enlaza a un `string[]` sin más ceremonia.

**`list`** — muestra lo que hay guardado, con los valores en claro:

```console
$ dotnet user-secrets list --project src/Tienda.Api
ConnectionStrings:TiendaDb = Host=localhost;Port=5432;Database=TiendaDB;Username=tienda;Password=local-dev
Pasarela:ApiKey = sk_test_4f8b21c9d05e
```

Si el almacén está vacío, el mensaje es `No secrets configured for this application.`, que es la comprobación más rápida de si has hecho `init` en el proyecto que creías. Cuidado con este comando al compartir pantalla.

**`remove`** y **`clear`** — quitan una clave o vacían el almacén del proyecto. Ninguno imprime nada si va bien:

```bash
dotnet user-secrets remove "Pasarela:ApiKey" --project src/Tienda.Api
dotnet user-secrets clear --project src/Tienda.Api
```

## Cómo se leen desde el código

No hay ninguna API específica de user-secrets: los valores están en `IConfiguration` como cualquier otro, así que el acceso directo funciona.

```csharp
var apiKey = builder.Configuration["Pasarela:ApiKey"];
```

Es válido, pero tiene dos defectos: devuelve `null` en silencio si la clave no está, y esparce cadenas literales por todo el código. Lo razonable es el **patrón de opciones**: una clase que describe la sección y se inyecta tipada.

```csharp
public sealed class PasarelaOptions
{
    public const string Seccion = "Pasarela";

    [Required, Url]            public string BaseUrl { get; set; } = "";
    [Required, MinLength(20)]  public string ApiKey  { get; set; } = "";
}
```

Se registra enlazándola a la sección, y aquí está la parte que cambia las cosas: añadir la validación y pedir que se ejecute **al arrancar**.

```csharp
builder.Services
    .AddOptions<PasarelaOptions>()
    .Bind(builder.Configuration.GetSection(PasarelaOptions.Seccion))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Sin `ValidateOnStart()`, la validación no corre hasta que alguien pide `IOptions<PasarelaOptions>` por primera vez, es decir, en la primera petición que intente cobrar un pedido: un `500` en producción, con el usuario delante. Con `ValidateOnStart()`, la aplicación se niega a levantarse:

```
Unhandled exception. Microsoft.Extensions.Options.OptionsValidationException:
DataAnnotation validation failed for 'PasarelaOptions' members: 'ApiKey'
with the error: 'The ApiKey field is required.'.
```

Ese fallo aparece en el despliegue, no tres horas después en el primer pago. Es la diferencia entre un despliegue que no arranca y un incidente.

A partir de ahí el consumo es inyección de dependencias normal: un servicio que reciba `IOptions<PasarelaOptions>` en el constructor y lea `opciones.Value.ApiKey` tiene el valor ya tipado y ya validado.

Lo que hagas con esa clave una vez en memoria —no meterla en la URL, no dejarla en los logs— es otro tema, y lo cubre [Credenciales en llamadas salientes](../secretos-en-llamadas-salientes/Credenciales-en-Llamadas-Salientes.md).

## Cómo se cargan solos, y solo en `Development`

`WebApplication.CreateBuilder(args)` añade el proveedor de user-secrets por su cuenta, con dos condiciones: que el ensamblado de entrada tenga `UserSecretsId` y que el entorno sea `Development`.

El entorno lo decide la variable `ASPNETCORE_ENVIRONMENT`. Si no está definida, .NET asume `Production`, así que el valor `Development` no aparece por magia: lo pone `Properties/launchSettings.json` cuando ejecutas con `dotnet run` o desde el IDE.

```json
"profiles": {
  "Tienda.Api": {
    "commandName": "Project",
    "environmentVariables": { "ASPNETCORE_ENVIRONMENT": "Development" }
  }
}
```

Ese fichero **no se copia a la salida de compilación**. Por eso los secretos aparecen al depurar y desaparecen al ejecutar el `dotnet Tienda.Api.dll` publicado: el entorno pasa a ser `Production` y el proveedor no se añade. Es el comportamiento deseado —así nadie depende de user-secrets sin darse cuenta— pero desconcierta la primera vez.

Cuando el host no es una `WebApplication`, hay que añadirlo a mano. El caso más habitual son los **tests de integración**, que corren en su propio ensamblado:

```csharp
// El .csproj del proyecto de tests necesita su propio <UserSecretsId>
var configuracion = new ConfigurationBuilder()
    .AddJsonFile("appsettings.Test.json", optional: true)
    .AddUserSecrets<PruebasDePagos>()   // tipo de este ensamblado
    .AddEnvironmentVariables()
    .Build();
```

`AddUserSecrets<T>()` busca el `UserSecretsId` en el ensamblado que contiene `T`, no en el que ejecuta: pasar un tipo de `Tienda.Api` desde el proyecto de tests hace que se lean los secretos de la API, que a veces es justo lo que quieres y a veces no. Dicho esto, si los tests necesitan una credencial real de la pasarela, algo huele mal: deberían apuntar al *sandbox* del proveedor o a un doble, porque un suite que no arranca sin la clave de cada persona es una fuente inagotable de «en mi máquina funciona».

## El paso a producción: la misma clave, otro proveedor

En producción no hay user-secrets. La clave `Pasarela:ApiKey` llega por variable de entorno, y su nombre se escribe con **dos guiones bajos** donde había dos puntos: `Pasarela:ApiKey` se convierte en `Pasarela__ApiKey`.

El motivo es que `:` no es un carácter legal en el nombre de una variable de entorno en POSIX (solo letras, dígitos y `_`), así que el proveedor de .NET traduce `__` a `:` al leerlas. Un guion bajo simple no vale: `Pasarela_ApiKey` se interpreta como una clave llamada literalmente así, y tu configuración seguirá vacía.

Así se inyectan en un contenedor y en un servicio de systemd, que son los dos destinos habituales:

```yaml
# docker-compose.yml — los valores vienen del .env del host o del gestor de secretos
services:
  tienda-api:
    image: tienda-api:1.4.0
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      Pasarela__ApiKey: ${PASARELA_API_KEY}
      ConnectionStrings__TiendaDb: ${TIENDA_DB_CONNECTION}
```

```ini
# /etc/systemd/system/tienda-api.service
[Service]
ExecStart=/usr/bin/dotnet /opt/tienda-api/Tienda.Api.dll
Environment=ASPNETCORE_ENVIRONMENT=Production
EnvironmentFile=/etc/tienda-api/secretos.env
```

El `EnvironmentFile` contiene las líneas `Pasarela__ApiKey=sk_live_...` y debe tener permisos `600` y dueño `root`: si lo dejas legible por todos, has cambiado un secreto en tu portátil por un secreto legible por cualquier usuario del servidor.

| | Desarrollo | Producción |
|---|---|---|
| Dónde vive | `secrets.json` en tu perfil, en claro | Variables de entorno del proceso, o un *vault* |
| Quién lo provee | Tú, con `dotnet user-secrets set` | El despliegue: CI/CD, systemd, orquestador |
| Alcance del valor | Clave de pruebas del proveedor | Clave real, con dinero detrás |
| Quién puede leerlo | Cualquier proceso de tu usuario | Solo `root` y el proceso de la aplicación |
| Si se pierde | Lo vuelves a poner en dos minutos | Hay que rotarlo en el proveedor y redesplegar |
| Si se filtra | Incidente menor: clave de test | Incidente serio: cargos reales |

El siguiente escalón, cuando el número de servicios o de personas crece, es un gestor dedicado —Azure Key Vault, AWS Secrets Manager, HashiCorp Vault—. Todos se integran como un proveedor más de `IConfiguration`, así que el código del apartado anterior no cambia; lo que aportan es rotación, auditoría de accesos y permisos por servicio. Si además necesitas guardar credenciales de terceros en tu propia base de datos, eso es otro problema y lo trata [Cifrado en reposo de credenciales](Cifrado-En-Reposo-De-Credenciales.md).

## Onboarding: qué se lleva quien clona el repositorio

Quien clona obtiene el `UserSecretsId` del `.csproj` y ni uno de los valores. Al arrancar verá el error de validación de `PasarelaOptions`, que es informativo pero no dice **cuáles** son todas las claves que hacen falta. Por eso conviene dejar dos cosas en el repositorio: `appsettings.json` con las claves declaradas y vacías, que documenta la forma de la configuración sin filtrar nada, y un script que deje el almacén inicializado con valores de ejemplo.

```bash
#!/usr/bin/env bash
# scripts/init-secretos.sh — deja los user-secrets listos para desarrollo local
set -euo pipefail
PROYECTO="src/Tienda.Api"

secreto() { dotnet user-secrets set "$1" "$2" --project "$PROYECTO"; }

dotnet user-secrets init --project "$PROYECTO"
secreto "ConnectionStrings:TiendaDb" "Host=localhost;Port=5432;Database=TiendaDB;Username=tienda;Password=local-dev"
secreto "Pasarela:ApiKey"            "sk_test_SUSTITUYE_ESTE_VALOR"

echo "Hecho. Pide tu clave de pruebas de la pasarela y vuelve a ejecutar el último set."
dotnet user-secrets list --project "$PROYECTO"
```

Termina con un `list`, así que quien lo ejecuta ve de una vez todas las claves que existen y cuál sigue con el valor de relleno. El script solo contiene valores de ejemplo y de servicios locales, así que puede estar versionado sin problema; lo que nunca se versiona es la variante con la clave real dentro, que es `appsettings.json` con pasos extra.

## Limitaciones y cuándo NO usarlo

- **No está cifrado.** Cualquier proceso que corra con tu usuario lee `secrets.json` en claro. Protege del repositorio, no de tu propia máquina.
- **Es por usuario del sistema operativo.** Dos personas en el mismo usuario de Windows comparten secretos; dos usuarios distintos en la misma máquina, no.
- **No sirve para producción.** El proveedor ni se registra cuando el entorno no es `Development`.
- **No funciona en una librería** que no tenga su propio `UserSecretsId`, y el que cuenta es el del ensamblado de entrada, no el de la biblioteca de clases que hace la llamada.
- **No comparte nada entre personas.** No hay sincronización: cada quien pone sus valores. Si el equipo necesita compartir una credencial de pruebas, hace falta un gestor de contraseñas, no esto.
- **No hay rotación ni auditoría.** Ni caducidad, ni registro de quién leyó qué, ni versiones. Si el secreto necesita eso, necesita un *vault*.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| En local funciona; en producción la clave llega `null` | El valor solo está en user-secrets; falta la variable de entorno en el despliegue |
| La variable de entorno está puesta pero se ignora | Se escribió `Pasarela:ApiKey` o `Pasarela_ApiKey`; el separador correcto es `__` |
| Los secretos no se cargan al depurar | `ASPNETCORE_ENVIRONMENT` no es `Development`; falta el perfil en `launchSettings.json` |
| `list` dice `No secrets configured for this application.` | El `init` se hizo en otro proyecto de la solución; cada `.csproj` tiene su propio almacén |
| `set` responde `Could not find the global property 'UserSecretsId'` | Falta el `dotnet user-secrets init` en ese proyecto |
| La clave llega con comillas: `"sk_test_..."` | Se pasó `set "Pasarela:ApiKey" "\"sk_test_...\""`; las comillas son del shell, no del valor |
| `Could not find a MSBuild project file in <ruta>` | `--project` apunta a una carpeta sin `.csproj`, típicamente la raíz de la solución |
| Los tests no ven los secretos de la API | El proyecto de tests necesita su propio `UserSecretsId`, o `AddUserSecrets<T>` con un tipo de la API |

## Buenas prácticas avanzadas

- **Valida la configuración al arrancar, siempre.** `ValidateDataAnnotations()` + `ValidateOnStart()` convierte «falta un secreto» en un despliegue que no levanta, y no en un `500` intermitente cuatro horas más tarde. Es la diferencia entre un rollback limpio y una investigación con el servicio caído: el fallo ocurre antes de que el balanceador mande tráfico al contenedor nuevo.
- **Declara en `appsettings.json` todas las claves, con el valor vacío.** El fichero versionado es la documentación real de qué configuración existe, y una clave declarada y vacía hace evidente qué hay que rellenar. Además vuelve visible en la revisión de código cualquier intento de escribir un valor sensible ahí.
- **Nunca uses claves de producción en user-secrets, ni «un momento para probar algo».** El fichero está en claro y sobrevive meses en tu portátil sin que nadie lo mire. Todo proveedor serio da un par de claves de test; si el tuyo no separa entornos, es una razón legítima para pedir una segunda cuenta.
- **Verifica el paso a producción con la variable de entorno, no con el código.** Antes de desplegar, arranca la aplicación en local con `ASPNETCORE_ENVIRONMENT=Production` y las variables `__` puestas a mano. Es el único ensayo que detecta un `_` de más, un nombre de sección mal escrito o una clave que solo existía en tu `secrets.json`; leer el YAML del despliegue no lo detecta.
- **Un `UserSecretsId` por proyecto ejecutable, y sabido cuál es.** En una solución con API, *worker* y tests, cada ejecutable tiene su almacén. Compartir el mismo GUID entre dos proyectos «para no repetir» funciona hasta que uno necesita un valor distinto, y entonces el `clear` de un proyecto vacía los secretos del otro.
- **Cronometra el arranque en frío de alguien nuevo.** Si tarda medio día en levantar la API porque va descubriendo claves de una en una, el problema no es de esa persona: falta el script de inicialización. Ese cronómetro es la única medida honesta de si la documentación de configuración sirve.

## Recursos didácticos

- [Safe storage of app secrets in development in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/app-secrets) — la documentación oficial de la herramienta: rutas exactas por sistema operativo, la opción `--id` y el detalle de cómo interactúa con la línea de comandos.
- [Configuration in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/) — la referencia del modelo de proveedores y su orden. Merece una lectura completa: casi todos los problemas de «no lee mi valor» son de precedencia, no de user-secrets.
- [Options pattern in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/options) — el detalle de `IOptions`, `IOptionsSnapshot` y la validación, incluida la validación con código propio cuando las anotaciones no llegan.

---

*En resumen: user-secrets saca la clave del repositorio, no la cifra — úsalo solo con credenciales de prueba en tu máquina, entiende que gana el proveedor de configuración que va después, y prepara el mismo valor como variable de entorno con `__` antes de desplegar.*
