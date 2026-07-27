# Copias de seguridad

## ¿Qué es?

Una copia de seguridad es un volcado de tus datos guardado **fuera del servidor que los produce**, con una antigüedad conocida y —esto es lo que casi nadie comprueba— la certeza de que se puede restaurar.

## ¿Por qué existe?

Un VPS es una sola máquina. Si su disco falla, si un comando borra lo que no debía o si alguien entra y cifra los datos, no hay ninguna réplica esperando. Los proveedores ofrecen *snapshots*, pero viven en la misma cuenta del mismo proveedor: no protegen frente a perder el acceso a esa cuenta ni frente a un borrado que se descubre tres semanas después.

Hay una segunda razón, menos dramática y más frecuente: el error humano con retraso. Alguien ejecuta una migración mal probada un martes y el lunes siguiente se descubre que lleva una semana corrompiendo registros. Sin copias con varios días de historia, no hay a dónde volver.

> Si has usado el control de versiones para recuperar un fichero que borraste hace tres commits, la idea es la misma aplicada a los datos: no basta con tener "la última copia", hace falta poder elegir un punto en el tiempo.

## ¿Cuándo y para qué se usa?

Desde el primer día que haya datos que no se puedan regenerar: la base de datos, los ficheros que suben los usuarios, la configuración del servidor. El código no entra en esta lista porque ya está en Git; las imágenes tampoco, porque están en el [registro](Registros-de-Imagenes-Privados.md).

---

## Qué copiar, y qué no

Antes de escribir un script conviene separar lo insustituible de lo reproducible:

| Dato | ¿Copia? | Por qué |
|---|---|---|
| Base de datos | **Sí** | Irrecuperable |
| Ficheros subidos por usuarios | **Sí** | Irrecuperable |
| `docker-compose.yml`, `vhost.d/` | Sí (o Git) | Reproducible, pero tenerlo acelera mucho |
| Ficheros `.env` con secretos | Sí, **cifrados aparte** | No pueden ir a Git |
| Certificados TLS | Opcional | Let's Encrypt los reemite |
| Imágenes Docker | No | Están en el registro |
| Logs | Depende | Solo si hay obligación de conservarlos |

Los `.env` son el caso incómodo: son imprescindibles para levantar el sistema y no pueden vivir en el repositorio. La solución habitual es un gestor de secretos, o cifrarlos con una clave que se custodie fuera del servidor.

## La regla 3-2-1

Es la referencia clásica y sigue siendo útil: **3** copias de los datos, en **2** soportes distintos, con **1** fuera de la ubicación principal.

En un VPS se traduce en algo muy concreto: los datos vivos en el servidor, una copia reciente en el disco local (rápida de restaurar) y una copia en almacenamiento de objetos de otro proveedor (que sobrevive a que el VPS desaparezca entero). Eso es lo que construye el script de esta ficha.

## Volcar una base de datos que está en un contenedor

El error de partida es copiar los ficheros de la base de datos mientras está en marcha: se obtiene una copia inconsistente que puede negarse a restaurar. Lo correcto es un **volcado lógico**, que pide a la propia base de datos que exporte su contenido en un estado coherente.

Como la base de datos corre en un contenedor, el comando se lanza con `docker exec`. Para PostgreSQL:

```bash
docker exec tienda-db pg_dumpall -U postgres > /srv/backups/tienda-db.sql
```

Fíjate en dónde está la redirección: `docker exec` escribe en su salida estándar y el `>` la captura **en el host**. El fichero se crea fuera del contenedor, que es lo que queremos.

Los equivalentes para otros motores:

```bash
# MySQL / MariaDB
docker exec tienda-db sh -c 'mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases' > /srv/backups/tienda-db.sql

# MongoDB
docker exec tienda-db sh -c 'mongodump --username "$MONGO_INITDB_ROOT_USERNAME" --password "$MONGO_INITDB_ROOT_PASSWORD" --archive' > /srv/backups/tienda-db.archive
```

Los dos últimos tienen un detalle deliberado que merece explicación. Las variables van **entre comillas simples**, dentro de un `sh -c`. Eso hace que las expanda la shell **de dentro del contenedor**, usando las variables de entorno que ya tiene la base de datos, en lugar de que la shell del host las escriba en el comando.

La diferencia importa, y mucho:

```bash
# ❌ MAL — la contraseña acaba en la lista de procesos del host
docker exec tienda-db mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases

# ✅ BIEN — la contraseña no sale del contenedor
docker exec tienda-db sh -c 'mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases'
```

Con la primera forma, cualquier usuario del servidor que ejecute `ps aux` durante el volcado ve la contraseña completa. Con la segunda, la variable ya existía dentro del contenedor y nunca se escribe en ninguna línea de comandos.

## El script completo

