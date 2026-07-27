# Volúmenes y permisos

## ¿Qué es?

Un volumen es la forma que tiene un contenedor de guardar datos que sobrevivan a su destrucción. Los permisos son el motivo por el que, la primera vez que montas una carpeta del servidor dentro de un contenedor, aparece un `Permission denied` que no se entiende.

## ¿Por qué existe?

El sistema de ficheros de un contenedor es efímero por diseño: al recrearlo —cosa que ocurre en cada despliegue— todo lo que se escribió dentro desaparece. Eso es bueno para la aplicación, que así es siempre idéntica a su imagen, y catastrófico para su base de datos.

Los volúmenes rompen esa regla de forma controlada: montan un trozo de almacenamiento del host en una ruta del contenedor. Lo que se escriba ahí vive fuera del contenedor y sobrevive a todo.

El problema de permisos surge porque Linux identifica a los usuarios por número, no por nombre, y **el contenedor y el host no comparten la lista de usuarios**. El proceso de dentro escribe con el número que le tocó al construir la imagen; el directorio de fuera pertenece a otro número. Como los números no coinciden, el sistema deniega la escritura — y el mensaje de error no menciona nada de esto.

> Si has peleado con permisos de una carpeta compartida en red entre Windows y Linux, el mecanismo es el mismo: dos sistemas que no comparten el directorio de usuarios y un fichero que pertenece a alguien que "no existe" al otro lado.

## ¿Cuándo y para qué se usa?

En tres situaciones que aparecen en cualquier despliegue: guardar los datos de una base de datos, sacar los logs del contenedor al host para poder leerlos o rotarlos, y meter ficheros de configuración dentro sin tener que reconstruir la imagen.

---

## Los dos tipos de montaje, y cuándo usar cada uno

Docker ofrece dos mecanismos que se escriben casi igual y se comportan muy distinto.

**Volumen con nombre.** Docker gestiona el almacenamiento en un directorio suyo (`/var/lib/docker/volumes/`) y tú solo le das un nombre:

```yaml
services:
  tienda-db:
    image: postgres:16
    volumes:
      - tienda-db-data:/var/lib/postgresql/data

volumes:
  tienda-db-data:
```

**Bind mount.** Montas una ruta concreta del host, indicada con `/` o `./` al principio:

```yaml
services:
  tienda-api:
    image: registro.ejemplo.com/tienda-api:1.4.0
    volumes:
      - /srv/logs/tienda:/app/log
```

Cuándo usar cada uno:

| | Volumen con nombre | Bind mount |
|---|---|---|
| Ruta en el host | La gestiona Docker | La eliges tú |
| Permisos al crearse | Docker copia los del contenedor | Los del host, sin ajustar |
| Mejor para | Datos de bases de datos | Logs, configuración, scripts |
| Copia de seguridad | Vía contenedor auxiliar | `cp` o `tar` directo |

La regla práctica: **volumen con nombre para datos que solo entiende el contenedor** (los ficheros internos de PostgreSQL) y **bind mount para lo que tú vas a leer o escribir desde el host** (logs que quieres rotar, un `.env`, un script de arranque).

Hay una diferencia de comportamiento que explica la tabla y que es la clave de todo lo que viene: cuando Docker crea un volumen con nombre por primera vez, **copia dentro el contenido y los permisos que la imagen tuviera en esa ruta**. Un bind mount no hace nada de eso: monta el directorio del host tal cual, con su propietario y sus permisos.

Por eso los volúmenes con nombre "simplemente funcionan" y los bind mounts dan problemas.

## El error de permisos, paso a paso

Este es el escenario completo. Montas un directorio del host para recoger los logs de la aplicación:

```yaml
services:
  tienda-api:
    image: registro.ejemplo.com/tienda-api:1.4.0
    volumes:
      - /srv/logs/tienda:/app/log
```

Y el contenedor arranca y se cae, o arranca y no escribe ningún log. Para ver qué pasa, pregúntaselo al propio contenedor:

```bash
docker exec -it tienda-api sh -c 'id; ls -ld /app/log; touch /app/log/prueba || echo "fallo: $?"'
```

```
uid=1654(app) gid=1654(app) groups=1654(app)
drwxr-xr-x 2 root root 4096 Aug 25 08:43 /app/log
touch: cannot touch '/app/log/prueba': Permission denied
fallo: 1
```

Leído en orden, el diagnóstico está completo:

- `uid=1654(app)` — dentro del contenedor, el proceso corre como el usuario 1654. Muchas imágenes modernas crean un usuario sin privilegios en lugar de usar `root`, que es lo correcto por seguridad.
- `drwxr-xr-x 2 root root` — el directorio montado pertenece a `root` (uid 0) y los demás solo tienen lectura y acceso (`r-x`), sin `w`.
- `Permission denied` — el usuario 1654 no es el propietario ni está en el grupo, así que se le aplican los permisos de "otros": no puede escribir.

El detalle que hace esto confuso: el nombre `app` que muestra el contenedor **no existe en el host**. Si haces `ls -ln /srv/logs/tienda` en el servidor verás números, y esos números son lo único que ambos lados comparten.

### La solución: hacer coincidir el uid

Basta con que el directorio del host pertenezca al mismo número que usa el proceso de dentro:

```bash
chown -R 1654:1654 /srv/logs/tienda
```

Se escribe el número, no un nombre, precisamente porque ese usuario no existe en el host. Linux lo acepta sin problema: los permisos son numéricos.

Verifica que ahora sí:

```bash
docker exec tienda-api touch /app/log/prueba && echo "escritura OK"
```

```
escritura OK
```

Y si no sabes qué uid usa una imagen antes de arrancarla, se puede consultar sin ejecutarla del todo:

```bash
docker run --rm registro.ejemplo.com/tienda-api:1.4.0 id
```

```
uid=1654(app) gid=1654(app) groups=1654(app)
```

### La alternativa: forzar el usuario del contenedor

En vez de adaptar el host al contenedor, se puede hacer lo contrario con `user:`:

```yaml
services:
  tienda-api:
    image: registro.ejemplo.com/tienda-api:1.4.0
    user: "1000:1000"        # el uid del usuario del host
    volumes:
      - /srv/logs/tienda:/app/log
```

Funciona y evita el `chown`, pero tiene un riesgo: la aplicación puede necesitar acceder a otras rutas de la imagen que sí pertenecen a su usuario original, y al cambiarlo se rompen. Como norma, prefiere el `chown` en el host y reserva `user:` para imágenes que documenten explícitamente que lo soportan.

Lo que **no** debes hacer es la solución que aparece en la mitad de las respuestas de internet:

```bash
chmod -R 777 /srv/logs/tienda   # ❌ escritura para cualquier usuario del servidor
```

Funciona, y por eso se propone tanto. También significa que cualquier proceso del servidor puede modificar o borrar esos ficheros. En un directorio de logs es feo; en uno de configuración o de datos es una vulnerabilidad.

## Montar configuración sin reconstruir la imagen

El otro uso frecuente del bind mount es inyectar un fichero concreto. Se puede montar un fichero suelto, no solo un directorio, y conviene añadir `:ro` para que el contenedor no pueda modificarlo:

```yaml
services:
  tienda-api:
    volumes:
      - /srv/env/tienda.env:/app/appsettings.Production.json:ro
```

Cuidado con un comportamiento que confunde: si la ruta del host **no existe**, Docker crea un **directorio** vacío en su lugar en vez de fallar. El síntoma es que la aplicación se queja de que el fichero de configuración tiene un formato inválido, cuando en realidad es un directorio. Comprueba siempre que el fichero existe antes de arrancar:

```bash
ls -l /srv/env/tienda.env
```

Para secretos, la variante recomendada es no montar el fichero dentro sino cargarlo como variables de entorno con `env_file:`, que evita que quede accesible en el sistema de ficheros del contenedor:

```yaml
services:
  tienda-api:
    env_file:
      - /srv/env/tienda.env
```

Y en cualquier caso, `chmod 600` sobre ese fichero para que solo su propietario lo lea. Sobre por qué esos ficheros nunca deben llegar al repositorio, la colección de seguridad lo cubre en [Por qué los secretos no van a Git](../../seguridad/gestion-de-secretos-en-desarrollo/Por-Que-Los-Secretos-No-Van-A-Git.md).

## Copiar y restaurar un volumen con nombre

Un bind mount se copia con `cp` porque sus ficheros están en una ruta que controlas. Un volumen con nombre vive en `/var/lib/docker/volumes/` y, aunque técnicamente se puede copiar de ahí, la forma correcta es a través de un contenedor auxiliar:

```bash
docker run --rm \
  -v tienda-db-data:/origen:ro \
  -v /srv/backups:/destino \
  alpine tar czf /destino/tienda-db-data.tar.gz -C /origen .
```

