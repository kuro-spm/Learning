# GitHub Container Registry (ghcr.io)

## ¿Qué es?

Un **registro de contenedores** es un almacén en la nube donde se suben y descargan imágenes Docker, igual que npm almacena paquetes o NuGet librerías. `ghcr.io` es el registro de GitHub: las imágenes viven junto al repositorio que las produce y se publican con la misma cuenta y los mismos permisos.

## ¿Por qué existe?

Una imagen Docker se construye en un sitio (tu portátil, un runner de CI) pero se ejecuta en otro (un servidor, la máquina de otra persona del equipo). Entre medias hace falta un lugar donde publicarla con nombre y versión, y desde donde cualquier máquina autorizada la descargue.

El registro más conocido es **Docker Hub**, y funciona. Pero si el código ya está en GitHub, usar ghcr.io elimina una cuenta, un juego de credenciales y un sitio más donde mirar cuando algo falla: mismo login, mismos permisos, y el pipeline publica sin que nadie cree un token en otro servicio.

> Piensa en un registro como la tienda de aplicaciones de tus imágenes: tú publicas versiones etiquetadas y cualquier máquina autorizada las instala por su nombre exacto. ghcr.io es la tienda que ya viene incluida con la cuenta de GitHub que ya tienes.

Frente a Docker Hub, en las cuatro cosas que se notan a diario:

| | Docker Hub (plan gratuito) | ghcr.io |
|---|---|---|
| **Límites de descarga** | Cuota por horas según cuenta e IP; los runners de CI comparten IP y agotan la cuota ajena | Sin límite de tirones para repositorios públicos; las descargas desde GitHub Actions no consumen cuota |
| **Privacidad** | Un solo repositorio privado en el plan gratuito | Paquetes privados ilimitados; solo cuentan almacenamiento y transferencia fuera de Actions |
| **Integración con el repo** | Ninguna: cuenta aparte, credenciales aparte | El paquete se enlaza al repositorio, hereda sus permisos y aparece en su barra lateral |
| **Coste** | Por plan de suscripción | Gratis en repos públicos; en privados se factura almacenamiento y ancho de banda saliente |

Ese primer punto es la razón por la que muchos equipos migran: un pipeline que falla con `toomanyrequests: You have reached your pull rate limit` no es un problema de código, es un problema de registro.

## ¿Cuándo y para qué se usa?

- **Publicar desde CI.** El pipeline construye la imagen de la aplicación y la sube etiquetada; el servidor de producción hace `pull` de esa etiqueta exacta. Es el uso principal y el que cubre casi toda esta ficha.
- **Versionar despliegues.** Cada release corresponde a una imagen inmutable (`1.4.0`); volver atrás es descargar la etiqueta anterior en lugar de revertir commits y reconstruir.
- **Compartir imágenes internas** entre proyectos o personas del equipo sin hacerlas públicas, y distribuir herramientas propias que otros repos consumen como imagen base.

Esta ficha cubre `ghcr.io` en concreto y **cómo se publica** en él. El otro lado —cómo un servidor de producción se autentica y descarga, con cualquier proveedor— está en [Registros de imágenes privados](../despliegue-en-vps/Registros-de-Imagenes-Privados.md).

---

## Anatomía del nombre de una imagen

Antes de publicar nada hay que entender el nombre, porque de él depende a qué servidor se conecta Docker:

```
ghcr.io / mi-organizacion / tienda-api : 1.4.0
└ registro ┘ └ propietario ┘ └ imagen ┘ └ tag ┘
```

- **Registro** — `ghcr.io`. Si el nombre no empieza por un dominio, Docker asume Docker Hub. Por eso `docker pull nginx` funciona sin configurar nada y `docker pull ghcr.io/...` requiere autenticarse antes si el paquete es privado.
- **Propietario** — tu usuario u organización de GitHub, **siempre en minúsculas**. Si la organización se llama `Mi-Organizacion`, la imagen es `ghcr.io/mi-organizacion/...`; Docker rechaza las mayúsculas antes incluso de hablar con el servidor (`invalid reference format: repository name must be lowercase`).
- **Imagen** — nombre libre. No tiene que coincidir con el del repositorio, aunque suele ser buena idea.
- **Tag** — la versión. Si se omite, Docker asume `:latest`, que casi nunca es lo que quieres (más abajo).

## Autenticarse en ghcr.io

En local hace falta un **PAT** (*Personal Access Token*): una credencial que representa a tu cuenta de GitHub sin ser tu contraseña, y a la que se le conceden permisos concretos llamados *scopes*. Se crea en *Settings → Developer settings → Personal access tokens*.

Los scopes necesarios son solo dos:

