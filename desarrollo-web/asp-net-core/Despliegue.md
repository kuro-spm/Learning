# Despliegue de una app ASP.NET Core

## ¿Qué es?

Desplegar es llevar tu aplicación del «funciona en mi máquina» a un servidor donde atiende peticiones reales de forma permanente. En ASP.NET Core son dos gestos: **publicar** (`dotnet publish`, que genera un artefacto optimizado y listo para producción) y **ejecutar ese artefacto** en el entorno de destino, ya sea una máquina, un contenedor o un servicio en la nube.

## ¿Por qué existe?

`dotnet run`, el comando con el que desarrollas, está pensado para tu equipo: compila al vuelo, necesita el SDK completo instalado y sirve en `localhost`. Nada de eso vale para producción, donde quieres un artefacto ya compilado y optimizado, sin las herramientas de desarrollo, con la configuración del entorno real y accesible desde fuera.

`dotnet publish` produce justo ese paquete: los `.dll` de tu app, sus dependencias y los archivos de configuración, recortados y listos para copiar al servidor y arrancar.

> Si vienes de Node, `dotnet publish` es el equivalente conceptual a un `build` de producción: no ejecuta la app, prepara el paquete que sí se ejecutará en el servidor.

## ¿Cuándo y para qué se usa?

Cada vez que quieres que alguien más use tu aplicación: subir la API de una tienda online a un servidor para que la consuma su app móvil, meter el backend de un blog en un contenedor para desplegarlo en la nube, o publicar un panel de administración en la red interna de una empresa. Es el paso que convierte tu código en un servicio en marcha.

## Lo mínimo que necesitas saber

**1. `dotnet publish` genera el artefacto de producción**

```bash
dotnet publish -c Release -o ./publish
```

`-c Release` compila optimizado y sin símbolos de depuración (a diferencia de `Debug`); `-o` es la carpeta de salida. Dentro tendrás `MyApp.dll`, sus dependencias y los `appsettings*.json`: eso es lo que llevas al servidor.

**2. Framework-dependent vs self-contained**

Por defecto la publicación es **framework-dependent**: el artefacto es pequeño pero el servidor necesita tener instalado el runtime de .NET. La alternativa **self-contained** incluye el propio runtime dentro del paquete, así que corre en una máquina sin nada de .NET instalado, a cambio de un artefacto mucho más grande:

```bash
dotnet publish -c Release -r linux-x64 --self-contained
```

**3. En producción ejecutas el `.dll`, no `dotnet run`**

```bash
dotnet MyApp.dll   # arranca Kestrel y se queda escuchando peticiones
```

Esto lanza el servidor [Kestrel](De-NET-Core-a-ASP-NET-Core.md) directamente sobre el artefacto publicado, sin recompilar nada.

**4. La configuración cambia según el entorno**

La variable de entorno `ASPNETCORE_ENVIRONMENT` decide qué configuración se carga. Con valor `Production`, se fusiona `appsettings.Production.json` sobre el `appsettings.json` base:

```bash
export ASPNETCORE_ENVIRONMENT=Production
```

Las claves y cadenas de conexión reales **no van en esos JSON** (acaban en el repositorio): se inyectan como variables de entorno o desde un almacén de secretos.

**5. Kestrel suele ir detrás de un reverse proxy**

En producción, Kestrel rara vez se expone directo a internet. Lo habitual es ponerlo detrás de un *reverse proxy* (Nginx, IIS, un balanceador de la nube) que gestiona el TLS/HTTPS, sirve varios sitios y absorbe el tráfico malicioso. Kestrel escucha en un puerto interno, que fijas con `ASPNETCORE_URLS`:

```bash
export ASPNETCORE_URLS=http://localhost:5000
```

**6. Empaquetar en un contenedor con Docker**

La forma más portable de desplegar: una imagen que ya trae la app y su runtime. Un `Dockerfile` *multi-stage* compila con la imagen del SDK y copia el resultado a la imagen ligera de runtime:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:8.0        # solo runtime, sin SDK
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

## Lo que NO hace

- **`dotnet publish` no despliega** — solo crea el artefacto; llevarlo al servidor (copiarlo, subir la imagen a un registro, un pipeline de CI/CD) es un paso aparte.
- **No instala el runtime por ti** — en modo framework-dependent, si el servidor no tiene .NET, la app no arranca.
- **No gestiona el HTTPS de cara a internet** — de eso se encarga normalmente el reverse proxy, no Kestrel.
- **No mantiene el proceso vivo** — si la máquina se reinicia o la app cae, algo externo tiene que volver a levantarla.

## Buenas prácticas avanzadas

- **La app necesita un supervisor que la reinicie** — Kestrel no resucita solo. En Linux se usa un servicio de `systemd`; en Windows, IIS o un *Windows Service*; en la nube, el orquestador (Kubernetes, App Service). Sin esto, el primer reinicio del servidor deja tu servicio caído hasta que alguien lo note.
- **Configura `ForwardedHeaders` cuando haya un reverse proxy** — detrás de un proxy, todas las peticiones parecen venir de `localhost` y como HTTP. Si no activas el middleware `UseForwardedHeaders`, la app pierde la IP real del cliente y el esquema `https`, lo que rompe la redirección a HTTPS y llena los logs con la IP del proxy. Es un fallo clásico que no se ve hasta que estás en producción.
- **Las migraciones de base de datos no van en el arranque de la app** — es tentador llamar a `Database.Migrate()` en `Program.cs`, pero en producción con varias instancias competirían por migrar a la vez, sin control ni *rollback*. Ejecuta las migraciones como un paso separado del pipeline de despliegue, antes de arrancar la nueva versión.
- **Expón *health checks* para el balanceador** — `AddHealthChecks()` + `MapHealthChecks("/health")` dan un endpoint que el orquestador consulta para saber si la instancia está lista antes de enviarle tráfico (*readiness*) y si sigue viva (*liveness*). Sin ellos, el balanceador manda peticiones a una instancia que aún está arrancando.
- **En el contenedor, imagen mínima y usuario no-root** — usa las variantes *chiseled*/*alpine* de las imágenes oficiales para reducir tamaño y superficie de ataque, y ejecuta como usuario no privilegiado. Una imagen que corre como `root` es un riesgo de seguridad innecesario.

## Recursos didácticos

La guía oficial «Host and deploy ASP.NET Core» cubre cada destino (Linux, IIS, contenedores, Azure) paso a paso: <https://learn.microsoft.com/aspnet/core/host-and-deploy/>. Para probarlo sin servidor, `dotnet publish` seguido de un `docker build` y `docker run` en tu propia máquina te deja ver el artefacto de producción arrancando exactamente como lo haría en la nube. Y para las estrategias de puesta en producción sin cortes (blue-green, canary), la ficha [Estrategias de despliegue](../../devops/ci-cd/Estrategias-de-Despliegue.md) de la guía de CI/CD.

---

*En resumen: desplegar ASP.NET Core es publicar un artefacto optimizado con `dotnet publish`, ejecutarlo con `dotnet MyApp.dll` (normalmente en un contenedor y detrás de un reverse proxy), darle la configuración del entorno por variables y dejar que un supervisor lo mantenga vivo.*
