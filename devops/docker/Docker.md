# Docker

## ¿Qué es?

Docker es una plataforma de **contenedores**: empaqueta una aplicación junto con todo lo que necesita para ejecutarse —runtime, dependencias, ficheros de configuración— en una unidad portable llamada *imagen*, que se ejecuta igual en cualquier máquina con Docker instalado.

## ¿Por qué existe?

Existe por *"en mi máquina funciona"*. Una API compila y arranca en el portátil de quien la escribió, y falla en el servidor porque allí hay otra versión del runtime, falta una librería del sistema o una variable de entorno se llama distinto. El problema no es el código: es que el **entorno** viaja por un canal distinto —un documento de instalación, la memoria de alguien, un script desactualizado— y los dos canales se desincronizan siempre. Docker mete el entorno dentro del propio artefacto: si la imagen funciona en tu portátil, funciona en producción, porque es literalmente el mismo sistema de ficheros ejecutándose con el mismo runtime.

> Piensa en el contenedor marítimo, del que Docker toma el nombre. Antes de estandarizarlo, cada mercancía se cargaba a su manera y cada puerto necesitaba saber qué transportaba para manipularla. Con el contenedor, el puerto solo conoce una caja de medidas fijas con unos enganches fijos: da igual si dentro hay plátanos o lavadoras. Tu servidor deja de saber si la aplicación es .NET, Node o Python; solo sabe arrancar cajas.

## ¿Cuándo y para qué se usa?

- **Desplegar aplicaciones**: el servidor no acumula runtimes ni dependencias; cada aplicación trae la suya dentro de su imagen.
- **Entornos de desarrollo reproducibles**: todo el equipo levanta la misma versión de PostgreSQL con un comando, sin instalar nada en el sistema anfitrión.
- **Microservicios**: varios servicios independientes en la misma máquina, aislados entre sí aunque necesiten versiones incompatibles de la misma librería.
- **CI/CD**: ejecutar tests en un entorno limpio y idéntico en cada *pull request*, sin arrastrar basura de la ejecución anterior.

Esta ficha cubre qué es realmente un contenedor y cómo escribir un `Dockerfile`. Operar Docker en un servidor de producción —instalación, rotación de logs, políticas de reinicio— está en [Docker en un VPS](../despliegue-en-vps/Docker-en-un-VPS.md).

## Contenedor y máquina virtual no son lo mismo

Una máquina virtual emula un ordenador completo: virtualiza hardware, arranca su propio kernel y encima de él un sistema operativo entero. Un contenedor **no arranca ningún sistema operativo**: es un proceso normal de Linux que se ejecuta sobre el kernel del host, al que se le ha mentido sobre lo que puede ver.

Esa mentira son dos mecanismos del kernel:

- **Namespaces** — aíslan lo que el proceso *ve*. Tiene su propio árbol de procesos (tu API es el PID 1 y no ve nada más), su propio sistema de ficheros, su propia interfaz de red y su propio hostname. No es que no tenga permiso para ver el resto: es que para él no existe.
- **cgroups** (*control groups*) — limitan lo que el proceso *consume*: cuánta CPU, cuánta memoria, cuánta E/S de disco. Es lo que permite decir "este contenedor no pasa de 512 MB" y que el kernel lo haga cumplir.

| | Máquina virtual | Contenedor |
|---|---|---|
| Qué aísla | Hardware virtualizado | Procesos, mediante namespaces y cgroups |
| Kernel | Uno propio por VM | Comparte el del host |
| Arranque | Decenas de segundos (boot completo) | Milisegundos (arrancar un proceso) |
| Tamaño típico | Varios GB | Decenas o cientos de MB |
| Densidad por servidor | Unas pocas | Decenas o cientos |
| Aislamiento | Fuerte (frontera de hardware) | Menor: un fallo del kernel afecta a todos |
| Sistema operativo invitado | Cualquiera | Solo el que comparta kernel con el host |

La última fila es la limitación que más sorprende: un contenedor Linux **no se ejecuta de forma nativa en Windows**. Docker Desktop arranca por debajo una máquina virtual Linux ligera (WSL2) y ejecuta ahí los contenedores.

