# GitHub Container Registry (ghcr.io)

## ¿Qué es?

Un *registro de contenedores*: un almacén en la nube donde se suben y descargan imágenes Docker, igual que npm almacena paquetes o NuGet librerías. `ghcr.io` es el registro de GitHub — las imágenes viven junto al repositorio que las produce.

## ¿Por qué existe?

Una imagen Docker se construye en un sitio (tu máquina, un runner de CI) pero se ejecuta en otro (un servidor, el portátil de otra persona). Entre medias necesita un lugar donde publicarse y desde donde descargarse con nombre y versión.

El registro más conocido es **Docker Hub**, pero usar ghcr.io tiene una ventaja práctica cuando el código ya está en GitHub: **misma cuenta, mismos permisos, cero configuración extra**. Quien puede ver el repo puede ver sus imágenes (si así se configura), y el pipeline de CI puede publicar sin crear cuentas ni tokens en otro servicio.

> Piensa en un registro como la "tienda de aplicaciones" de tus imágenes: tú publicas versiones etiquetadas y cualquier máquina autorizada las instala por su nombre exacto.

## ¿Cuándo y para qué se usa?

- **Publicar desde CI**: el pipeline construye la imagen de la app (una tienda online, una API...) y la sube etiquetada; el servidor de producción luego hace `pull` de esa etiqueta exacta.
- **Versionar despliegues**: cada release corresponde a una imagen inmutable (`v1.2.0`); volver atrás es descargar la etiqueta anterior.
- **Compartir imágenes internas** entre proyectos o compañeras sin hacerlas públicas.

## Lo mínimo que necesitas saber

**1. El nombre completo de una imagen lleva el registro delante**

```bash
#      registro   propietario   imagen          etiqueta
docker pull ghcr.io/mi-organizacion/tienda-backend:v1.2.0
```

Si no se indica registro (`docker pull nginx`), Docker asume Docker Hub. El propietario en ghcr.io es tu usuario u organización de GitHub, **siempre en minúsculas**.

**2. Autenticarse (en local, con un token clásico con permiso `write:packages`)**

```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u mi-usuario --password-stdin
```

**3. Publicar: etiquetar y subir**

```bash
docker build -t ghcr.io/mi-organizacion/tienda-backend:v1.2.0 .
docker push ghcr.io/mi-organizacion/tienda-backend:v1.2.0
```

**4. Desde GitHub Actions no necesitas crear ningún token**

El propio workflow recibe un `GITHUB_TOKEN` temporal con permiso sobre los paquetes del repo:

```yaml
permissions:
  packages: write
steps:
  - uses: docker/login-action@v3
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}   # lo inyecta GitHub, no se configura
```

**5. Visibilidad y ubicación**

Las imágenes aparecen en la pestaña **Packages** del perfil u organización de GitHub. Cada paquete tiene su propia visibilidad (privada por defecto si el push viene de un repo privado) y se puede vincular al repositorio para heredar sus permisos.

## Lo que NO hace

- **No construye imágenes** — solo las almacena; el `build` ocurre en tu máquina o en CI.
- **No despliega** — publicar en ghcr.io no pone nada en marcha; alguien (un pipeline, un servidor) tiene que hacer `pull` y arrancar el contenedor.
- **No garantiza inmutabilidad de etiquetas** — una etiqueta puede re-publicarse apuntando a otra imagen; por convención las de versión (`v1.2.0`) no se reutilizan nunca (las móviles tipo `latest` sí se mueven).
- **No es solo para Docker** — admite cualquier artefacto OCI (charts de Helm, por ejemplo), aunque su uso típico son imágenes de contenedor.

## Buenas prácticas avanzadas

- **En producción, despliega por digest, no por etiqueta** — como las etiquetas son mutables, `v1.2.0` podría re-publicarse (por error o por un atacante con acceso al registro) y tu servidor descargaría otra cosa sin enterarse. El digest (`ghcr.io/mi-organizacion/tienda-backend@sha256:...`) identifica el contenido exacto y es inmutable por construcción; `docker/build-push-action` lo expone como output del step, listo para pasarlo al despliegue.
- **Genera las etiquetas con `docker/metadata-action`, no a mano** — esta action deriva automáticamente el juego completo de etiquetas desde el evento de Git: de un tag `v1.2.0` saca `v1.2.0`, `v1.2`, `v1` y `latest`, y de un push a rama saca `sha-<commit>`. Así quien consume la imagen elige su nivel de riesgo (fijarse a `v1.2.0` o seguir `v1`) sin que tú mantengas lógica de etiquetado en el workflow.
- **Añade el label `org.opencontainers.image.source` al Dockerfile** — `LABEL org.opencontainers.image.source=https://github.com/mi-organizacion/tienda-backend` vincula el paquete al repositorio automáticamente en el primer push: hereda sus permisos, aparece en su barra lateral y muestra el README. Sin él, el paquete queda "huérfano" y hay que vincularlo a mano en la web (y los permisos no se sincronizan).
- **Vigila las versiones sin etiqueta: se acumulan y, en paquetes privados, se facturan** — cada push que reutiliza una etiqueta deja la imagen anterior como versión *untagged*, y los builds multi-arquitectura generan además manifiestos intermedios sin etiqueta. Programa una limpieza periódica (p. ej. la action `actions/delete-package-versions`), pero con cuidado en multi-arch: borrar manifiestos sin etiqueta a ciegas puede romper la imagen etiquetada que los referencia.
- **Usa ghcr.io también como caché de capas del CI** — los runners de GitHub Actions arrancan vacíos, así que sin caché cada build reconstruye todas las capas. Con `cache-from`/`cache-to` de tipo `registry` (o `gha`), el build reutiliza las capas del build anterior publicadas en el propio registro y un cambio solo de código tarda segundos en lugar de minutos.

---

*En resumen: ghcr.io es el almacén de imágenes que ya viene con tu cuenta de GitHub — publicas versiones etiquetadas desde CI con el token automático, y tus servidores las descargan por nombre exacto.*