Cómo funciona: se arranca un contenedor Alpine desechable (`--rm` lo borra al terminar) con el volumen montado en `/origen` en solo lectura y una carpeta del host en `/destino`, y se comprime lo primero dentro de lo segundo. El `-C /origen .` guarda rutas relativas, que es lo que hace que la restauración funcione en cualquier destino.

Para restaurar, el mismo truco al revés:

```bash
docker run --rm \
  -v tienda-db-data:/destino \
  -v /srv/backups:/origen:ro \
  alpine tar xzf /origen/tienda-db-data.tar.gz -C /destino
```

**Para una base de datos, esto no es lo que quieres.** Copiar sus ficheros mientras está en marcha produce una copia inconsistente que puede no restaurar. Lo correcto es un volcado lógico (`pg_dump`, `mysqldump`, `mongodump`), que es lo que se explica en [Copias de seguridad](Copias-de-Seguridad.md). El método de arriba sirve para volúmenes de ficheros: subidas de usuarios, imágenes, adjuntos.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `Permission denied` al escribir | El uid del contenedor no es propietario del directorio del host |
| La configuración montada "es un directorio" | La ruta del host no existía y Docker creó una carpeta |
| El volumen aparece vacío dentro | La ruta de montaje tapa el contenido original de la imagen |
| Los datos desaparecen al recrear | Se usó un bind mount a una ruta temporal, o no había volumen |
| `docker compose down -v` borró la BD | `-v` elimina también los volúmenes con nombre |

Ese último merece un aviso explícito: `docker compose down` para y borra los contenedores, pero conserva los volúmenes. `docker compose down -v` **borra también los datos**. Es un carácter de diferencia entre reiniciar un servicio y perder la base de datos de producción.

Para inspeccionar qué está montado realmente en un contenedor en marcha:

```bash
docker inspect tienda-api -f '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}}{{println}}{{end}}'
```

```
bind /srv/logs/tienda -> /app/log
volume /var/lib/docker/volumes/tienda-db-data/_data -> /var/lib/postgresql/data
```

## Buenas prácticas avanzadas

- **Monta en solo lectura todo lo que el contenedor no deba modificar.** Añadir `:ro` a la configuración, los certificados y el socket de Docker cuesta tres caracteres y convierte un contenedor comprometido en uno que no puede alterar el host. Es especialmente crítico con `/var/run/docker.sock`, donde escritura equivale a control total de la máquina.
- **Declara el `uid:gid` en el `Dockerfile` y documéntalo.** Una imagen que crea su usuario con un `USER app` sin número deja el uid a merced del orden de instalación de paquetes, y puede cambiar entre versiones — con lo que un despliegue rompe los permisos de un directorio que llevaba meses funcionando. Fijar `RUN adduser -u 1654 app` hace el número parte del contrato de la imagen.
- **Nunca montes un bind mount sobre un directorio que la imagen ya use.** Si montas `/srv/algo:/app` y la imagen tenía la aplicación en `/app`, el montaje **tapa** el contenido original y el contenedor arranca vacío. El error parece de la aplicación ("no encuentra el ejecutable") y la causa es el volumen. Monta siempre en subdirectorios dedicados.
- **Comprueba el espacio de los volúmenes por separado.** `df -h` mira el disco entero y `docker system df -v` desglosa por volumen. Una base de datos que crece sin vigilancia llena el disco, y cuando eso pasa **todos** los contenedores fallan a la vez con errores que no apuntan al disco. Un aviso al 80 % evita la investigación a ciegas.
- **Haz el `chown` en el despliegue, no a mano.** Un `chown` manual se pierde en cuanto alguien recrea el directorio o monta un servidor nuevo, y el fallo reaparece meses después sin que nadie recuerde la solución. Ponlo en el script de despliegue junto al `docker compose up`, para que la configuración correcta sea reproducible.

## Recursos didácticos

- [Chmod Calculator](https://chmod-calculator.com/) — marcas las casillas de lectura, escritura y ejecución y te da el número octal, o al revés. Muy útil mientras el `755` y el `600` no sean automáticos.
- [Explainshell](https://explainshell.com/) — desmenuza los comandos de esta ficha, en particular los `docker run` con dos volúmenes y `tar -C`.
- [Documentación de volúmenes de Docker](https://docs.docker.com/engine/storage/volumes/) — la referencia oficial, con la comparativa entre volúmenes, bind mounts y `tmpfs`.

---

*En resumen: el contenedor y el host no comparten usuarios, solo números — si algo da `Permission denied`, mira el `uid` de dentro y haz que el directorio de fuera le pertenezca.*
