# Fail2ban

## ¿Qué es?

Fail2ban es un servicio que lee los logs del sistema buscando intentos de acceso fallidos y, cuando una misma IP acumula demasiados en poco tiempo, la bloquea en el firewall durante un rato.

## ¿Por qué existe?

Un [firewall](UFW.md) decide qué puertos están abiertos, pero no distingue entre quien entra bien y quien lleva mil intentos fallidos. Si el puerto 22 está abierto, está abierto para todos por igual, y un atacante puede probar credenciales sin límite y sin coste.

Fail2ban añade la dimensión que le falta al firewall: el **comportamiento a lo largo del tiempo**. En lugar de mirar paquetes sueltos, lee lo que los servicios escriben en sus logs, detecta el patrón "esta IP falla una y otra vez" y reacciona añadiendo una regla temporal de bloqueo.

> Si has implementado alguna vez un *rate limiting* en una API —bloquear a quien haga más de N peticiones por minuto—, Fail2ban es exactamente eso, pero a nivel de sistema operativo y alimentándose de ficheros de log en lugar de un contador en memoria.

## ¿Cuándo y para qué se usa?

Su caso principal es SSH, porque es el servicio que todo servidor con IP pública tiene expuesto y el que reciben los ataques automatizados por defecto. Pero funciona con cualquier servicio que registre sus fallos en un log: un panel de administración, un servidor de correo, un formulario de login que escriba los intentos fallidos.

Una aclaración importante sobre su alcance: **si ya has desactivado la autenticación por contraseña en SSH** (ver [Acceso SSH seguro](Acceso-SSH-Seguro.md)), la fuerza bruta contra SSH deja de ser un riesgo real, porque no hay contraseña que adivinar. Fail2ban sigue aportando en ese escenario, pero por otro motivo: reduce drásticamente el ruido de los logs y el consumo de CPU de atender miles de conexiones basura. No es un sustituto de las claves SSH; es un complemento.

---

## Cómo funciona por dentro

Fail2ban se apoya en tres piezas que conviene tener claras antes de tocar la configuración, porque los nombres aparecen en todos los ficheros:

- **Filtro** (*filter*): una colección de expresiones regulares que reconocen "esto es un intento fallido" en el formato de log de un servicio concreto. Vienen escritas de fábrica para los servicios habituales.
- **Cárcel** (*jail*): la unidad de configuración. Une un filtro con un log que vigilar, unos umbrales y una acción. Una jail por servicio.
- **Acción**: qué hacer cuando se supera el umbral. Por defecto, añadir una regla de bloqueo al firewall; también puede enviar un correo.

El ciclo completo es: el servicio escribe un fallo en su log → el filtro lo reconoce → Fail2ban cuenta cuántos lleva esa IP dentro de la ventana de tiempo → si supera el umbral, ejecuta la acción → pasado el tiempo de bloqueo, retira la regla.

## Instalación y el fichero que hay que editar

```bash
apt install fail2ban -y
```

Y aquí viene la única convención que hay que respetar sí o sí. Fail2ban trae su configuración en `/etc/fail2ban/jail.conf`, pero **ese fichero no se toca**: las actualizaciones del paquete lo sobrescriben y se llevan por delante tus cambios sin avisar. Lo que se lee es un fichero `jail.local` que tiene prioridad y del que solo hace falta escribir lo que quieras cambiar.

```bash
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
nano /etc/fail2ban/jail.local
```

Copiar el `.conf` entero como punto de partida es cómodo porque deja todas las opciones a la vista comentadas. Si prefieres algo más limpio, un `jail.local` de cinco líneas también funciona: lo que no esté escrito se hereda del `.conf`.

## Configurar la cárcel de SSH

Dentro de `jail.local`, busca la sección `[sshd]` y déjala así:

```ini
[sshd]
enabled  = true
port     = 22
maxretry = 5
findtime = 10m
bantime  = 1h
```

Qué significa cada valor:

