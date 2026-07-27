# Tareas programadas con cron

## ¿Qué es?

Cron es el servicio de Linux que ejecuta comandos a horas determinadas. Le das una expresión de cinco campos y un comando, y se encarga de lanzarlo mientras el servidor esté encendido.

## ¿Por qué existe?

Todo servidor acumula trabajo que hay que hacer con regularidad y que nadie quiere hacer a mano: la [copia de seguridad](Copias-de-Seguridad.md) diaria, la limpieza de imágenes viejas, la rotación de logs, el aviso de que un certificado está a punto de caducar.

Cron existe desde los años setenta y sigue siendo la respuesta por defecto porque es sencillo, está en todas partes y no depende de nada. Su reputación de "cosa que falla en silencio" no viene del programa, sino de que su entorno de ejecución es mucho más pobre de lo que la gente supone, y casi todos los problemas salen de ahí.

> Si has configurado un *background job* recurrente en una aplicación (un Hangfire, un Quartz, un `BackgroundService`), cron es lo mismo a nivel de sistema operativo: la diferencia es que no tiene reintentos, ni cola, ni registro de ejecuciones. Lanza el comando y se olvida.

## ¿Cuándo y para qué se usa?

Para mantenimiento del servidor: lo que no forma parte de la aplicación pero hace que siga funcionando. Es la herramienta correcta para tareas idempotentes que se pueden repetir sin daño y cuyo fallo puntual no es crítico.

No es la herramienta correcta para lógica de negocio con reintentos, dependencias entre pasos o necesidad de saber qué pasó en cada ejecución. Para eso está la cola de trabajos de la aplicación.

---

## La expresión de cinco campos

Una línea de cron son cinco campos de tiempo y un comando:

```
┌───────────── minuto (0-59)
│ ┌─────────── hora (0-23)
│ │ ┌───────── día del mes (1-31)
│ │ │ ┌─────── mes (1-12)
│ │ │ │ ┌───── día de la semana (0-7, donde 0 y 7 son domingo)
│ │ │ │ │
0 3 * * *  /bin/bash /srv/scripts/backup.sh
```

El `*` significa "cualquier valor". La línea de arriba se lee: minuto 0, hora 3, cualquier día, cualquier mes, cualquier día de la semana → **todos los días a las 3:00**.

Los operadores que cubren casi todo:

| Expresión | Cuándo se ejecuta |
|---|---|
| `0 3 * * *` | Todos los días a las 3:00 |
| `*/15 * * * *` | Cada 15 minutos |
| `0 2 * * 0` | Los domingos a las 2:00 |
| `0 9 1 * *` | El día 1 de cada mes a las 9:00 |
| `0 9-18 * * 1-5` | Cada hora en punto, de 9 a 18, de lunes a viernes |
| `30 4 * * 1,4` | Lunes y jueves a las 4:30 |

Dos trampas de la sintaxis:

- **`*/15` no significa "cada 15 minutos desde ahora"**, sino "en los minutos divisibles entre 15": :00, :15, :30 y :45. Si el script tarda 20 minutos, se solaparán ejecuciones.
- **Cuando se especifican a la vez día del mes y día de la semana, cron hace un OR, no un AND.** `0 3 15 * 1` no es "el día 15 si es lunes", sino "el día 15 **o** cualquier lunes". Es de los errores más difíciles de detectar porque la tarea se ejecuta de más, no de menos.

## Instalar una tarea

El *crontab* es la tabla de tareas de un usuario. Cada usuario tiene la suya, y eso importa: la de `root` y la de `deployer` son ficheros distintos con permisos distintos.

```bash
sudo crontab -e        # edita el crontab de root
```

La primera vez pregunta qué editor usar. Añade la línea al final:

```cron
0 3 * * * /bin/bash /srv/scripts/backup.sh >> /srv/logs/backup.log 2>&1
```

Y comprueba que quedó registrada:

```bash
sudo crontab -l
```

```
0 3 * * * /bin/bash /srv/scripts/backup.sh >> /srv/logs/backup.log 2>&1
```

Nunca edites el fichero de `/var/spool/cron/` directamente: `crontab -e` valida la sintaxis antes de guardar y recarga el servicio solo. Si te equivocas, avisa:

```
crontab: installing new crontab
"/tmp/crontab.XXXX":1: bad minute
errors in crontab file, can't install.
```

Y no hace falta reiniciar nada. `systemctl restart cron` es innecesario tras un `crontab -e`.

## Las tres cosas que hacen fallar a cron

