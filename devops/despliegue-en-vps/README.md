# Despliegue en un VPS — Guía de tecnologías

Cómo llevar una aplicación dockerizada a un servidor Linux propio y dejarla funcionando de forma segura: desde el primer `ssh root@...` hasta las copias de seguridad automáticas, pasando por el reverse proxy, los certificados HTTPS y el acceso restringido por VPN.

Está pensada para perfiles backend que saben construir una imagen Docker pero nunca han administrado el servidor donde se ejecuta. No presupone experiencia previa en administración de sistemas: cada pieza se explica desde qué problema resuelve, con el comando exacto y qué devuelve.

Todas las fichas usan el mismo ejemplo de principio a fin —una tienda online con frontend, API y base de datos, servida en `tienda.ejemplo.com` desde un VPS en `203.0.113.10`— para que las piezas encajen entre documentos.

---

## Orden de lectura recomendado

Sigue este orden si partes de un VPS recién contratado. Cada bloque deja el servidor en un estado utilizable y el siguiente se apoya en él.

### 1. El servidor: primeros pasos y cierre de puertas

Antes de desplegar nada, el servidor tiene que dejar de ser una máquina abierta a internet. Este bloque es el que evita que te lo tomen prestado la primera semana.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [VPS](VPS.md) | Qué te dan y qué tienes que poner tú, los primeros diez minutos y la estructura de directorios sobre la que se apoya todo lo demás. |
| 2 | [Acceso SSH seguro](Acceso-SSH-Seguro.md) | Cambiar la contraseña por una clave y cerrar el acceso de `root`. El primer cambio real de seguridad. |
| 3 | [UFW](UFW.md) | El firewall: dejar accesibles solo los puertos que hacen falta, y por qué Docker se lo salta. |
| 4 | [Fail2ban](Fail2ban.md) | Bloquear automáticamente a quien insiste con credenciales fallidas. |

### 2. Docker en producción