## Imagen, contenedor y capas

Tres términos que se confunden constantemente:

- **Imagen**: el artefacto inmutable. Un sistema de ficheros congelado más los metadatos de cómo arrancarlo. No se ejecuta; se guarda y se copia.
- **Contenedor**: una imagen en ejecución. De la misma imagen puedes lanzar veinte contenedores a la vez, y cada uno tiene su propia capa de escritura temporal.
- **Dockerfile**: el fichero de texto que describe cómo construir la imagen.

La relación es la de una clase con sus instancias: la imagen es la clase, el contenedor el objeto. Y lo que escribes dentro del contenedor **se pierde al eliminarlo**, porque esa capa de escritura muere con él; los datos que deben sobrevivir van en un [volumen](../despliegue-en-vps/Volumenes-y-Permisos.md).

La parte que de verdad hay que entender es que **una imagen no es un fichero, es una pila de capas**. Cada instrucción del `Dockerfile` que modifica el sistema de ficheros (`FROM`, `COPY`, `RUN`) produce una capa nueva que guarda solo la *diferencia* respecto a la anterior, y la imagen final es todas ellas apiladas y vistas como un único sistema de ficheros. Dos consecuencias muy prácticas:

1. **Las capas se comparten.** Si tienes cinco imágenes construidas sobre `mcr.microsoft.com/dotnet/aspnet:8.0`, esos 217 MB están una sola vez en disco. Por eso `docker images` puede sumar 3 GB y el disco ocupar mucho menos.
2. **Las capas se cachean.** Si una capa no cambia, Docker reutiliza la que ya calculó. Y si una capa cambia, **todas las siguientes se invalidan** aunque sus instrucciones sean idénticas. De ahí que el orden de las instrucciones decida si un build tarda 5 segundos o 3 minutos.

## El Dockerfile de `tienda-api`, instrucción a instrucción

El ejemplo que acompaña toda la ficha es `tienda-api`, la API .NET de una tienda online. El `Dockerfile` va en la raíz del proyecto, junto al `.csproj`, y se llama exactamente así, sin extensión.

Un detalle antes: .NET necesita el **SDK** para compilar (compilador, plantillas, herramientas: ~830 MB) pero solo el **runtime** para ejecutar (~217 MB). Meter el SDK en la imagen de producción es cargar 600 MB de herramientas que nadie usará, además de todo el código fuente. La solución es un **multi-stage build**: varias etapas en el mismo fichero, de las que solo la última acaba en la imagen final.

```dockerfile
# ---------- Etapa 1: compilar ----------
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Primero solo el .csproj: cambia poco y permite cachear el restore
COPY tienda-api.csproj .
RUN dotnet restore

# Ahora sí el resto del código, que cambia en cada commit
COPY . .
RUN dotnet publish tienda-api.csproj -c Release -o /app/publish --no-restore

# ---------- Etapa 2: ejecutar ----------
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# Solo se copia el resultado compilado; el SDK y el código fuente se quedan atrás
COPY --from=build /app/publish .

ENV ASPNETCORE_HTTP_PORTS=8080
EXPOSE 8080
USER app
ENTRYPOINT ["dotnet", "tienda-api.dll"]
```

Qué hace cada instrucción:

| Instrucción | Qué hace | Detalle que importa |
|---|---|---|
| `FROM imagen AS nombre` | Empieza una etapa a partir de una imagen base | El `AS build` la nombra para poder copiar de ella después |
| `WORKDIR /src` | Fija el directorio de trabajo | Lo crea si no existe; evita `RUN cd ...`, que no persiste entre instrucciones |
| `COPY origen destino` | Copia del contexto de build al sistema de ficheros de la imagen | El origen es relativo al contexto, nunca puede salir de él con `../` |
| `RUN comando` | Ejecuta un comando **durante el build** y congela el resultado en una capa | No se ejecuta al arrancar el contenedor |
| `COPY --from=build` | Copia desde otra etapa en lugar del contexto | La clave del multi-stage |
| `ENV CLAVE=valor` | Variable de entorno presente en tiempo de build y de ejecución | Nunca metas secretos aquí: quedan visibles en la imagen |
| `EXPOSE 8080` | **Documenta** el puerto en el que escucha la app | No publica nada: eso lo hace `-p` en `docker run` |
| `USER app` | Cambia el usuario que ejecutará el proceso | Todo lo posterior corre sin privilegios |
| `ENTRYPOINT ["…"]` | El proceso que arranca con el contenedor | Forma *exec* (lista JSON), no cadena suelta |