| Scope | Para qué | Quién lo necesita |
|---|---|---|
| `read:packages` | Descargar imágenes (`docker pull`) | Servidores, máquinas de desarrollo |
| `write:packages` | Publicar imágenes (`docker push`). Incluye lectura | Quien publique a mano |
| `delete:packages` | Borrar versiones publicadas | Solo para tareas de limpieza |

Un token de servidor lleva **únicamente** `read:packages`. Un token con `write:packages` en una máquina de producción convierte cualquier intrusión en la capacidad de publicar una imagen envenenada que se desplegará sola en todas partes.

El login se hace pasando el token por la entrada estándar:

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u mi-usuario --password-stdin
```

```
Login Succeeded
```

`--password-stdin` no es un detalle estético. La alternativa, `docker login -p "$GHCR_TOKEN"` (❌), deja el token escrito en `~/.bash_history` y visible en la lista de procesos mientras el comando corre; el propio Docker responde `WARNING! Using --password via the CLI is insecure. Use --password-stdin.`

La credencial queda guardada en `~/.docker/config.json` codificada en base64, **no cifrada**. Trátala como una contraseña en claro: `chmod 600` al fichero y rotación inmediata si la máquina se compromete. Sobre cómo manejar este tipo de credenciales en el día a día, ver [Gestión de secretos en desarrollo](../../seguridad/gestion-de-secretos-en-desarrollo/README.md).

## Publicar a mano

Tres pasos: construir, etiquetar y subir. El `build` puede hacer las dos primeras cosas de una vez si le das ya el nombre completo:

```bash
docker build -t ghcr.io/mi-organizacion/tienda-api:1.4.0 .
```

```
[+] Building 34.2s (12/12) FINISHED
 => => naming to ghcr.io/mi-organizacion/tienda-api:1.4.0
```

Si la imagen ya existe con otro nombre (la construiste como `tienda-api:dev`), se le añade el del registro con `docker tag` — una misma imagen puede tener varios nombres — y se sube:

```bash
docker tag tienda-api:dev ghcr.io/mi-organizacion/tienda-api:1.4.0
```

```bash
docker push ghcr.io/mi-organizacion/tienda-api:1.4.0
```

```
The push refers to repository [ghcr.io/mi-organizacion/tienda-api]
5f70bf18a086: Pushed
a1b2c3d4e5f6: Pushed
9e8d7c6b5a43: Layer already exists
1.4.0: digest: sha256:9f2b8c1d... size: 1786
```

Dos detalles: `Layer already exists` significa que esa capa ya estaba en el registro de un push anterior y no se ha vuelto a transferir (por eso el segundo push tarda segundos), y el `digest: sha256:...` final es el identificador **inmutable** del contenido exacto publicado. Publicar a mano está bien para probar; en cuanto haya más de una persona, el push lo hace el CI.

## Publicar desde GitHub Actions

Aquí está la ventaja real de ghcr.io: **no hace falta crear ningún PAT**. Cada ejecución de un workflow recibe un `GITHUB_TOKEN` temporal que GitHub genera e invalida al terminar, y que puede publicar en los paquetes del propio repositorio si se lo pides con `permissions`.

```yaml
name: Publicar imagen

on:
  push:
    tags: ['v*']            # solo se publica al crear un tag de versión

env:
  IMAGE: ghcr.io/${{ github.repository_owner }}/tienda-api

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read        # para hacer checkout del código
      packages: write       # para publicar en ghcr.io — sin esto, el push falla

    steps:
      - uses: actions/checkout@v4

      # Autenticación contra el registro. El token lo inyecta GitHub:
      # no se configura ningún secreto en el repositorio.
      - name: Login en ghcr.io
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Calcula el juego de etiquetas a partir del evento de Git.
      - name: Calcular etiquetas
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=semver,pattern={{version}}       # v1.4.0 -> 1.4.0
            type=semver,pattern={{major}}.{{minor}}  # v1.4.0 -> 1.4
            type=sha,format=long                  # sha-a3f9c21...

      - name: Construir y publicar
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Qué hace y qué deja detrás: al empujar el tag `v1.4.0`, el workflow construye la imagen una vez y publica **tres etiquetas apuntando al mismo digest** (`1.4.0`, `1.4` y `sha-a3f9c21...`), con los labels OCI estándar ya incrustados. `cache-from`/`cache-to` guardan las capas en la caché de Actions, así que un cambio que solo toca código reutiliza las capas de dependencias y el build baja de minutos a segundos. El `${{ secrets.GITHUB_TOKEN }}` no aparece en la pestaña de secretos ni hay que crearlo: existe solo durante el job. Sobre el resto de secretos que sí se configuran a mano, ver [CI/CD](../ci-cd/README.md).