- `enabled = true` — activa la jail. En muchas distribuciones viene desactivada por defecto, y este es el motivo número uno de "instalé Fail2ban y no bloquea nada".
- `port = 22` — el puerto que se bloqueará. Si cambiaste el puerto de SSH, cámbialo aquí también o los bloqueos no servirán de nada.
- `maxretry = 5` — cuántos fallos hacen falta para bloquear.
- `findtime = 10m` — la ventana en la que se cuentan esos fallos. Cinco fallos en diez minutos bloquean; cinco fallos repartidos en tres horas, no.
- `bantime = 1h` — cuánto dura el bloqueo. Con `-1` el bloqueo es permanente, lo cual suena atractivo pero llena la tabla del firewall de decenas de miles de reglas con el tiempo.

Los valores de arriba son un punto de partida razonable: lo bastante tolerante para que no te bloquees tú por teclear mal la passphrase, y lo bastante estricto para frenar un ataque.

Los mismos parámetros se pueden poner en una sección `[DEFAULT]` al principio del fichero para que apliquen a todas las jails, y sobrescribirlos solo donde haga falta.

## Arrancar y comprobar que funciona

```bash
systemctl enable fail2ban    # que arranque solo tras un reinicio
systemctl start fail2ban
```

El comando que de verdad importa es el que consulta el estado de una jail concreta:

```bash
fail2ban-client status sshd
```

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 2
|  |- Total failed:     1847
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 3
   |- Total banned:     212
   `- Banned IP list:   192.0.2.44 198.51.100.7 203.0.113.201
```

Cómo leer esta salida:

- `File list` confirma **qué log está vigilando**. Si aparece vacío o apunta a un fichero que no existe, la jail no está viendo nada.
- `Total failed` alto con `Total banned` a cero significa que el filtro reconoce los fallos pero la acción no se aplica: casi siempre un problema del *backend* del firewall.
- `Currently banned` es el número de IPs bloqueadas ahora mismo.

Que `Total failed` suba solo a los pocos minutos de arrancar el servidor no es señal de que te estén atacando a ti en particular: es el escaneo de fondo permanente de internet.

Para ver la lista de todas las jails activas:

```bash
fail2ban-client status
```

## Desbloquear una IP (probablemente la tuya)

Antes o después te bloquearás a ti mismo. Con la sesión abierta o desde otra red:

```bash
fail2ban-client set sshd unbanip 198.51.100.25
```

Y para evitar que vuelva a pasar, la lista blanca. En `jail.local`, dentro de `[DEFAULT]`:

```ini
ignoreip = 127.0.0.1/8 ::1 198.51.100.25 10.10.0.0/24
```

Se admiten IPs sueltas, rangos CIDR y nombres de host. En el ejemplo se ignoran localhost, la IP fija de la oficina y toda la subred de la [VPN](WireGuard.md). Tras editarlo:

```bash
fail2ban-client reload
```

`reload` recarga la configuración sin perder los bloqueos activos; `systemctl restart fail2ban` los descarta todos.

> Añadir tu IP a `ignoreip` solo es buena idea si es fija. Con IP dinámica, la que hoy es tuya mañana puede ser de otra persona, y le estarías dando barra libre.

## Vigilar otros servicios

La misma mecánica sirve para cualquier log. Fail2ban trae filtros de fábrica para nginx, Apache, Postfix y muchos más; se activan añadiendo su jail a `jail.local`:

```ini
[nginx-http-auth]
enabled  = true
port     = http,https
logpath  = /srv/docker/nginx-proxy/logs/error.log
maxretry = 5
```

Con contenedores hay un detalle que se pasa por alto: **Fail2ban corre en el host y solo puede leer ficheros del host**. Si nginx escribe sus logs dentro del contenedor, Fail2ban no los ve. Hace falta montar el directorio de logs en el host con un *bind mount* y apuntar `logpath` ahí:

```yaml
services:
  nginx-proxy:
    volumes:
      - /srv/logs/nginx:/var/log/nginx    # los logs salen al host
```

Los permisos de ese directorio montado tienen su propia trampa; está explicada en [Volúmenes y permisos](Volumenes-y-Permisos.md).

