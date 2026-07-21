# User-secrets en .NET

## Qué son

**User-secrets** (Secret Manager) es el mecanismo oficial de .NET para guardar secretos **durante el desarrollo**, fuera del árbol del proyecto. En lugar de escribir la clave en `appsettings.json` (que va a git), la guardas en un fichero JSON que vive en **tu perfil de usuario** del sistema operativo. La app lo lee automáticamente al arrancar, pero solo en entorno de **desarrollo**.

Es para **dev únicamente**. No es un almacén cifrado ni sirve para producción: el fichero está en claro en tu carpeta de usuario. Su valor es *sacar el secreto del repositorio*, no protegerlo criptográficamente. Para producción se usan variables de entorno o un *vault* gestionado.

## Dónde viven físicamente

Cada proyecto tiene un identificador, el `UserSecretsId` (un GUID), declarado en el `.csproj`. Los secretos de ese proyecto se guardan en:

- **Windows:** `%APPDATA%\Microsoft\UserSecrets\<UserSecretsId>\secrets.json`
- **Linux / macOS:** `~/.microsoft/usersecrets/<UserSecretsId>/secrets.json`

Ese fichero **no está dentro del repositorio**, así que git ni lo ve. Lo único que hay en git es el `UserSecretsId` en el `.csproj`, que es un simple identificador, no un secreto.

## Cómo se cargan solos (y solo en Development)

Al crear el host, .NET añade la fuente de configuración de user-secrets **automáticamente cuando el entorno es `Development`** (y el ensamblado tiene un `UserSecretsId`). Es decir:

```csharp
var builder = WebApplication.CreateBuilder(args);
// Si ASPNETCORE_ENVIRONMENT=Development y hay UserSecretsId,
// los user-secrets ya están en builder.Configuration, por encima de appsettings.json.
```

En producción esa fuente **no se añade**, así que no hay riesgo de que dependas de ella sin darte cuenta. Allí los mismos valores llegan por variables de entorno.

## Los comandos, paso a paso

Todos se ejecutan con `dotnet user-secrets ... --project <ruta-al-.csproj-o-carpeta>`. Si estás dentro de la carpeta del proyecto, puedes omitir `--project`.

**1. Inicializar** (una vez por proyecto): crea el `UserSecretsId` en el `.csproj`.
```bash
dotnet user-secrets init --project src/MiApi
```

**2. Guardar un secreto.** La clave usa `:` para anidar (equivale a las secciones de `appsettings.json`):
```bash
dotnet user-secrets set "ConnectionStrings:Db" "Host=...;Password=..." --project src/MiApi
dotnet user-secrets set "ProveedorExterno:ApiKey" "sk-xxxxx"        --project src/MiApi
```

**3. Listar** lo que tienes guardado:
```bash
dotnet user-secrets list --project src/MiApi
```

**4. Borrar** uno, o vaciar todo:
```bash
dotnet user-secrets remove "ProveedorExterno:ApiKey" --project src/MiApi
dotnet user-secrets clear --project src/MiApi
```

## Cómo se leen en el código

Igual que cualquier otra configuración — no hay API especial. Los user-secrets se fusionan en `IConfiguration`, así que:

```csharp
// Enlazado a una clase de opciones
builder.Services.Configure<ProveedorExternoSettings>(
    builder.Configuration.GetSection("ProveedorExterno"));

// O acceso directo
var apiKey = builder.Configuration["ProveedorExterno:ApiKey"];
```

En producción, esa misma clave `ProveedorExterno:ApiKey` llega por la variable de entorno **`ProveedorExterno__ApiKey`** (dos guiones bajos `__` reemplazan al `:`, por compatibilidad entre sistemas). El código no cambia.

## Notas prácticas

- Como el fichero está por **usuario del sistema operativo**, si dos personas comparten el mismo usuario de Windows comparten también los secretos; si son usuarios distintos, cada una pone los suyos.
- Un compañero que clona el repo verá el `UserSecretsId` pero **no** tus valores: tiene que poner los suyos con `dotnet user-secrets set`. Conviene documentar en el README *qué* claves hay que setear (no sus valores).
- `appsettings.json` debe declarar las claves **vacías o de ejemplo**, para que se vea la forma de la config sin filtrar nada.

## Lo mínimo que hay que recordar

- `init` una vez → `set` cada secreto → `list` para comprobar.
- Viven en `%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json`, **fuera de git**.
- Se cargan **solo en Development**; en producción, variables de entorno (`Seccion__Clave`).
- Es comodidad de dev, **no** cifrado: no lo uses como medida de seguridad en producción.
