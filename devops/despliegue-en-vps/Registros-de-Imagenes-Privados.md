# Registros de imágenes privados

## ¿Qué es?

Un registro de imágenes es el almacén desde el que un servidor descarga los contenedores que va a ejecutar. Que sea **privado** significa que hace falta autenticarse para descargar de él, y esta ficha va sobre cómo se autentica un servidor de producción.

## ¿Por qué existe?

Docker Hub es público por defecto: cualquiera puede hacer `docker pull nginx` sin identificarse. Eso es perfecto para imágenes base, y es exactamente lo que no quieres para la imagen de tu aplicación, que contiene tu código compilado y a veces configuración sensible.

Un registro privado resuelve dos problemas a la vez. El evidente es el control de acceso: solo quien tenga credenciales descarga tu imagen. El menos evidente, y a menudo más importante en la práctica, es que **el servidor deja de construir imágenes**. En lugar de clonar el repositorio, instalar el SDK y compilar en la máquina que sirve tráfico, el CI construye una vez, publica el resultado, y el servidor solo hace `docker pull` de un artefacto ya listo.

> Si trabajas con NuGet o npm, un registro de imágenes es lo mismo un nivel más arriba: en vez de publicar un paquete de biblioteca, publicas la aplicación entera con su runtime dentro. Y el `docker pull` es el `restore`.

## ¿Cuándo y para qué se usa?

En cuanto el código sea privado, que es casi siempre. Las opciones habituales son el registro del proveedor de nube que ya uses (Artifact Registry en Google Cloud, ECR en AWS, ACR en Azure), el del sistema de control de versiones (GitHub Container Registry, GitLab Registry) o los planes privados de Docker Hub.

La ficha de [GitHub Container Registry](../docker/GitHub-Container-Registry.md) cubre el caso concreto de `ghcr.io` y cómo se publica desde CI. Aquí nos ocupamos del otro lado: **cómo un VPS se autentica y descarga**, independientemente del proveedor.

---

## Cómo se nombra una imagen

El nombre completo de una imagen tiene cuatro partes, y de ellas depende a qué servidor va a conectarse Docker:

```
europe-west1-docker.pkg.dev / mi-proyecto / imagenes / tienda-api : 1.4.0
└──────── registro ────────┘ └── espacio de nombres ──┘ └ imagen ┘ └ tag ┘
```

La primera parte es la clave: si empieza por un nombre de dominio, Docker va a ese registro. Si no lo lleva, asume Docker Hub. Por eso `docker pull nginx` funciona sin configurar nada y `docker pull registro.ejemplo.com/tienda-api:1.4.0` requiere haberse autenticado antes.

Comprobar el nombre completo de lo que tienes en el servidor:

```bash
docker images --format '{{.Repository}}:{{.Tag}}'
```

```
europe-west1-docker.pkg.dev/mi-proyecto/imagenes/tienda-api:1.4.0
nginxproxy/nginx-proxy:1.6
postgres:16
```

## Autenticarse: el caso genérico

El comando universal es `docker login`. Pide usuario y contraseña, o los acepta por la entrada estándar:

```bash
echo "$TOKEN_DEL_REGISTRO" | docker login registro.ejemplo.com -u usuario --password-stdin
```

```
Login Succeeded
```

`--password-stdin` no es un detalle menor. La alternativa, `docker login -p "$TOKEN"`, deja el token **escrito en el historial de la shell** (`~/.bash_history`) y visible en la lista de procesos para cualquier usuario del servidor mientras el comando se ejecuta. Docker incluso lo advierte:

```
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
```

Tras el login, la credencial queda guardada en `~/.docker/config.json`:

```bash
cat ~/.docker/config.json
```

```json
{
  "auths": {
    "registro.ejemplo.com": {
      "auth": "dXN1YXJpbzp0b2tlbi1zZWNyZXRv"
    }
  }
}
```

Y aquí hay algo que sorprende a mucha gente: **ese `auth` no está cifrado, es base64**. Se descodifica con `base64 -d` y devuelve `usuario:token` en claro. No es una contraseña protegida, es una contraseña codificada. Las consecuencias prácticas:

- `chmod 600 ~/.docker/config.json`, siempre.
- Usa un token con permisos de solo lectura para el servidor, nunca tus credenciales personales ni un token con permiso de escritura.
- Si el servidor se compromete, ese token se considera filtrado y hay que rotarlo.

## Autenticarse con un *credential helper*