## Estrategia de etiquetado

`latest` es la etiqueta por defecto y una mala idea en producción por una razón concreta: **es móvil**. Apunta a lo último publicado, así que dos servidores que hicieron `pull` en momentos distintos ejecutan código distinto creyendo lo contrario, y `docker ps` no te ayuda a distinguirlos.

| Patrón | Qué identifica | Cuándo usarlo |
|---|---|---|
| `latest` | Lo último publicado, sea lo que sea | Nunca en producción. Como mucho, para probar en local |
| `sha-a3f9c21` | Un commit exacto | Builds de rama, entornos de preview, trazabilidad |
| `1.4.0` (semver) | Una release concreta, inmutable por convención | Producción |
| `1.4`, `1` | La última correctiva / menor de esa línea | Consumidores que aceptan parches automáticos |
| `@sha256:9f2b...` (digest) | Un contenido exacto, inmutable por construcción | Piezas críticas y despliegues verificables |

La combinación práctica es **sha + semver**: cada push a una rama publica `sha-<commit>` para poder desplegar cualquier build a un entorno de prueba, y cada tag de Git publica el semver que va a producción. Así el nombre de la imagen desplegada responde por sí solo a "¿qué código es esto?".

Escribir esas etiquetas a mano se desincroniza enseguida; por eso el ejemplo usa [`docker/metadata-action`](https://github.com/docker/metadata-action), que las deriva del evento de Git y genera además los labels OCI. Si quieres que los tags de versión muevan también `latest`, se añade `type=raw,value=latest,enable={{is_default_branch}}` y la lógica sigue viviendo en un solo sitio.

## Visibilidad del paquete y del repositorio

Esta es la fuente de confusión clásica de ghcr.io: **la visibilidad del paquete es independiente de la del repositorio**.

- Todo paquete nace **privado**, venga de un repositorio privado o público. Que el código sea público no hace pública la imagen, y descargarla exige `docker login` con `read:packages`.
- Hacer público el repositorio más tarde **no cambia** la visibilidad de los paquetes ya publicados.
- Publicar un paquete es una acción explícita en *Package settings → Change visibility*, y una vez público **cualquiera hace `pull` sin autenticarse**, incluso sin cuenta de GitHub.

El síntoma típico es un servidor que falla al descargar una imagen "que es pública porque el repo es público":

```
Error response from daemon: Head "https://ghcr.io/v2/mi-organizacion/tienda-api/manifests/1.4.0": denied
```

Antes de tocar tokens, comprueba la visibilidad del paquete en la pestaña **Packages** de la organización.

## Vincular el paquete al repositorio

Por defecto un paquete recién publicado queda "huérfano": aparece bajo la organización, pero no en el repositorio, no muestra README y sus permisos se gestionan aparte. Se arregla con una etiqueta OCI estándar en el `Dockerfile`:

```dockerfile
LABEL org.opencontainers.image.source=https://github.com/mi-organizacion/tienda-api
```

En el primer push con ese label, GitHub vincula el paquete al repositorio indicado: hereda sus permisos de colaboración, aparece en su barra lateral y muestra su README. Sin él hay que vincularlo a mano en la web, y los permisos no se sincronizan. Se comprueba sobre una imagen ya construida:

```bash
docker inspect ghcr.io/mi-organizacion/tienda-api:1.4.0 \
  -f '{{index .Config.Labels "org.opencontainers.image.source"}}'
```

```
https://github.com/mi-organizacion/tienda-api
```

`docker/metadata-action` añade este label automáticamente, junto con `.revision`, `.created` y `.version`: otra razón para no etiquetar a mano.

## Retención y limpieza

Cada push que reutiliza una etiqueta deja la imagen anterior como versión **untagged**: sigue ocupando espacio y sigue siendo descargable por digest, pero ya no tiene nombre. Los builds multiarquitectura generan además manifiestos intermedios sin etiqueta. En un repositorio activo se acumulan cientos de versiones en meses, y en paquetes privados eso se factura.

GitHub no borra nada por su cuenta, así que la limpieza se programa en un workflow con `on: schedule` (`cron: '0 3 * * 0'`, domingos a las 3:00) y `permissions: packages: write`:

```yaml
      - uses: actions/delete-package-versions@v5
        with:
          package-name: tienda-api
          package-type: container
          min-versions-to-keep: 20
          delete-only-untagged-versions: true
```

Qué hace: conserva las 20 versiones más recientes y borra solo las que no tienen etiqueta. Ese `delete-only-untagged-versions: true` es importante en imágenes multiarquitectura, porque un manifest list etiquetado referencia manifiestos hijos sin etiqueta y borrarlos a ciegas rompe la imagen etiquetada, que sigue apareciendo como válida.

## Lo que NO hace ghcr.io

- **No construye imágenes.** Solo las almacena; el `build` ocurre en tu máquina o en el runner de CI.
- **No despliega.** Publicar no pone nada en marcha: alguien tiene que hacer `pull` y arrancar el contenedor. Eso lo cubre [Despliegue en un VPS](../despliegue-en-vps/README.md).
- **No garantiza inmutabilidad de etiquetas.** Una etiqueta puede re-publicarse apuntando a otra imagen; solo el digest es inmutable.
- **No es solo para Docker.** Admite cualquier artefacto OCI (charts de Helm, por ejemplo).

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `denied: permission_denied: write_package` | Falta `permissions: packages: write` en el job, o el PAT no tiene el scope `write:packages` |
| `unauthorized: authentication required` | No se hizo `docker login ghcr.io`, o el PAT caducó |
| `denied` al hacer `pull` de un repo público | El paquete sigue siendo privado: la visibilidad no se hereda del repositorio |
| `invalid reference format: repository name must be lowercase` | Hay mayúsculas en el nombre de la organización o de la imagen |
| El paquete no aparece en el repositorio | Falta el label `org.opencontainers.image.source`; el paquete está huérfano bajo la organización |
| `manifest unknown` | La etiqueta no existe: errata en el nombre o en la versión |
| Falla al publicar desde un fork | Los workflows de forks reciben un `GITHUB_TOKEN` de solo lectura por diseño |

## Buenas prácticas avanzadas

- **En producción, despliega por digest, no por etiqueta.** Como las etiquetas son mutables, `1.4.0` podría re-publicarse (por error o por alguien con acceso de escritura) y tu servidor descargaría otra cosa sin enterarse. El digest (`ghcr.io/mi-organizacion/tienda-api@sha256:9f2b8c...`) identifica el contenido exacto; `docker/build-push-action` lo expone como `steps.<id>.outputs.digest`, listo para pasarlo al paso de despliegue.
- **El CI escribe, el servidor lee.** El `GITHUB_TOKEN` del workflow tiene `packages: write` y vive segundos; el token del servidor tiene solo `read:packages`. Con permisos disjuntos, un servidor comprometido es un incidente contenido y no una puerta para inyectar código en el registro del que descargan todos los entornos.
- **Usa ghcr.io como caché de capas del CI.** Los runners arrancan vacíos, así que sin caché cada build reconstruye todo. Además de `type=gha`, puedes cachear en el propio registro con `cache-to: type=registry,ref=ghcr.io/mi-organizacion/tienda-api:buildcache,mode=max`, que persiste más allá de los límites de la caché de Actions y se comparte entre repositorios.
- **Etiqueta la procedencia siempre.** Con `org.opencontainers.image.revision` (hash del commit) y `.source` (URL del repo) incrustados, responder a "¿de qué código salió lo que está en producción?" es un `docker inspect`. Sin ellos, la respuesta es una conjetura basada en fechas.
- **Programa la retención desde el primer día.** Es mucho más fácil añadir el workflow de limpieza cuando hay 10 versiones que auditar 800 para decidir cuáles se pueden borrar sin romper un despliegue antiguo.

## Recursos didácticos

- [Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) — la documentación oficial de GitHub. Es la referencia canónica para scopes de PAT, visibilidad de paquetes y el comportamiento exacto del `GITHUB_TOKEN`, que es justo donde la información de blogs suele estar desactualizada.
- [docker/metadata-action](https://github.com/docker/metadata-action) — el README documenta todos los patrones de etiquetado (`type=semver`, `type=sha`, `type=ref`) con ejemplos de qué genera cada uno según el evento de Git. Leerlo entero es la forma más rápida de diseñar una estrategia de tags sin inventarla.
- [Docker: Introduction to GitHub Actions](https://docs.docker.com/build/ci/github-actions/) — la guía oficial de Docker sobre `build-push-action`, con las secciones de caché, multiplataforma y publicación en varios registros a la vez. Complementa a la de GitHub desde el lado de la construcción.

Para el contexto alrededor: [Docker](Docker.md) para el modelo de imágenes y capas, [GitHub Actions y CI/CD](../ci-cd/README.md) para los workflows, [Testcontainers](Testcontainers.md) para consumir imágenes desde los tests, y [Registros de imágenes privados](../despliegue-en-vps/Registros-de-Imagenes-Privados.md) para el lado del servidor.

---

*En resumen: ghcr.io es el almacén de imágenes que ya viene con tu cuenta de GitHub — el CI publica versiones etiquetadas con el token automático del workflow, el paquete se vincula al repositorio con un label OCI, y los servidores descargan por nombre exacto con un token de solo lectura.*