El servidor ya es seguro; ahora hay que convertirlo en algo capaz de ejecutar contenedores sin llenarse de logs ni quedarse parado tras un reinicio.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 5 | [Docker en un VPS](Docker-en-un-VPS.md) | Instalarlo bien, que arranque solo y los dos ajustes de `daemon.json` que evitan un disco lleno. |
| 6 | [Reverse proxy con nginx-proxy](Reverse-Proxy-con-nginx-proxy.md) | La pieza que reparte el tráfico entre aplicaciones y se configura sola con una variable de entorno. |
| 7 | [HTTPS con Let's Encrypt](HTTPS-con-Lets-Encrypt.md) | Certificados emitidos y renovados automáticamente. Depende del proxy anterior. |
| 8 | [Redes Docker y exposición de servicios](Redes-Docker-y-Exposicion-de-Servicios.md) | Qué contenedor puede hablar con cuál: el patrón de red pública + red interna. |
| 9 | [Volúmenes y permisos](Volumenes-y-Permisos.md) | Dónde viven los datos que sobreviven al despliegue, y el `Permission denied` que aparece la primera vez. |
| 10 | [Registros de imágenes privados](Registros-de-Imagenes-Privados.md) | Cómo el servidor se autentica y descarga tu imagen en vez de construirla. |

### 3. Que siga funcionando solo

Lo que separa un servidor que funciona hoy de uno que sigue funcionando dentro de seis meses.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 11 | [Copias de seguridad](Copias-de-Seguridad.md) | Volcar la base de datos, sacar la copia del servidor y comprobar que se puede restaurar. |
| 12 | [Tareas programadas con cron](Tareas-Programadas-con-Cron.md) | Automatizar lo anterior, y las tres razones por las que un script que funciona a mano falla en cron. |

### 4. Acceso privado (opcional)

Cerrar del todo la administración: que el puerto de SSH y los paneles internos dejen de existir para internet.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 13 | [WireGuard](WireGuard.md) | La VPN que permite retirar SSH de internet y exponer servicios solo hacia dentro. |
| 14 | [DNS interno con dnsmasq](DNS-Interno-con-dnsmasq.md) | Que el mismo dominio resuelva a la IP del túnel para quien administra. Requiere la VPN anterior. |

---

## Índice completo por bloque

<details>
<summary>Ver todos los archivos</summary>

**El servidor**
- [VPS](VPS.md)
- [Acceso SSH seguro](Acceso-SSH-Seguro.md)
- [UFW](UFW.md)
- [Fail2ban](Fail2ban.md)

**Docker en producción**
- [Docker en un VPS](Docker-en-un-VPS.md)
- [Reverse proxy con nginx-proxy](Reverse-Proxy-con-nginx-proxy.md)
- [HTTPS con Let's Encrypt](HTTPS-con-Lets-Encrypt.md)
- [Redes Docker y exposición de servicios](Redes-Docker-y-Exposicion-de-Servicios.md)
- [Volúmenes y permisos](Volumenes-y-Permisos.md)
- [Registros de imágenes privados](Registros-de-Imagenes-Privados.md)

**Operación**
- [Copias de seguridad](Copias-de-Seguridad.md)
- [Tareas programadas con cron](Tareas-Programadas-con-Cron.md)

**Acceso privado**
- [WireGuard](WireGuard.md)
- [DNS interno con dnsmasq](DNS-Interno-con-dnsmasq.md)

</details>

---

## Lista de verificación del servidor

Los comandos que responden "¿está esto funcionando?" para cada pieza. Útiles al terminar el montaje y cuando algo va mal.

| Pieza | Comando |
|---|---|
| Firewall | `ufw status verbose` |
| Fail2ban | `fail2ban-client status sshd` |
| Docker | `docker ps` |
| Reverse proxy | `docker logs nginx-proxy --tail 20` |
| Certificados | `docker logs nginx-proxy-letsencrypt --tail 20` |
| Caducidad del certificado | `echo \| openssl s_client -connect tienda.ejemplo.com:443 -servername tienda.ejemplo.com 2>/dev/null \| openssl x509 -noout -dates` |
| Copia de seguridad | `tail -20 /srv/logs/backup.log` |
| Tareas programadas | `sudo crontab -l` |
| VPN | `wg show` |
| DNS interno | `dig @10.10.0.1 tienda.ejemplo.com +short` |

## Dónde se aplica cada cambio

Cuál es el comando correcto después de tocar cada fichero. Casi todos evitan reiniciar contenedores.

| Qué has cambiado | Cómo se aplica |
|---|---|
| `vhost.d/<dominio>` del proxy | `docker exec nginx-proxy nginx -t && docker exec nginx-proxy nginx -s reload` |
| `docker-compose.yml` de una app | `cd /srv/docker/tienda && docker compose up -d` |
| `/etc/docker/daemon.json` | `systemctl restart docker` (reinicia los contenedores) |
| Reglas de UFW | `ufw reload` |
| `jail.local` de Fail2ban | `fail2ban-client reload` |
| `/etc/wireguard/wg0.conf` | `systemctl restart wg-quick@wg0` |
| `/etc/dnsmasq.conf` | `systemctl restart dnsmasq` |
| `/etc/ssh/sshd_config` | `sshd -t && systemctl restart ssh` |

---

> Piezas relacionadas en otras colecciones: [Docker](../docker/README.md) para escribir el `Dockerfile` que da lugar a la imagen, [GitHub Container Registry](../docker/GitHub-Container-Registry.md) para publicarla desde CI, [CI/CD](../ci-cd/README.md) para automatizar el despliegue, [Observabilidad](../observabilidad/README.md) para saber qué pasa dentro una vez está en marcha, y [SSH](../../redes/redes-y-acceso-remoto/SSH.md) y [VPN](../../redes/redes-y-acceso-remoto/VPN.md) para los fundamentos de las dos formas de acceso remoto que aquí se dan por sabidas.