Dos más que aparecen a menudo: `CMD` define los argumentos por defecto que recibe el `ENTRYPOINT` y que puedes sobrescribir en `docker run`; `ARG` declara una variable disponible **solo durante el build** (`ARG VERSION=1.0.0`, y se pasa con `docker build --build-arg VERSION=1.4.0`). Y sobre `ASPNETCORE_HTTP_PORTS=8080`: desde .NET 8 ese ya es el puerto por defecto, pero fijarlo evita sorpresas al cambiar de imagen base. Lo que **no** debe hacer la aplicación es escuchar en `localhost`, porque dentro del contenedor `localhost` es el contenedor y nada de fuera podrá conectarse.

## La caché de capas: por qué el `.csproj` va antes que el código

Esta es la optimización con más impacto en el día a día, y explica la parte del `Dockerfile` que parece redundante: ¿por qué copiar el `.csproj`, restaurar y *luego* copiar todo otra vez con `COPY . .`?

❌ **Lo intuitivo, y lento:**

```dockerfile
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish
```

Cualquier cambio en cualquier fichero —una línea en un controlador, una coma en un README— invalida la capa del `COPY . .` y con ella la del `restore`. Docker vuelve a resolver y descargar todos los paquetes NuGet **en cada build**.

✅ **Lo correcto:**

```dockerfile
COPY tienda-api.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish tienda-api.csproj -c Release -o /app/publish --no-restore
```

La capa `COPY tienda-api.csproj .` solo cambia cuando cambian las dependencias del proyecto, que es raro. Mientras el `.csproj` sea idéntico, Docker reutiliza la capa del `restore` y salta directamente a compilar. Se ve en las líneas `CACHED` de la salida del build:

```
[+] Building 21.4s (14/14) FINISHED
 => CACHED [build 3/6] COPY tienda-api.csproj .                              0.0s
 => CACHED [build 4/6] RUN dotnet restore                                    0.0s
 => [build 5/6] COPY . .                                                     0.2s
 => [build 6/6] RUN dotnet publish tienda-api.csproj -c Release ...         18.9s
 => exporting to image                                                       1.4s
 => => naming to docker.io/library/tienda-api:1.0.0                          0.0s
```

Sin ese orden, el paso 4 costaría entre 30 y 90 segundos más en cada build. La regla general: **ordena las instrucciones de menos a más volátil**, dejando para el final lo que cambia en cada commit.

## `.dockerignore`

`COPY . .` copia el directorio entero al *contexto de build*, y ahí está todo: `bin/`, `obj/`, `.git/`, `.vs/` y, con suerte, un `appsettings.Development.json` con la cadena de conexión de tu base de datos local. Tres problemas, en orden de gravedad:

1. **Rompe la caché.** `obj/` cambia cada vez que compilas en local, así que el `COPY . .` se invalida siempre aunque no hayas tocado el código.
2. **Engorda la imagen y el build.** Un `.git/` con años de historia son cientos de MB enviados al demonio en cada build.
3. **Filtra secretos.** Un fichero copiado a una capa **queda en esa capa para siempre**, aunque una instrucción posterior lo borre; quien tenga la imagen puede extraerlo.

El remedio es un `.dockerignore` junto al `Dockerfile`, con la misma sintaxis que `.gitignore`:

```
bin/
obj/
.git/
.vs/
.vscode/
**/appsettings.Development.json
**/*.user
Dockerfile
.dockerignore
README.md
```