Aquí está el 90 % de los problemas reales, y las tres tienen la misma raíz: **cron no ejecuta tus comandos en el entorno al que estás acostumbrado**.

### 1. El PATH es mínimo

Tu sesión interactiva carga `~/.bashrc`, `/etc/profile` y demás. Cron no carga nada de eso: arranca con un `PATH` que suele ser solo `/usr/bin:/bin`.

El resultado es el error más desconcertante del mundo: el script funciona perfectamente cuando lo lanzas tú y falla en cron con un `command not found` de un programa que evidentemente está instalado.

```
/srv/scripts/backup.sh: line 24: gsutil: command not found
```

`gsutil`, `docker compose` instalado como plugin, o cualquier binario en `/usr/local/bin` o instalado vía snap son candidatos típicos. Hay dos soluciones y conviene usar las dos:

**Rutas absolutas en el script:**

```bash
/usr/bin/docker exec tienda-db pg_dumpall -U postgres > "$DIR_TRABAJO/tienda-db.sql"
/snap/bin/gsutil cp "$ARCHIVO" "gs://$BUCKET/"
```

**O declarar el PATH al principio del crontab**, donde aplica a todas las tareas:

```cron
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

0 3 * * * /bin/bash /srv/scripts/backup.sh >> /srv/logs/backup.log 2>&1
```

Para averiguar la ruta de un comando: `which gsutil` desde tu sesión.

### 2. No hay variables de entorno

Todo lo que definas en `~/.bashrc` o exportes en tu sesión no existe para cron. Si el script depende de una variable, tiene que cargarla él mismo:

```bash
set -a
source /srv/env/backup.env
set +a
```

Esto afecta también a la autenticación de Docker con un [registro privado](Registros-de-Imagenes-Privados.md): las credenciales están en `~/.docker/config.json` del usuario, y cron corriendo como `root` busca en `/root/.docker/`. Un `docker pull` que funciona a mano puede fallar con `denied: Permission denied` en cron por este motivo exacto.

### 3. Las rutas relativas apuntan a otro sitio

Cron ejecuta con el directorio de trabajo en el *home* del usuario, no donde está el script. Un `cd` implícito no existe:

```bash
# ❌ MAL — depende de dónde se ejecute
docker compose up -d

# ✅ BIEN — explícito
cd /srv/docker/tienda && docker compose up -d
```

La regla general: **en un script de cron, todas las rutas son absolutas**. Sin excepciones.

## El `%` y otras rarezas de la sintaxis

En un crontab, el carácter `%` **no es literal**: cron lo interpreta como un salto de línea que se envía por la entrada estándar del comando. Esto rompe cualquier intento de usar `date` con formato directamente en la línea:

```cron
# ❌ MAL — el % rompe la línea
0 3 * * * tar -czf /srv/backups/backup_$(date +%Y%m%d).tar.gz /srv/docker

# ✅ BIEN — escapado
0 3 * * * tar -czf /srv/backups/backup_$(date +\%Y\%m\%d).tar.gz /srv/docker
```

Es un fallo silencioso y muy irritante. La solución más limpia es no meter lógica en el crontab: la línea debería llamar a un script y nada más. Dentro del script, bash se comporta con normalidad.

## Capturar la salida (y no perderla)

Esta parte del comando es la que separa una tarea diagnosticable de una caja negra:

```cron
0 3 * * * /bin/bash /srv/scripts/backup.sh >> /srv/logs/backup.log 2>&1
```

- **`>>`** añade al final del fichero. Con `>` se sobrescribiría en cada ejecución y solo tendrías la última.
- **`2>&1`** redirige la salida de error al mismo sitio que la salida normal. **El orden es obligatorio**: `2>&1 >> fichero` no hace lo mismo y es un error clásico. Sin esto, los mensajes de error se pierden — que son justo los que necesitas.

Prepara el fichero antes:

```bash
mkdir -p /srv/logs
touch /srv/logs/backup.log
chmod 644 /srv/logs/backup.log
```

Y ponle rotación, o crecerá sin límite. Con `logrotate`, que ya viene instalado:

```bash
nano /etc/logrotate.d/backup
```

```
/srv/logs/backup.log {
    weekly
    rotate 8
    compress
    missingok
    notifempty
}
```

Conserva ocho semanas comprimidas y no protesta si el fichero no existe todavía.

Cron también deja constancia de **que lanzó** la tarea (no de si funcionó) en el journal del sistema:

```bash
journalctl -u cron --since "today" | grep backup
```