Con eso claro, este es un script de copia diaria que vuelca, comprime, sube y limpia. Va en `/srv/scripts/backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

FECHA=$(date +"%Y%m%d_%H%M")
DIR_TRABAJO="/srv/backups/$FECHA"
BUCKET="backups-tienda"

mkdir -p "$DIR_TRABAJO"

### Base de datos ###
if docker ps --format '{{.Names}}' | grep -q '^tienda-db$'; then
  docker exec tienda-db pg_dumpall -U postgres > "$DIR_TRABAJO/tienda-db.sql"
else
  echo "ERROR: el contenedor tienda-db no está en marcha" >&2
  exit 1
fi

### Ficheros subidos por usuarios ###
cp -r /srv/docker/tienda/uploads "$DIR_TRABAJO/uploads"

### Comprimir ###
tar -czf "/srv/backups/backup_$FECHA.tar.gz" -C /srv/backups "$FECHA"
rm -rf "$DIR_TRABAJO"

### Subir fuera del servidor ###
gsutil cp "/srv/backups/backup_$FECHA.tar.gz" "gs://$BUCKET/"

### Limpiar copias locales de más de 7 días ###
find /srv/backups -name 'backup_*.tar.gz' -type f -mtime +7 -delete

echo "Copia completada: backup_$FECHA.tar.gz"
```

Las decisiones que hacen que este script sea fiable:

- **`set -euo pipefail`** es la línea más importante y la que casi nunca está. `-e` aborta al primer comando que falle, `-u` trata como error usar una variable no definida y `-o pipefail` hace que una tubería falle si falla cualquiera de sus tramos. Sin esto, un `pg_dumpall` que falla deja un fichero vacío, el script sigue adelante, lo comprime, lo sube y **termina con éxito**. Tendrías 30 días de copias de cero bytes y un `echo $?` diciendo `0`.
- **La comprobación de que el contenedor existe** con salida de error. Si la base de datos está parada, quieres que el backup falle ruidosamente, no que genere un fichero vacío. El `^...$` en el `grep` evita que `tienda-db` coincida con `tienda-db-replica`.
- **Las variables entre comillas** (`"$DIR_TRABAJO"`). Sin ellas, una ruta con espacios rompe el script de formas creativas.
- **`tar -C /srv/backups "$FECHA"`** guarda rutas relativas dentro del archivo. Sin el `-C`, el `.tar.gz` contendría la ruta absoluta y al restaurar sobrescribiría directorios reales.
- **`find ... -mtime +7 -delete`** limita el uso de disco local. El `-name 'backup_*.tar.gz'` es la red de seguridad: sin ese filtro, un `find /srv/backups -type f -delete` mal escrito borra cualquier cosa que hubiera ahí.

Y lo que **no** está en el script, deliberadamente: ninguna contraseña. El script original del que suele partir esta receta lleva las credenciales de la base de datos escritas dentro, y ese fichero acaba copiado, versionado o leído por cualquiera con acceso al servidor. Si necesitas credenciales, cárgalas de un fichero aparte con permisos restringidos:

```bash
set -a
source /srv/env/backup.env      # chmod 600, propietario root
set +a
```

`set -a` hace que todo lo que se defina a continuación se exporte como variable de entorno, y `set +a` lo desactiva. Y `/srv/env/backup.env` nunca entra en el repositorio, por los motivos que detalla [Por qué los secretos no van a Git](../../seguridad/gestion-de-secretos-en-desarrollo/Por-Que-Los-Secretos-No-Van-A-Git.md).

Dale permisos de ejecución y pruébalo a mano antes de automatizar nada:

```bash
chmod +x /srv/scripts/backup.sh
/srv/scripts/backup.sh
echo $?
```

```
Copia completada: backup_20250825_0300.tar.gz
0
```

Un `0` significa éxito. Cualquier otro número es un fallo que hay que resolver antes de programarlo en [cron](Tareas-Programadas-con-Cron.md).

## Sacar la copia del servidor

Una copia en `/srv/backups` no protege de nada que le pase al servidor. El destino habitual es almacenamiento de objetos: barato, con reglas de retención propias y en otra infraestructura.

Crear el contenedor de destino, una sola vez:

```bash
gsutil mb -l europe-west1 gs://backups-tienda/
```

La orden de subida ya está en el script. Comprueba que llega:

```bash
gsutil ls -lh gs://backups-tienda/
```

```
 48.2 MiB  2025-08-25T03:00:41Z  gs://backups-tienda/backup_20250825_0300.tar.gz
 47.9 MiB  2025-08-24T03:00:38Z  gs://backups-tienda/backup_20250824_0300.tar.gz
```

El comando concreto cambia según el proveedor (`aws s3 cp`, `az storage blob upload`, `rclone copy`), pero el patrón es idéntico.

## Retención: que la borre el proveedor, no tu script

El script borra las copias **locales** de más de siete días. Las remotas conviene que las borre el propio almacenamiento, mediante una regla de ciclo de vida:

```bash
nano /srv/scripts/gcs-lifecycle.json
```

```json
{
  "rule": [
    {
      "action": {"type": "Delete"},
      "condition": {"age": 30}
    }
  ]
}
```