Un truco para comprobar qué se está enviando: la primera línea del build muestra el tamaño del contexto. Si `transferring context: 340.12MB` aparece en un proyecto de una API pequeña, falta el `.dockerignore`.

## Comandos del día a día

**Construir y arrancar.** En `build`, el `-t` etiqueta la imagen y el `.` final indica que el contexto es el directorio actual. En `run`, `-d` deja el contenedor en segundo plano, `-p host:contenedor` publica el puerto, `--name` le da un nombre legible y `-e` pasa variables de entorno:

```bash
docker build -t tienda-api:1.0.0 .
docker run -d --name tienda-api -p 8080:8080 \
  -e ASPNETCORE_ENVIRONMENT=Production \
  tienda-api:1.0.0
```

`run` devuelve el ID completo del contenedor. Ojo con el orden en `-p`: **primero el puerto del host**, después el del contenedor. `-p 5000:8080` significa que entras por el 5000 de tu máquina y llegas al 8080 de dentro.

**Ver qué está corriendo:**

```bash
docker ps
```

```
CONTAINER ID   IMAGE              COMMAND                  CREATED         STATUS         PORTS                    NAMES
a3f19c2e8b10   tienda-api:1.0.0   "dotnet tienda-api.d…"   2 minutes ago   Up 2 minutes   0.0.0.0:8080->8080/tcp   tienda-api
```

`docker ps -a` añade los parados, que es donde aparecen los contenedores que murieron al arrancar: si uno "no aparece", casi siempre es que ya terminó.

**Ver los logs**, lo primero que hay que mirar cuando algo falla. `-f` los sigue en vivo y `--tail` limita las líneas iniciales:

```bash
docker logs -f --tail 50 tienda-api
```

```
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://[::]:8080
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
```

**Entrar dentro, parar y eliminar.** `exec -it` abre una terminal interactiva en un contenedor en marcha; si responde `executable file not found`, la imagen no trae `bash` y hay que usar `sh`:

```bash
docker exec -it tienda-api bash
docker stop tienda-api    # envía SIGTERM y espera 10 s antes de matar
docker rm tienda-api      # elimina el contenedor parado y su capa de escritura
docker rm -f tienda-api   # las dos cosas de golpe
```

**Ver las imágenes locales:**

```bash
docker images
```

```
REPOSITORY                        TAG     IMAGE ID       CREATED          SIZE
tienda-api                        1.0.0   7c4b2f8a1d93   2 minutes ago    231MB
mcr.microsoft.com/dotnet/aspnet   8.0     e19b5c33a7f1   3 weeks ago      217MB
mcr.microsoft.com/dotnet/sdk      8.0     b8d21f094c6a   3 weeks ago      834MB
```

Los 231 MB de `tienda-api` no se suman a los 217 MB del runtime: comparten las mismas capas. Lo propio de la aplicación son apenas 14 MB.

## Varios servicios con Docker Compose

`tienda-api` necesita una base de datos. Lanzar dos `docker run` a mano y acordarse de los parámetros no escala; Compose lo declara todo en un `docker-compose.yml`:

```yaml
services:
  tienda-api:
    build: .
    ports:
      - "8080:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__Tienda: "Host=db;Database=tienda;Username=tienda;Password=${DB_PASSWORD}"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16.4
    environment:
      POSTGRES_USER: tienda
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: tienda
    volumes:
      - datos-tienda:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tienda"]
      interval: 5s
      retries: 10
    restart: unless-stopped

volumes:
  datos-tienda:
```

```bash
docker compose up -d --build   # construye y arranca todo en segundo plano
docker compose ps              # estado de los servicios
docker compose logs -f tienda-api
docker compose down            # para y elimina contenedores (los volúmenes se quedan)
```

Dos cosas que se aprenden a base de tropezar:

- **El host de la base de datos es `db`, no `localhost`.** Compose crea una red propia y registra cada servicio con su nombre; dentro de `tienda-api`, `localhost` es `tienda-api`. Cómo funcionan esas redes está en [Redes Docker y exposición de servicios](../despliegue-en-vps/Redes-Docker-y-Exposicion-de-Servicios.md).
- **`depends_on` en su forma corta solo espera a que el contenedor arranque, no a que el servicio esté listo.** PostgreSQL tarda unos segundos en aceptar conexiones, y la API falla en ese hueco. Por eso el ejemplo usa `healthcheck` más `condition: service_healthy`.

El `${DB_PASSWORD}` lo lee Compose de un fichero `.env` en el mismo directorio, que nunca se sube al repositorio.

## No ejecutes como root

Casi todas las imágenes base arrancan como `root`. Es cómodo y es un riesgo real: si alguien explota una vulnerabilidad de tu API, obtiene `root` **dentro** del contenedor, y desde ahí cualquier volumen montado, capacidad extra o error de configuración es un camino hacia el host.

Las imágenes de .NET 8 y posteriores ya traen un usuario sin privilegios llamado `app` (UID 1654), así que basta la línea `USER app` antes del `ENTRYPOINT`. En una imagen que no lo traiga, se crea a mano:

```dockerfile
RUN adduser --disabled-password --gecos "" --uid 1654 app \
    && chown -R app:app /app
USER app
```

Se comprueba desde fuera:

```bash
docker exec tienda-api whoami
```

```
app
```

Efecto colateral que conviene anticipar: un usuario sin privilegios **no puede abrir puertos por debajo del 1024** ni escribir en directorios cuyo propietario sea `root`. Si montas un volumen para logs o ficheros subidos, el directorio del host debe pertenecer al mismo UID; ese detalle está en [Volúmenes y permisos](../despliegue-en-vps/Volumenes-y-Permisos.md).

## Tamaño de imagen: qué base elegir

El tamaño no es cosmética: cada MB se descarga en cada despliegue y en cada job de CI, y cada paquete instalado es superficie de ataque que alguien tendrá que parchear.

| Base para `tienda-api` | Tamaño aprox. | Cuándo usarla |
|---|---|---|
| `sdk:8.0` como imagen final | ~1,05 GB | Nunca en producción. Solo como etapa de build. |
| `aspnet:8.0` (Debian bookworm-slim) | ~231 MB | Por defecto. Máxima compatibilidad, trae `bash` y `glibc`. |
| `aspnet:8.0-alpine` | ~125 MB | Cuando el tamaño importa y has verificado que todo funciona. |
| `aspnet:8.0-noble-chiseled` | ~110 MB | Producción endurecida: sin shell ni gestor de paquetes. |

Alpine usa **musl** en lugar de **glibc** como librería estándar de C. La mayoría de las aplicaciones .NET funcionan sin cambios, pero las que dependen de librerías nativas (algunos drivers, procesamiento de imágenes con SkiaSharp) pueden fallar de formas poco obvias. Y las imágenes *chiseled* no tienen shell, así que `docker exec ... bash` no es una opción: se depura solo con logs.

La ganancia grande, en cualquier caso, no viene de elegir Alpine sino del multi-stage: pasar de 1,05 GB a 231 MB es un factor de 4,5, mucho más de lo que aporta después cambiar de distribución.

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| `curl localhost:8080` da *connection refused* y el contenedor está `Up` | Falta `-p 8080:8080`, o la app escucha en `localhost` dentro del contenedor en vez de en `0.0.0.0` |
| El contenedor arranca y desaparece de `docker ps` al instante | El proceso principal terminó. Mira `docker logs`: casi siempre es el nombre del `.dll` mal escrito en el `ENTRYPOINT` |
| `COPY failed: no such file or directory` | El fichero está excluido en `.dockerignore`, o la ruta sale del contexto de build con `../` |
| El `restore` se repite en cada build pese al orden correcto | Falta `.dockerignore` y `obj/` invalida el `COPY` anterior |
| La API no encuentra la base de datos en Compose | Se usó `localhost` en la cadena de conexión en lugar del nombre del servicio (`db`) |
| Falla al conectar solo los primeros segundos tras `compose up` | `depends_on` sin `healthcheck`: la BD aún no acepta conexiones |
| `Permission denied` al escribir en un volumen montado | El UID del usuario del contenedor no es propietario del directorio del host |
| `docker stop` tarda siempre 10 segundos exactos | `ENTRYPOINT` en forma *shell*: `sh` es PID 1 y la app nunca recibe el `SIGTERM` |
| `no space left on device` en el servidor | Imágenes viejas y caché de build acumuladas: `docker system df` y `docker system prune` |