Los registros de las nubes grandes no usan contraseñas fijas, sino tokens de vida corta que caducan en una hora. Sería inviable renovarlos a mano, así que Docker admite **ayudantes de credenciales**: programas externos que generan el token en el momento de cada `pull`.

La configuración tiene dos pasos. Primero se autentica la herramienta del proveedor en el servidor:

```bash
gcloud auth login          # abre un flujo de autorización en el navegador
gcloud config list         # verificar cuenta y proyecto activos
```

```
[core]
account = despliegue@mi-proyecto.iam.gserviceaccount.com
project = mi-proyecto
```

Y después se le dice a Docker que use esa herramienta para ese registro concreto:

```bash
gcloud auth configure-docker europe-west1-docker.pkg.dev
```

Lo que hace ese comando es añadir una entrada a `~/.docker/config.json`:

```json
{
  "credHelpers": {
    "europe-west1-docker.pkg.dev": "gcloud"
  }
}
```

A partir de ahí, cada `docker pull` contra ese dominio invoca a `gcloud` para obtener un token fresco. No hay credencial almacenada que caduque ni que rotar.

El detalle que hace perder más tiempo aquí es que **la autenticación es por usuario del sistema**, porque `~/.docker/config.json` está en el *home* de cada uno. Si te autenticas como `deployer` y luego ejecutas el despliegue con `sudo`, Docker busca la configuración en `/root/.docker/` y no la encuentra:

```bash
sudo docker pull europe-west1-docker.pkg.dev/mi-proyecto/imagenes/tienda-api:1.4.0
```

```
Error response from daemon: Head "https://europe-west1-docker.pkg.dev/v2/...": denied: Permission denied
```

Lo mismo pasa con [cron](Tareas-Programadas-con-Cron.md), que ejecuta como `root` por defecto. La solución no es autenticarse dos veces, sino ser consistente: si el usuario `deployer` está en el grupo `docker` (ver [Docker en un VPS](Docker-en-un-VPS.md)), no necesita `sudo` para nada, y los tres caminos usan la misma configuración.

## Autenticarse sin persona: cuentas de servicio

Un `gcloud auth login` abre un navegador, lo que sirve para configurar el servidor a mano pero no para un proceso automático. Para eso están las **cuentas de servicio**: identidades que no pertenecen a nadie y se autentican con un fichero de credenciales.

```bash
gcloud auth activate-service-account --key-file=/srv/env/registro-lectura.json
gcloud auth configure-docker europe-west1-docker.pkg.dev
```

Ese fichero JSON es equivalente a una contraseña y merece el mismo trato:

```bash
chmod 600 /srv/env/registro-lectura.json
```

Y sobre todo, la cuenta debe tener **solo permiso de lectura** sobre el registro. El servidor de producción nunca necesita publicar imágenes: eso lo hace el CI, con otra credencial distinta. Si alguien compromete el servidor, la diferencia entre un token de lectura y uno de escritura es la diferencia entre "han visto tu imagen" y "han publicado una imagen envenenada que se desplegará sola en todas partes".

Es exactamente el principio que la colección de seguridad desarrolla en [Credenciales en llamadas salientes](../../seguridad/secretos-en-llamadas-salientes/Credenciales-en-Llamadas-Salientes.md): cada credencial con el mínimo alcance que necesite.

## Desplegar tirando de la imagen

Con la autenticación resuelta, el `docker-compose.yml` referencia la imagen por su nombre completo:

```yaml
services:
  tienda-api:
    image: europe-west1-docker.pkg.dev/mi-proyecto/imagenes/tienda-api:1.4.0
    container_name: tienda-api
    restart: always
    networks:
      - internal-net
```

Y el despliegue completo son dos comandos:

```bash
cd /srv/docker/tienda
docker compose pull && docker compose up -d
```

```
[+] Pulling 1/1
 ✔ tienda-api Pulled                                          8.4s
[+] Running 1/1
 ✔ Container tienda-api  Started                              1.2s
```

`docker compose pull` descarga las imágenes nuevas mientras los contenedores viejos siguen sirviendo tráfico; `up -d` solo entonces sustituye los que hayan cambiado. Separar los dos pasos reduce el tiempo de corte a los segundos que tarda el arranque, en lugar de incluir toda la descarga.

## Por qué `latest` es una mala idea en producción

Es tentador escribir `image: .../tienda-api:latest` y hacer `pull` cada vez. Los problemas:

- **No sabes qué está corriendo.** `docker ps` dice `latest` en ambos casos, tanto si el contenedor lleva un mes como si se creó hace un minuto con código distinto.
- **No puedes volver atrás.** Si el despliegue rompe algo, no hay una etiqueta anterior a la que apuntar.
- **Dos servidores pueden ejecutar cosas distintas** creyendo lo contrario, según cuándo hizo `pull` cada uno.
- **`docker compose up -d` no actualiza nada** si la etiqueta no cambió, aunque en el registro haya una imagen nueva. Hace falta el `pull` explícito, y ese paso se olvida.

La alternativa es una etiqueta inmutable por versión: el número de versión (`1.4.0`), el hash del commit (`a3f9c21`) o ambos. Así el `docker-compose.yml` **es** el registro de qué hay desplegado, y volver atrás es cambiar un número y ejecutar `up -d`.

Para saber exactamente qué imagen corre un contenedor, más allá de la etiqueta, está el *digest* — el hash del contenido, que no se puede reasignar:

```bash
docker inspect tienda-api -f '{{.Image}} {{index .Config.Labels "org.opencontainers.image.revision"}}'
```

```
sha256:9f2b8c... a3f9c21
```

## Errores frecuentes

| Mensaje | Causa |
|---|---|
| `denied: Permission denied` | No autenticado, o autenticado con otro usuario del sistema |
| `unauthorized: authentication required` | Token caducado o sin permiso sobre ese repositorio |
| `manifest unknown` | La etiqueta no existe: errata en el nombre o en la versión |
| `no basic auth credentials` | Falta el `docker login` para ese registro concreto |
| `exec: "docker-credential-gcloud": not found` | El helper está configurado pero el binario no está en el `PATH` |

Ese último es muy típico en cron y en scripts: `gcloud` instalado vía snap no siempre está en el `PATH` de un entorno no interactivo. Se comprueba con `which docker-credential-gcloud` desde el mismo contexto donde falla.

## Buenas prácticas avanzadas

- **El servidor solo lee; el CI solo escribe.** Dos credenciales distintas con permisos disjuntos. Es el control que convierte un servidor comprometido en un incidente contenido en lugar de en una puerta para inyectar código en el registro del que todos los entornos descargan.
- **Ancla por digest lo que no puede fallar.** Una etiqueta se puede reasignar: quien publique `1.4.0` otra vez cambia lo que descargas sin cambiar el nombre. `image: .../tienda-api@sha256:9f2b8c...` referencia un contenido exacto e irrepetible. Es incómodo de leer y es lo correcto para el proxy y para cualquier pieza crítica.
- **Configura la limpieza en el registro, no solo en el servidor.** El `docker system prune` del VPS libera disco local, pero el registro sigue acumulando cada imagen que has publicado, y eso se factura. Casi todos los proveedores permiten reglas de retención por antigüedad o número de versiones; sin ellas, la factura crece en silencio durante meses.
- **Firma o al menos etiqueta la procedencia de las imágenes.** Añadir en el `Dockerfile` las etiquetas OCI estándar (`org.opencontainers.image.revision` con el hash del commit, `.source` con la URL del repositorio) permite responder a "¿de qué código salió esto que está en producción?" en un comando. Sin eso, la única respuesta es una conjetura basada en fechas.
- **Ten un plan para cuando el registro no esté disponible.** Si el registro cae y necesitas recrear un contenedor, no hay `pull` que valga. Las imágenes en producción ya están en el disco del servidor, así que **no borres con `prune` la versión anterior a la actual**: el `--filter "until=168h"` de la ficha de [Docker en un VPS](Docker-en-un-VPS.md) existe justo para eso.

## Recursos didácticos

- [Dive](https://github.com/wagoodman/dive) — herramienta de terminal que abre una imagen capa por capa y muestra qué ficheros añade cada una y cuánto espacio desperdicia. Es la forma más rápida de entender por qué una imagen pesa 1,2 GB y de bajarla a 200 MB.
- [Docker Hub](https://hub.docker.com/) — aunque uses un registro privado, explorar los tags y capas de imágenes públicas conocidas ayuda a fijar la relación entre nombre, etiqueta y digest.
- [Especificación de anotaciones OCI](https://github.com/opencontainers/image-spec/blob/main/annotations.md) — la lista de etiquetas estándar (`revision`, `source`, `created`, `version`) que conviene añadir a toda imagen propia.

---

*En resumen: el servidor de producción no construye imágenes, las descarga — con un token de solo lectura, con el usuario correcto y con una etiqueta que diga exactamente qué versión es.*
