# Docker en un VPS

## ¿Qué es?

Instalar y configurar Docker en un servidor de producción: el motor que ejecutará todos tus contenedores, con los ajustes que en tu portátil no importan y en un servidor que lleva meses encendido sí.

## ¿Por qué existe?

Docker en local y Docker en un servidor son el mismo programa con expectativas muy distintas. En tu portátil lo instalas con Docker Desktop, lo arrancas cuando lo necesitas y lo apagas al terminar; si un contenedor se cuelga, lo reinicias. En un servidor nadie está mirando: tiene que arrancar solo después de un reinicio, no puede llenar el disco de logs, y cuando algo falle a las cuatro de la mañana tiene que recuperarse sin ayuda.

Esta ficha cubre esa diferencia. Da por sabido qué es un contenedor y cómo se escribe un `Dockerfile`; si eso te falta, empieza por [Docker](../docker/Docker.md) en la colección de contenedores.

> Si vienes de desplegar publicando en un IIS o un servidor de aplicaciones, piensa en Docker en el VPS como "el runtime que ya no instalas en la máquina": el servidor deja de saber si tu aplicación es .NET, Node o Python. Solo sabe arrancar contenedores.

## ¿Cuándo y para qué se usa?

Siempre que el despliegue vaya a ser dockerizado, que a día de hoy es prácticamente siempre en un VPS. La ventaja concreta es que el servidor no acumula dependencias: no hay un .NET 6 instalado que nadie se atreve a quitar porque una aplicación vieja lo usa, ni conflictos entre dos versiones de Python. Cada aplicación trae la suya dentro de su imagen.

---

## Instalar desde el repositorio oficial (y no desde el de la distribución)

Debian y Ubuntu traen un paquete llamado `docker.io` en sus repositorios. Instalarlo es más rápido y es un error: suele ir varias versiones por detrás y, sobre todo, **no incluye el plugin de Compose**, que es lo que vas a usar el 100 % del tiempo.

La instalación correcta añade el repositorio de Docker al sistema. Son cuatro pasos.

**1. Dependencias para poder añadir un repositorio firmado:**

```bash
apt install ca-certificates curl -y
```

**2. La clave GPG oficial de Docker.** Sirve para que `apt` verifique que los paquetes vienen realmente de Docker y no los ha alterado nadie por el camino:

```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```

Las opciones de `curl` son las de siempre en scripts: `-f` falla si el servidor devuelve un error en vez de guardar la página de error, `-sS` oculta la barra de progreso pero muestra los errores, y `-L` sigue redirecciones. El `chmod a+r` es necesario porque `apt` lee la clave con un usuario sin privilegios.

Si tu servidor es Ubuntu en lugar de Debian, cambia `/linux/debian/` por `/linux/ubuntu/` aquí y en el paso siguiente.

**3. El repositorio:**

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Los dos `$(...)` son sustituciones que rellenan la línea con los datos de tu máquina: `dpkg --print-architecture` devuelve `amd64` o `arm64`, y `$VERSION_CODENAME` el nombre de la versión (`bookworm`, `noble`...). Así la misma orden vale para cualquier servidor. El `> /dev/null` solo evita que `tee` repita por pantalla lo que acaba de escribir.

**4. Instalar:**