Hay un segundo detalle, más sutil: detrás de un reverse proxy, todas las peticiones llegan a la aplicación con la IP del proxy, no la del cliente. Si el log registra esa IP, Fail2ban acabará bloqueando al proxy y tirando el sitio entero. La aplicación tiene que registrar la cabecera `X-Forwarded-For` para que el bloqueo apunte a quien toca.

## Probar un filtro sin esperar a un ataque

`fail2ban-regex` aplica un filtro a un log y cuenta cuántas líneas reconoce. Es la herramienta para verificar una jail nueva sin tener que provocar fallos reales:

```bash
fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```

```
Results
=======

Failregex: 1847 total
|-  #) [# of hits] regular expression
|   1) [1731] ^Failed password for .* from <HOST>( port \d+)?( ssh\d*)?$
|   2) [116] ^Invalid user .* from <HOST>
`-

Lines: 21043 lines, 0 ignored, 1847 matched, 19196 missed
```

`matched` mayor que cero confirma que el filtro entiende el formato del log. Un `matched: 0` con un log lleno de fallos significa que el formato no coincide —típico cuando el servicio cambia de versión o escribe en formato JSON— y que la jail nunca bloqueará a nadie por muy activada que esté.

## Buenas prácticas avanzadas

- **Activa el bloqueo incremental.** Poner `bantime.increment = true` y `bantime.factor = 2` en `[DEFAULT]` hace que cada reincidencia duplique la duración del bloqueo: una hora, dos, cuatro, ocho. Los escáneres automáticos acaban con bloqueos de semanas mientras que un despiste tuyo se resuelve en una hora. Es mucho mejor equilibrio que un `bantime` fijo largo, y casi nadie lo configura.
- **Usa el backend `systemd` en distribuciones modernas.** `backend = systemd` en la jail lee directamente del *journal* en lugar de vigilar `/var/log/auth.log`. En sistemas donde ese fichero ya no existe —cada vez más habitual— es la diferencia entre una jail que funciona y una que nunca ve nada. El síntoma de no haberlo hecho es una jail activa con `Total failed: 0` en un servidor que evidentemente recibe intentos.
- **Vigila el tamaño de la tabla de bloqueos.** Con `bantime = -1` y varios meses de servicio, la cadena de `iptables` acumula decenas de miles de reglas y cada paquete entrante tiene que recorrerlas. `banaction = nftables-multiport` o el uso de *ipsets* mantiene el rendimiento constante independientemente del número de IPs bloqueadas. Es un problema que no se nota hasta que el servidor va inexplicablemente lento.
- **No dependas de Fail2ban donde puedas eliminar el ataque de raíz.** Es una capa reactiva: solo actúa después de varios intentos, y no puede hacer nada contra un ataque distribuido desde miles de IPs distintas, cada una con dos intentos. Desactivar las contraseñas en SSH o restringir el puerto a la [VPN](WireGuard.md) elimina la categoría entera de ataque en lugar de contenerla. Ordena las defensas así: primero quitar la puerta, después vigilarla.
- **Comprueba el filtro cada vez que actualices el servicio vigilado.** Un cambio de formato de log rompe el filtro en silencio: la jail sigue activa, `systemctl status` sigue en verde y no bloquea absolutamente nada. Ejecutar `fail2ban-regex` tras cada actualización mayor es un minuto que evita meses de falsa sensación de seguridad.

## Recursos didácticos

- [Regex101](https://regex101.com/) — imprescindible si escribes un filtro propio: pegas una línea real de tu log y la expresión regular, y ves en tiempo real qué captura cada grupo. Marca el modo de captura del `<HOST>` que Fail2ban espera.
- [AbuseIPDB](https://www.abuseipdb.com/) — pegas una de las IPs que ha bloqueado tu servidor y ves cuántos otros servidores la han denunciado y por qué. Es una forma muy gráfica de entender que el ruido de fondo de internet es real y constante.
- [Documentación oficial de Fail2ban](https://github.com/fail2ban/fail2ban/wiki) — la referencia de la sintaxis de jails, filtros y acciones.

---

*En resumen: Fail2ban lee los logs, cuenta los fallos y cierra la puerta a quien insiste — pero su mejor uso es limpiar el ruido de una puerta que ya no se puede forzar.*
