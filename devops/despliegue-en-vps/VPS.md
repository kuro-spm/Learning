# VPS

## ¿Qué es?

Un **VPS** (*Virtual Private Server*) es una máquina Linux virtual que alquilas por meses y administras entera: entras por SSH como `root`, instalas lo que quieras y nadie decide por ti qué software corre dentro.

## ¿Por qué existe?

Entre "tengo un servidor físico en un armario" y "subo el código y que la plataforma se encargue" hay un hueco enorme. El servidor físico obliga a comprar hardware, alimentarlo y sustituirlo cuando se rompe. Las plataformas gestionadas (PaaS) te quitan ese problema, pero a cambio te imponen su forma de desplegar, su catálogo de servicios y su factura, que crece rápido en cuanto necesitas una base de datos, almacenamiento y un par de entornos.

El VPS ocupa el punto intermedio: el proveedor se encarga del hardware, la red y la corriente; tú te encargas del sistema operativo hacia arriba. Pagas una tarifa plana pequeña y predecible, y a cambio asumes el trabajo de administrarlo.

> Si vienes de desplegar en un App Service o en Heroku, piensa en un VPS como "el mismo servidor, pero sin que nadie te haya configurado nada": ni firewall, ni certificados, ni actualizaciones automáticas, ni copias de seguridad. Todo eso pasa a ser tuyo.

## ¿Cuándo y para qué se usa?

El VPS es la opción natural cuando el proyecto es pequeño o mediano y quieres controlar el coste: la tienda online de una empresa, el ERP interno de una pyme, un blog con su base de datos, el panel de administración de una app móvil, o el entorno de preproducción de cualquiera de ellos.

Encaja especialmente bien cuando el despliegue ya está **dockerizado**, porque entonces el VPS no necesita saber nada de tu stack: no instalas .NET, ni Node, ni PostgreSQL en el sistema. Instalas Docker una vez y a partir de ahí todo son contenedores. El servidor se convierte en una pieza aburrida y reemplazable, que es exactamente lo que quieres de un servidor.

Deja de encajar cuando necesitas escalar a varias máquinas de forma automática, alta disponibilidad real (si el VPS cae, tu servicio cae) o cumplir certificaciones que exigen infraestructura gestionada.

---

## Lo que te dan y lo que tienes que poner tú

Cuando contratas un VPS en un proveedor (Hetzner, DigitalOcean, OVH, Scaleway, Contabo...) eliges tres cosas: **recursos** (CPU, RAM, disco), **región** (dónde está físicamente la máquina) e **imagen del sistema** (normalmente Debian o Ubuntu). En unos segundos tienes una IP pública y una forma de entrar.

Lo que te entregan es deliberadamente mínimo: un Linux recién instalado, con el usuario `root`, el puerto SSH abierto y **nada más configurado**. En concreto, no traen:

| No viene configurado | Lo tienes que hacer tú |
|---|---|
| Firewall | [UFW](UFW.md) |
| Protección contra fuerza bruta | [Fail2ban](Fail2ban.md) |
| Acceso sin contraseña | [Claves SSH](Acceso-SSH-Seguro.md) |
| Certificados HTTPS | [Let's Encrypt](HTTPS-con-Lets-Encrypt.md) |
| Copias de seguridad | [Script + almacenamiento externo](Copias-de-Seguridad.md) |
| Actualizaciones | `apt upgrade`, tú |

Un VPS recién creado con la contraseña de `root` por defecto y el puerto 22 abierto recibe intentos de acceso automatizados **en cuestión de minutos**. No es una exageración: los rangos de IP de los proveedores conocidos se escanean continuamente. Por eso el orden de los primeros pasos importa.

## Los primeros diez minutos

Este es el orden recomendado. Cada paso deja el servidor un poco menos expuesto que el anterior, y ninguno depende de que hayas desplegado nada todavía.

**1. Entrar por primera vez.** El proveedor te da una IP y, o bien una contraseña de `root`, o bien la posibilidad de pegar tu clave pública al crear la máquina. Si puedes elegir, elige la clave: te ahorra el paso de migrar después.

```bash
ssh root@203.0.113.10
```

La primera vez, SSH te pedirá confirmar la huella del servidor (`Are you sure you want to continue connecting?`). Al aceptar, esa huella se guarda en `~/.ssh/known_hosts` y en conexiones futuras se comprueba automáticamente: si algún día cambia, SSH avisará con un error muy visible.

**2. Actualizar el sistema antes de instalar nada.** La imagen del proveedor puede llevar semanas sin refrescarse, así que arrastra paquetes con vulnerabilidades ya parcheadas.

```bash
apt update && apt upgrade -y
```

`apt update` refresca la lista de paquetes disponibles (no instala nada) y `apt upgrade -y` instala las versiones nuevas de lo que ya está instalado, respondiendo "sí" a las confirmaciones. Es el único comando de esta guía que conviene repetir de forma periódica durante toda la vida del servidor.

**3. Poner la zona horaria correcta.** Parece cosmético y no lo es: las entradas de log, los certificados y las tareas programadas se apoyan en la hora del sistema. Un servidor en UTC con un backup programado "a las 3" se ejecuta a las 4 o a las 5 de la mañana local según la época del año.

```bash
timedatectl set-timezone Europe/Madrid
timedatectl                              # verifica el cambio
```

La salida de `timedatectl` confirma la zona y si el reloj está sincronizado:

```
               Local time: Mon 2025-08-25 11:42:03 CEST
           Time zone: Europe/Madrid (CEST, +0200)
System clock synchronized: yes
```

**4. Crear un usuario que no sea `root`.** Trabajar siempre como `root` significa que cualquier error tipográfico se ejecuta con permisos totales, y que si alguien roba esa credencial entra directamente con control absoluto. La convención es crear un usuario de despliegue y darle acceso a `sudo`.

```bash
adduser deployer              # pide contraseña y datos; los datos se pueden dejar en blanco
usermod -aG sudo deployer     # -aG = añadir (append) al grupo, sin quitarle los que ya tenga
```

Ojo con la `-a`: `usermod -G sudo deployer` (sin la `a`) **sustituye** todos los grupos del usuario por `sudo` en lugar de añadirlo, y es una forma clásica de dejarse fuera de grupos que hacían falta.

Antes de cerrar la sesión de `root`, abre **otra** terminal y comprueba que el usuario nuevo entra y puede usar `sudo`:

```bash
ssh deployer@203.0.113.10
sudo whoami        # debe responder: root
```

Esta comprobación en una segunda terminal es la red de seguridad de todo lo que viene después: mientras la sesión de `root` siga abierta, siempre puedes deshacer un cambio que te haya dejado fuera.

**5. Cerrar la puerta.** A partir de aquí ya toca configurar el acceso por clave y desactivar el login de `root` ([Acceso SSH seguro](Acceso-SSH-Seguro.md)), levantar el firewall ([UFW](UFW.md)) y activar la protección contra fuerza bruta ([Fail2ban](Fail2ban.md)).

## Una estructura de directorios para no perderte

En un servidor que solo ejecuta contenedores, casi todo lo que escribes a mano son ficheros de configuración, volúmenes y scripts. Si los dejas repartidos por `/home`, `/opt` y `/root`, al cabo de unos meses nadie sabe dónde está qué — y menos aún el script de backup.

La convención de Linux para "datos servidos por esta máquina" es `/srv`, y funciona muy bien como raíz única:

```bash
mkdir -p /srv/{docker,logs,backups,scripts,certs,env}
chmod -R 755 /srv
```

La sintaxis `{a,b,c}` es *brace expansion* de la shell: crea las seis carpetas de una vez, y `-p` evita el error si alguna ya existe. `chmod 755` deja que el propietario escriba y que el resto solo lea y entre en los directorios.

Cada carpeta tiene un cometido claro:

| Carpeta | Qué guarda |
|---|---|
| `/srv/docker/<app>/` | El `docker-compose.yml` de cada aplicación y sus volúmenes |
| `/srv/logs/` | Logs que los contenedores escriben en el host |
| `/srv/backups/` | Copias de seguridad antes de subirlas fuera |
| `/srv/scripts/` | Scripts de mantenimiento que ejecuta cron |
| `/srv/certs/` | Certificados TLS |
| `/srv/env/` | Ficheros `.env` con secretos, fuera del árbol de la aplicación |

Con esto, una aplicación llamada `tienda` vive entera en `/srv/docker/tienda/`, y el resto del servidor sigue siendo el Linux que instaló el proveedor. Ese es el objetivo: que un servidor nuevo se pueda reconstruir copiando `/srv` y ejecutando `docker compose up -d`.

> Los ficheros de `/srv/env/` contienen contraseñas. Restringe sus permisos con `chmod 600` para que solo su propietario pueda leerlos, y **nunca** los subas a Git.

## Cómo se dimensiona (y cómo se corrige si te quedas corto)

La duda habitual al contratar es cuánta RAM hace falta. Con contenedores, la regla práctica es sumar lo que consume cada pieza en reposo y dejar margen:

- Sistema + Docker: ~500 MB
- Un reverse proxy: ~50 MB
- Una API (.NET, Node, Python): 150–400 MB
- Una base de datos relacional pequeña: 300 MB–1 GB
- Compilar imágenes en el propio servidor: **picos de 1–2 GB**

Ese último punto es el que suele pillar por sorpresa: construir la imagen en el VPS puede necesitar más memoria que ejecutarla. La solución habitual no es contratar más RAM, sino **construir las imágenes fuera** (en CI) y que el servidor solo haga `docker pull` desde un [registro privado](Registros-de-Imagenes-Privados.md).

Para ver el consumo real:

```bash
free -h                # memoria total, usada y disponible
df -h /                # espacio en disco
docker stats --no-stream   # consumo por contenedor, una foto fija
```

`free -h` muestra la columna `available`, que es la que importa (no `free`: Linux usa la memoria libre como caché de disco y la cede cuando hace falta). Un servidor sano tiene `available` holgado:

```
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       1.5Gi       380Mi        12Mi       1.9Gi       2.1Gi
```

Casi todos los proveedores permiten **ampliar CPU y RAM** de un VPS existente con un reinicio, así que empezar pequeño es razonable. El disco, en cambio, suele poder crecer pero no encoger: ahí sí conviene no quedarse justo.

## Buenas prácticas avanzadas

- **Trata el servidor como reconstruible, no como una mascota.** Si el único sitio donde existe la configuración es el propio VPS, un fallo de disco se lleva por delante meses de ajustes que nadie recuerda. Mantén los `docker-compose.yml`, los `vhost.d/` y los scripts en un repositorio Git (sin los `.env`), y valida de vez en cuando que un servidor limpio se puede levantar solo con ese repositorio más una restauración de datos. Es la diferencia entre una caída de dos horas y una de dos días.
- **Activa las actualizaciones de seguridad desatendidas.** Nadie ejecuta `apt upgrade` cada semana de forma sostenida. `apt install unattended-upgrades` deja el sistema aplicando solo los parches del repositorio de seguridad, que es justo el subconjunto que casi nunca rompe nada. Sin esto, el servidor va acumulando CVEs conocidos en silencio.
- **Añade *swap* aunque tengas RAM de sobra.** Muchos VPS vienen sin swap, y sin ella el *OOM killer* de Linux mata el proceso más grande —normalmente tu base de datos— en cuanto hay un pico. Un fichero de swap del tamaño de la RAM convierte un corte abrupto en una degradación lenta que te da tiempo a reaccionar: `fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile`, más la línea correspondiente en `/etc/fstab` para que sobreviva a los reinicios.
- **No confundas los *snapshots* del proveedor con copias de seguridad.** Un snapshot es una foto del disco entero, vive en la misma cuenta del mismo proveedor y sirve para volver atrás tras un cambio fallido. Si pierdes acceso a la cuenta, o el borrado fue hace tres semanas, no te salva. Son complementarios a un backup de datos que sale del proveedor (ver [Copias de seguridad](Copias-de-Seguridad.md)).
- **Reinicia a propósito, en horario elegido.** Un servidor que lleva 400 días sin reiniciar no es un logro: es un servidor donde nadie ha comprobado que los servicios arrancan solos. Después de configurar todo, haz un `reboot` deliberado y verifica que Docker, los contenedores y el proxy vuelven sin tocar nada. Es mejor descubrir un `systemctl enable` que faltaba un martes por la mañana que durante un corte de luz.

## Recursos didácticos

- [Explainshell](https://explainshell.com/) — pegas cualquier comando de esta guía (`mkdir -p /srv/{docker,logs}`, `usermod -aG sudo deployer`) y te descompone cada opción con la página del manual correspondiente. Es la forma más rápida de dejar de copiar comandos a ciegas.
- [Linux Journey](https://linuxjourney.com/) — recorrido corto y ordenado por los fundamentos de Linux (permisos, procesos, red, systemd), pensado para quien ya programa pero nunca administró una máquina.
- [DigitalOcean Community Tutorials](https://www.digitalocean.com/community/tutorials) — guías de administración de servidores muy bien escritas y actualizadas; sirven aunque tu VPS esté en otro proveedor.

---

*En resumen: un VPS es un Linux vacío con IP pública — el proveedor te da la máquina, y todo lo que la convierte en un servidor de producción lo pones tú.*