```bash
apt update
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Qué es cada paquete: `docker-ce` es el demonio (el servicio que ejecuta contenedores), `docker-ce-cli` el comando `docker`, `containerd.io` el runtime de bajo nivel, `docker-buildx-plugin` el constructor moderno de imágenes y `docker-compose-plugin` el subcomando `docker compose`.

Verifica que ambos responden:

```bash
docker --version
docker compose version
```

```
Docker version 27.3.1, build ce12230
Docker Compose version v2.29.7
```

Ojo con la sintaxis: es `docker compose` (con espacio), el plugin actual. El antiguo `docker-compose` (con guion) era un programa aparte escrito en Python y está descontinuado. Si un tutorial usa el guion, está desactualizado.

## Que arranque solo y sin `sudo`

Dos ajustes que se olvidan y se pagan caros.

**Arranque automático.** Sin esto, un reinicio del servidor —planificado o no— deja todos los contenedores parados hasta que alguien se conecte:

```bash
systemctl enable docker
systemctl start docker
```

`enable` crea el enlace para que arranque en el próximo inicio; `start` lo arranca ahora. Son cosas distintas y hacen falta las dos.

**Usar Docker sin `sudo`.** El demonio escucha en un *socket* que pertenece al grupo `docker`. Añadir tu usuario a ese grupo evita anteponer `sudo` a cada comando:

```bash
usermod -aG docker deployer
```

El cambio de grupo **no se aplica a las sesiones ya abiertas**: hay que cerrar sesión y volver a entrar. Si `docker ps` sigue dando `permission denied` justo después del comando, es esto y no otra cosa.

> Formar parte del grupo `docker` equivale a tener `root`: quien puede lanzar contenedores puede montar el disco entero del host dentro de uno y leerlo o modificarlo. No es un permiso intermedio. Añade al grupo solo a quien ya confiarías el `sudo`.

## `daemon.json`: los dos ajustes que evitan un disco lleno

Este es el fichero de configuración del demonio, y en producción hay dos cosas que conviene poner desde el primer día. No existe por defecto; se crea:

```bash
nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["1.1.1.1", "8.8.8.8"]
}
```

**Rotación de logs.** Por defecto, Docker guarda todo lo que un contenedor escribe por consola en un fichero JSON **que crece sin límite**. Una API con log en modo `Debug` puede generar gigabytes en semanas, y el síntoma es un servidor que se queda sin disco y empieza a fallar por todos lados a la vez, sin que nada apunte a los logs. Con `max-size: 10m` y `max-file: 3`, cada contenedor conserva como mucho 30 MB y va descartando lo más antiguo.

Puedes comprobar cuánto ocupan ahora mismo:

```bash
du -sh /var/lib/docker/containers/*/*-json.log | sort -h | tail -5
```

```
12M     /var/lib/docker/containers/3f2a.../3f2a...-json.log
890M    /var/lib/docker/containers/8c41.../8c41...-json.log   ← este es el problema
```

**Servidores DNS.** Los contenedores heredan la resolución DNS del host, y algunos proveedores de VPS ponen resolutores propios lentos o poco fiables. El síntoma es muy característico: la aplicación funciona pero las llamadas a APIs externas tardan cinco segundos o fallan de forma intermitente con errores de nombre no resuelto. Fijar `1.1.1.1` y `8.8.8.8` lo descarta como causa.

El fichero es JSON estricto: una coma de más y el demonio no arranca. Valídalo antes de reiniciar:

```bash
python3 -c "import json; json.load(open('/etc/docker/daemon.json'))" && echo OK
systemctl restart docker
```

Reiniciar el demonio **para los contenedores** unos segundos. Los que tengan `restart: always` vuelven solos; los demás hay que arrancarlos a mano. Hazlo antes de desplegar nada.

Ojo: la rotación se aplica a los contenedores que se creen **después** del reinicio. Los que ya existían mantienen su configuración hasta que se recrean con `docker compose up -d --force-recreate`.

## Reinicio automático de contenedores

Que Docker arranque al iniciar el sistema no significa que tus contenedores lo hagan. Eso lo decide la política de reinicio de cada servicio:

```yaml
services:
  tienda-api:
    image: registro.ejemplo.com/tienda-api:1.4.0
    restart: always
```

Las opciones y cuándo usar cada una:

| Valor | Comportamiento |
|---|---|
| `no` | Por defecto. No se reinicia nunca. |
| `always` | Se reinicia siempre, incluso tras reiniciar el servidor. |
| `unless-stopped` | Igual que `always`, salvo si lo paraste tú a mano. |
| `on-failure` | Solo si termina con código de error. |

Para servicios de larga duración, `unless-stopped` suele ser la mejor elección: se comporta como `always` pero respeta un `docker stop` deliberado, así que un contenedor que paraste para depurar no reaparece solo al reiniciar la máquina. `always` es la opción segura si prefieres que nada se quede parado por descuido.

Comprueba que la política funciona **antes** de necesitarla:

```bash
reboot
# ...esperar y reconectar
docker ps
```

Si todos los contenedores aparecen con `Up X seconds`, el arranque automático está bien. Descubrir que no lo está durante un corte real es mucho peor.

## Mantener el disco a raya

Docker acumula capas de imágenes viejas, contenedores parados y volúmenes huérfanos con cada despliegue. En un servidor que despliega a diario, esto se come el disco en cuestión de meses.

Primero, ver cuánto ocupa cada cosa:

```bash
docker system df
```

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          24        4         8.2GB     6.7GB (81%)
Containers      6         4         120MB     45MB (37%)
Local Volumes   9         3         2.1GB     1.4GB (66%)
Build Cache     142       0         3.9GB     3.9GB
```

La columna `RECLAIMABLE` es lo que se puede liberar. Para hacerlo:

```bash
docker system prune -af --filter "until=168h"
```

- `-a` incluye las imágenes sin contenedor asociado, no solo las capas sueltas. Es donde está la mayor parte del espacio.
- `-f` no pide confirmación (necesario si lo ejecuta [cron](Tareas-Programadas-con-Cron.md)).
- `--filter "until=168h"` protege lo creado en la última semana, para no borrar la imagen anterior a la que quizá quieras volver.

**`prune` no toca los volúmenes con nombre a menos que añadas `--volumes`.** Y eso es exactamente lo que no quieres hacer sin pensarlo mucho: ahí es donde vive tu base de datos. Un `docker system prune -af --volumes` con un contenedor parado puede borrar datos de producción de forma irrecuperable. Deja los volúmenes fuera de la limpieza automática.

## Buenas prácticas avanzadas

- **Construye las imágenes fuera del servidor.** Compilar en el VPS necesita el código fuente, las herramientas de build y picos de memoria que pueden tumbar los contenedores en marcha. Que el CI construya, publique en un [registro](Registros-de-Imagenes-Privados.md) y el servidor solo haga `pull` convierte el despliegue en una operación de segundos que no consume casi nada, y permite volver a la versión anterior con un cambio de etiqueta.
- **Nunca despliegues con la etiqueta `latest`.** Con `latest` no hay forma de saber qué versión está corriendo ni de volver atrás, y dos servidores que hicieron `pull` en momentos distintos ejecutan código distinto creyendo que es el mismo. Usa etiquetas inmutables (`1.4.0`, o el hash del commit) y el `docker-compose.yml` pasa a ser un registro fiable de lo que hay desplegado.
- **Pon límites de memoria a cada contenedor.** Sin `mem_limit`, un contenedor con una fuga de memoria se lleva por delante toda la RAM del servidor y el *OOM killer* de Linux mata procesos al azar —a menudo la base de datos, que es la que más consume—. Con `mem_limit: 512m`, el fallo queda contenido en el contenedor culpable, que se reinicia solo. La diferencia entre un servicio degradado y un servidor caído.
- **Añade `healthcheck` y úsalo en las dependencias.** Un contenedor "arrancado" no es un contenedor "listo": una base de datos tarda segundos en aceptar conexiones. Sin healthcheck, la API arranca antes, falla al conectar y entra en bucle de reinicios. Definiendo `healthcheck` en la base de datos y `depends_on: { condition: service_healthy }` en la API, el orden queda garantizado sin scripts de espera.
- **No montes el socket de Docker en un contenedor si puedes evitarlo.** `/var/run/docker.sock` da control total del host a quien lo tenga. Algunos servicios legítimos lo necesitan —el [reverse proxy](Reverse-Proxy-con-nginx-proxy.md) lo usa para descubrir contenedores—, y en esos casos móntalo **en solo lectura** (`:ro`). Cualquier otro contenedor que lo pida merece una explicación muy buena.

## Recursos didácticos

- [Play with Docker](https://labs.play-with-docker.com/) — te da una máquina Linux con Docker en el navegador durante cuatro horas. Ideal para practicar `daemon.json`, políticas de reinicio o `prune` sin arriesgar nada, y para comprobar qué pasa al reiniciar el demonio.
- [Documentación oficial de instalación](https://docs.docker.com/engine/install/) — mantiene los comandos exactos por distribución y versión; consúltala si tu servidor no es Debian ni Ubuntu.
- [Explainshell](https://explainshell.com/) — desmenuza los comandos con sustituciones (`$(. /etc/os-release && echo "$VERSION_CODENAME")`) que aparecen en la instalación.

---

*En resumen: instalar Docker en un servidor es fácil; lo que separa un despliegue estable de una llamada a las cuatro de la mañana son la rotación de logs, la política de reinicio y no construir imágenes en la máquina que sirve tráfico.*