```bash
gsutil lifecycle set /srv/scripts/gcs-lifecycle.json gs://backups-tienda
gsutil lifecycle get gs://backups-tienda
```

Por qué en el proveedor y no en el script: **la limpieza sigue funcionando aunque el VPS esté apagado, caído o comprometido**. Si el borrado dependiera de un script en el servidor y el servidor deja de funcionar, o bien se acumulan copias indefinidamente, o bien —si alguien entró— tiene la posibilidad de borrarlas todas. La regla del lado del servidor es independiente.

Elige el plazo pensando en cuánto tarda un problema en detectarse. Siete días es corto: una corrupción de datos que se descubre a las dos semanas se queda sin copia sana. Treinta días es un mínimo razonable, y una escalera (diarias 30 días, semanales 6 meses) cuesta poco más.

## Restaurar: la parte que nadie prueba

Un backup que no se ha restaurado nunca no es un backup, es un fichero. Restaurar tiene sus propios fallos: falta una extensión de PostgreSQL, la versión del motor no coincide, el volcado está truncado.

El procedimiento completo:

```bash
# 1. Traer la copia
gsutil cp gs://backups-tienda/backup_20250825_0300.tar.gz /tmp/

# 2. Descomprimir
tar -xzf /tmp/backup_20250825_0300.tar.gz -C /tmp/

# 3. Cargar el volcado en la base de datos
cat /tmp/20250825_0300/tienda-db.sql | docker exec -i tienda-db psql -U postgres
```

El `-i` de `docker exec` es imprescindible: mantiene abierta la entrada estándar para que el fichero entre por ella. Sin él, `psql` no recibe nada y el comando termina sin error aparente habiendo restaurado cero.

**Hazlo en un entorno de pruebas, no sobre producción.** Una restauración es destructiva: si el volcado está mal, te quedas sin lo que había y sin lo que querías recuperar.

Una verificación mínima que puede ir en el propio script, para detectar volcados truncados sin llegar a restaurar:

```bash
tail -1 /tmp/20250825_0300/tienda-db.sql
```

```
-- PostgreSQL database dump complete
```

`pg_dumpall` escribe esa línea solo si terminó bien. Si no está, el volcado se cortó a la mitad y esa copia no sirve.

## Buenas prácticas avanzadas

- **Prueba la restauración en el calendario, no cuando haga falta.** Una restauración completa a un entorno limpio cada trimestre es lo que separa "tenemos backups" de "sabemos recuperarnos". Es donde aparecen los fallos que ninguna comprobación automática detecta: la extensión de PostgreSQL que no está en la imagen nueva, el `.env` que nadie copió, el orden en que hay que levantar los servicios.
- **Vigila el éxito con un *dead man's switch*.** Un backup que falla en silencio es peor que no tenerlo, porque genera confianza infundada. La monitorización normal avisa cuando algo pasa; aquí necesitas lo contrario: un servicio externo que espera un aviso diario y **te alerta si no llega**. Así detectas también el caso en que el servidor está tan roto que ni siquiera puede avisar.
- **Comprueba el tamaño, no solo el código de salida.** Un volcado que baja de 200 MB a 2 MB de un día para otro es una señal clarísima de que algo va mal, y el script termina con éxito igualmente. Comparar el tamaño con el de la copia anterior y avisar si cae más de un 50 % cuesta tres líneas y detecta el fallo silencioso más común.
- **Que la credencial de subida no pueda borrar.** Si el token que usa el servidor tiene permiso de borrado, quien comprometa el servidor puede vaciar el bucket antes de cifrar los datos — que es exactamente el manual de un ataque de *ransomware*. Un permiso de solo escritura, combinado con versionado de objetos o retención inmutable en el bucket, hace que las copias sobrevivan al ataque.
- **Silencia el trabajo en horario de mantenimiento, pero no lo hagas coincidir con otras tareas.** Un `pg_dumpall` compite por CPU y disco con lo que esté corriendo. Programarlo a las tres de la mañana está bien; programarlo a la misma hora que la limpieza de imágenes y la renovación de certificados hace que las tres vayan lentas y que, cuando algo falle, no sepas cuál fue.

## Recursos didácticos

- [Crontab.guru](https://crontab.guru/) — para cuando programes el script: escribes la expresión y te dice en español cuándo se ejecutará y las próximas fechas.
- [ShellCheck](https://www.shellcheck.net/) — pegas el script de bash y te señala los errores que rompen los backups en producción: variables sin comillas, comparaciones mal escritas, tuberías cuyo fallo se ignora. Pásalo por aquí antes de programarlo.
- [Explainshell](https://explainshell.com/) — desmenuza los `find -mtime +7 -delete` y `tar -czf -C` de esta ficha, que son los dos comandos donde una opción mal puesta borra lo que no debía.

---

*En resumen: una copia que nunca se ha restaurado no es una copia — automatiza el volcado, sácalo del servidor, deja que el proveedor gestione la retención y prueba la vuelta atrás antes de necesitarla.*