```
Aug 25 03:00:01 vps CRON[48213]: (root) CMD (/bin/bash /srv/scripts/backup.sh >> /srv/logs/backup.log 2>&1)
```

Si esa línea no aparece, el problema está en el crontab. Si aparece pero el log de la tarea está vacío o con errores, el problema está en el script.

## Depurar sin esperar a mañana

Esperar 24 horas para ver si una tarea funciona no es viable. Estas dos técnicas lo resuelven.

**Reproducir el entorno de cron.** Ejecutar el script con un entorno vaciado revela los problemas de PATH y variables al instante:

```bash
env -i /bin/bash --noprofile --norc -c '/bin/bash /srv/scripts/backup.sh'
```

`env -i` arranca sin ninguna variable de entorno y `--noprofile --norc` impide que bash cargue los ficheros de inicio. Si funciona así, funcionará en cron.

**Programarlo para dentro de dos minutos.** La prueba definitiva, con el cron real:

```cron
*/2 * * * * /bin/bash /srv/scripts/backup.sh >> /srv/logs/backup.log 2>&1
```

Se observa el log, se confirma y se devuelve la línea a su horario. Poco elegante y muy eficaz.

## Evitar que dos ejecuciones se solapen

Si una tarea tarda más que su intervalo, cron lanza la siguiente igualmente y acaban compitiendo por los mismos ficheros. `flock` lo impide con un bloqueo:

```cron
0 3 * * * /usr/bin/flock -n /tmp/backup.lock /bin/bash /srv/scripts/backup.sh >> /srv/logs/backup.log 2>&1
```

`-n` significa "si el bloqueo ya está tomado, no esperes: sal inmediatamente". La ejecución que llega tarde se descarta en lugar de acumularse. Es una sola palabra en la línea y evita la clase de fallo que solo aparece el día que la base de datos ha crecido lo suficiente.

## Buenas prácticas avanzadas

- **Que el script sea idempotente y que compruebe sus condiciones.** Cron no reintenta ni sabe si la ejecución anterior funcionó. Un script que verifica que el contenedor existe, que hay espacio en disco y que el destino es accesible **antes** de empezar convierte un fallo a medias —el peor caso— en un fallo limpio del que se puede volver.
- **No dependas de que cron te avise.** Cron envía por correo local la salida de una tarea que escriba algo, pero en un VPS sin servidor de correo configurado ese mensaje va a `/var/mail/root` y nadie lo lee jamás. Para lo que importe, el aviso tiene que ser activo (una llamada a un webhook, un *dead man's switch*) desde el propio script.
- **Desplaza las tareas para que no coincidan.** Todo el mundo programa a las 3:00 en punto, y así el backup, la limpieza de imágenes y la renovación de certificados compiten a la vez. Repartirlas (`0 3`, `20 3`, `40 3`) evita picos de carga y, sobre todo, hace que cuando algo vaya mal sepas cuál fue.
- **Considera *systemd timers* para lo crítico.** Hacen lo mismo que cron y además registran cada ejecución en el journal con su código de salida y su duración, permiten reintentos, dependencias entre unidades y `Persistent=true` (recuperar una ejecución perdida porque el servidor estaba apagado). Cron no tiene nada de eso. Más verboso de configurar, mucho mejor de operar.
- **Ten el crontab en Git, no solo en el servidor.** `crontab -l > /srv/scripts/crontab.txt` versionado junto a los scripts documenta qué se ejecuta y permite reconstruirlo con `crontab /srv/scripts/crontab.txt`. Sin eso, la programación del servidor solo existe dentro del servidor — y desaparece con él justo cuando estás restaurando.

## Recursos didácticos

- [Crontab.guru](https://crontab.guru/) — escribes la expresión y te la traduce a lenguaje natural mostrando las próximas ejecuciones. Es la herramienta que hace innecesario memorizar el orden de los cinco campos, y despeja al instante dudas como la del OR entre día del mes y día de la semana.
- [ShellCheck](https://www.shellcheck.net/) — analiza el script que va a lanzar cron y detecta justo los fallos que en cron son fatales: rutas relativas, variables sin comillas, redirecciones mal ordenadas.
- [Explainshell](https://explainshell.com/) — útil para la línea completa del crontab, incluyendo `flock -n` y el `>> ... 2>&1`.

---

*En resumen: cron lanza el comando y se olvida — todo lo que falla viene de que su entorno no es el tuyo, así que rutas absolutas, salida redirigida a un log y un aviso activo cuando algo va mal.*