## Buenas prácticas avanzadas

- **Fija la versión exacta de la imagen base, nunca `latest`.** `FROM postgres:latest` apunta a imágenes distintas según el día en que construyas, así que dos builds del mismo `Dockerfile` producen resultados diferentes y el fallo aparece semanas después sin que nada haya cambiado en el repositorio. Usa etiquetas concretas (`postgres:16.4`) y, para despliegues críticos, el digest inmutable (`postgres:16.4@sha256:...`), que no cambia aunque alguien republique la etiqueta.
- **Usa siempre la forma *exec* en `ENTRYPOINT` y `CMD`.** `ENTRYPOINT dotnet tienda-api.dll` (forma *shell*) envuelve el proceso en `/bin/sh -c`, y ese `sh` pasa a ser PID 1: tu aplicación nunca recibe el `SIGTERM` de `docker stop`, no cierra conexiones ni termina las peticiones en curso, y Docker la mata a la fuerza tras 10 segundos. La forma de lista JSON hace que la app sea PID 1 y reciba las señales directamente.
- **Agrupa los `RUN` que instalan paquetes y limpia en la misma instrucción.** Un `RUN apt-get install` seguido de un `RUN rm -rf /var/lib/apt/lists/*` en otra capa no libera nada: la primera capa ya contiene los ficheros y las capas no se pueden borrar hacia atrás. Encadena ambos con `&&` en una sola instrucción para que la capa nazca ya limpia.
- **Nunca pases secretos con `ARG` ni los dejes en un `ENV`.** Ambos quedan grabados en los metadatos de la imagen y cualquiera con acceso a ella los recupera con `docker history`. Las credenciales se inyectan en tiempo de ejecución (variables del entorno del contenedor, ficheros montados o un gestor de secretos) y, si hacen falta durante el build, con `RUN --mount=type=secret`, que no deja rastro en ninguna capa.
- **Construye la imagen en CI, no en el servidor.** Compilar en el VPS exige tener allí el código fuente y el SDK, y provoca picos de CPU y memoria que degradan los contenedores que están sirviendo tráfico. Que el pipeline construya, publique en un registro como [GitHub Container Registry](GitHub-Container-Registry.md) y el servidor solo haga `pull` convierte el despliegue en una operación de segundos y permite volver atrás cambiando una etiqueta. El montaje completo está en [CI/CD](../ci-cd/README.md) y [despliegue en VPS](../despliegue-en-vps/README.md).

## Recursos didácticos

- [Play with Docker](https://labs.play-with-docker.com/) — una máquina Linux con Docker en el navegador durante cuatro horas, sin instalar nada. Perfecta para escribir el `Dockerfile` de esta ficha y ver de verdad los `CACHED` apareciendo al reconstruir.
- [Dive](https://github.com/wagoodman/dive) — explorador interactivo de imágenes en terminal: recorres capa a capa, ves qué ficheros añadió cada instrucción y cuánto espacio desperdicias. Es la forma más rápida de entender el modelo de capas dejando de creerlo y empezando a verlo.
- [Hadolint](https://hadolint.github.io/hadolint/) — linter de `Dockerfile` con versión web: pegas el tuyo y te señala versiones sin fijar, `RUN` mal agrupados o ejecución como root, con la explicación de cada regla. Útil como revisión antes de dar por bueno un fichero.

---

*En resumen: un contenedor no es una máquina virtual pequeña, es un proceso al que el kernel le ha limitado la vista; y casi todo lo que separa un `Dockerfile` mediocre de uno bueno —multi-stage, orden de las capas, `.dockerignore`, usuario no root— sale de entender que una imagen es una pila de capas cacheadas.*
