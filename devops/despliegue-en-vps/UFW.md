# UFW

## ¿Qué es?

UFW (*Uncomplicated Firewall*) es la capa amable sobre el firewall de Linux: en lugar de escribir reglas de `iptables` o `nftables`, dices `ufw allow 443/tcp` y él traduce.

## ¿Por qué existe?

Linux lleva décadas incorporando un firewall potentísimo en el propio núcleo, pero configurarlo directamente es incómodo hasta para quien lo hace a menudo: hay que razonar en términos de cadenas, tablas y orden de evaluación, y una regla mal puesta puede dejarte fuera del servidor sin previo aviso. Esta es una regla real de `iptables`:

```bash
iptables -A INPUT -p tcp -m state --state NEW --dport 443 -j ACCEPT
```

Y esta es la misma regla en UFW:

```bash
ufw allow 443/tcp
```

UFW no sustituye al firewall del núcleo: lo configura por ti con unos valores por defecto sensatos. La potencia sigue estando debajo, pero el 95 % de los casos de un servidor web se cubren con media docena de comandos legibles.

## ¿Cuándo y para qué se usa?

En todo servidor con IP pública. Su función es reducir la **superficie de ataque**: de los 65 535 puertos posibles, dejar accesibles solo los tres o cuatro que tu servicio necesita de verdad.

Esto importa más de lo que parece en un servidor con contenedores. Es muy fácil que un `docker compose` publique sin querer el puerto de una base de datos, o que un servicio de monitorización abra un panel de administración sin autenticación. Un firewall bien configurado convierte esos descuidos en algo inalcanzable desde internet en lugar de en una brecha.

---

## El concepto: política por defecto y excepciones

Un firewall se define en dos pasos: qué se hace **por defecto** con el tráfico, y qué **excepciones** hay.

La configuración correcta para un servidor es prohibir todo lo que entra y permitir todo lo que sale. Es lo contrario de "bloqueo lo que sé que es malo": aquí solo entra lo que has autorizado explícitamente, así que un servicio nuevo que se abra por descuido queda tapado de serie.

```bash
ufw default deny incoming
ufw default allow outgoing
```

- `deny incoming` — todo el tráfico entrante se descarta salvo excepción explícita.
- `allow outgoing` — el servidor puede iniciar conexiones hacia fuera (descargar paquetes, resolver DNS, llamar a APIs). Restringir la salida también es posible, pero rompe cosas constantemente y rara vez compensa en un servidor de aplicaciones.

Estos dos comandos **no activan nada todavía**: solo fijan la política que se aplicará cuando el firewall arranque.

## El orden importa: cómo no dejarte fuera

Aquí está el error clásico, y es lo bastante frecuente como para merecer su propio aviso:

```bash
# ❌ MAL — te desconecta al instante
ufw default deny incoming
ufw enable
```

En cuanto se ejecuta `ufw enable`, la política de "denegar todo lo entrante" pasa a aplicarse, y eso incluye **tu propia sesión SSH**. La conexión se corta, y sin SSH no hay forma de añadir la regla que falta. Con suerte tienes consola web en el panel del proveedor; si no, toca reinstalar.

La forma correcta es definir todas las reglas **antes** de activar:

```bash
# ✅ BIEN — primero las excepciones, después el interruptor
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp          # SSH
ufw allow 80/tcp          # HTTP
ufw allow 443/tcp         # HTTPS
ufw enable
```

Al ejecutar `ufw enable`, UFW avisa de este riesgo exacto y pide confirmación:

```
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

Un detalle sobre `ufw allow 22/tcp`: UFW también acepta nombres de servicio, y `ufw allow OpenSSH` es equivalente porque lee `/etc/services` y los perfiles de aplicación. Usar el número es más explícito y no depende de que el perfil exista, sobre todo si has cambiado el puerto de SSH.

## Los puertos que hace falta abrir (y los que no)

Para un servidor que sirve una aplicación web dockerizada tras un [reverse proxy](Reverse-Proxy-con-nginx-proxy.md), la lista es corta:

| Puerto | Para qué | ¿Imprescindible? |
|---|---|---|
| 22/tcp | SSH | Sí, salvo que uses solo consola del proveedor |
| 80/tcp | HTTP | Sí — [Let's Encrypt](HTTPS-con-Lets-Encrypt.md) valida por aquí |
| 443/tcp | HTTPS | Sí, es por donde entra el tráfico real |
| 51820/udp | [WireGuard](WireGuard.md) | Solo si montas VPN |

El puerto 80 sorprende a quien piensa "si todo va por HTTPS, cierro el 80". No se puede: la validación automática de certificados (el *challenge* HTTP-01 de ACME) consiste en que Let's Encrypt pide un fichero concreto por HTTP en el puerto 80. Si está cerrado, la emisión y la renovación del certificado fallan, y el sitio se queda sin HTTPS a los 90 días.

Y estos **no** se abren nunca desde internet:

- `5432` (PostgreSQL), `3306` (MySQL), `27017` (MongoDB) — la base de datos habla con la API por la red interna de Docker, no por la red pública.
- `6379` (Redis) — igual, y además Redis sin contraseña es un regalo.
- Puertos de paneles de administración, métricas o depuración.

Si necesitas conectarte a la base de datos desde tu portátil, la respuesta no es abrir el puerto: es un túnel SSH (`ssh -L 5432:localhost:5432 tienda`), que reutiliza la conexión ya autenticada y no expone nada.

## Comprobar el estado

```bash
ufw status verbose
```

La salida dice la política por defecto y todas las reglas activas:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
443/tcp (v6)               ALLOW IN    Anywhere (v6)
```

Las líneas `(v6)` son las mismas reglas aplicadas a IPv6, que UFW añade automáticamente. Conviene mirarlas: si tu servidor tiene IPv6 y las reglas solo existieran para IPv4, el firewall estaría abierto de par en par por la otra vía.

Para borrar una regla, la forma cómoda es listarlas numeradas y eliminar por número:

```bash
ufw status numbered
ufw delete 3
```

También funciona repetir la regla con `delete` delante, que es lo práctico cuando escribes un script:

```bash
ufw delete allow 80/tcp
```

## Restringir por origen, no solo por puerto

Abrir un puerto "a todo el mundo" es lo habitual para el 80 y el 443, pero no tiene por qué serlo para el resto. UFW permite limitar el origen:

```bash
ufw allow from 198.51.100.25 to any port 22 proto tcp
```

Esto deja SSH accesible **solo** desde esa IP. Es excelente si tu oficina tiene IP fija; es una forma de dejarte fuera si trabajas desde casa con IP dinámica, así que úsalo con cuidado y con una segunda vía de acceso disponible.

La variante más útil en un servidor con VPN es restringir por **interfaz de red** en lugar de por IP. Una vez [WireGuard](WireGuard.md) está en marcha y crea la interfaz `wg0`, se puede exigir que la administración solo entre por el túnel:

```bash
ufw delete allow 22/tcp                 # quitar el acceso público a SSH
ufw allow in on wg0 to any port 22      # permitirlo solo desde la VPN
ufw reload
```

Tras esto, un escaneo desde internet no ve el puerto 22 en absoluto: no es que rechace la conexión, es que no responde. Haz el cambio **con la VPN ya conectada y funcionando**, y comprueba que puedes entrar por ella antes de borrar la regla pública.

> El orden de estos dos comandos es deliberado en la documentación, pero en la práctica conviene invertirlo: primero añade la regla de `wg0`, verifica que entras por la VPN, y solo entonces borra la regla pública.

## UFW y Docker: la trampa que casi todo el mundo pisa

Este es el comportamiento menos intuitivo de todo el documento, y explica por qué mucha gente cree tener un firewall que en realidad no le protege.

Cuando publicas un puerto en un contenedor, Docker escribe sus propias reglas directamente en `iptables`, en una cadena que se evalúa **antes** que las de UFW. Resultado: el puerto queda accesible desde internet aunque UFW diga que está bloqueado.

Se comprueba fácil. Con este `docker-compose.yml`:

```yaml
services:
  tienda-db:
    image: postgres:16
    ports:
      - "5432:5432"     # ⚠️ esto expone el puerto al mundo
```

`ufw status` seguirá sin mostrar el 5432 y, sin embargo, un escaneo desde fuera lo encuentra abierto. UFW no miente: es que el tráfico ni siquiera llega a sus reglas.

Hay dos formas de evitarlo, y la primera es la buena:

**1. No publicar el puerto.** Si la base de datos solo la usa la API, no necesita `ports:` en absoluto. Los contenedores de una misma red de Docker se ven entre ellos por su nombre sin publicar nada:

```yaml
services:
  tienda-db:
    image: postgres:16
    networks:
      - internal-net      # ✅ alcanzable solo desde otros contenedores
```

La API se conecta a `tienda-db:5432` y el puerto no existe para el resto del mundo. Esto se desarrolla en [Redes Docker y exposición de servicios](Redes-Docker-y-Exposicion-de-Servicios.md).

**2. Si tienes que publicarlo, átalo a localhost.** Añadir la IP delante limita la escucha a la propia máquina:

```yaml
ports:
  - "127.0.0.1:5432:5432"   # accesible solo desde el servidor, no desde fuera
```

Desde tu portátil llegarías con un túnel SSH, no directamente. Esta forma sí respeta lo que esperarías del firewall, porque el puerto no se abre en la interfaz pública.

## Buenas prácticas avanzadas

- **Ten siempre una segunda vía antes de tocar reglas.** La consola web del proveedor (VNC, *rescue mode*) es la única que no depende de la red. Localízala y pruébala **antes** de necesitarla, no cuando ya te has bloqueado. Quien administra servidores a diario tiene ese enlace guardado; quien no, aprende por las malas.
- **Usa `ufw limit` en SSH si lo dejas abierto al mundo.** `ufw limit 22/tcp` bloquea una IP que abra más de seis conexiones en 30 segundos. No sustituye a [Fail2ban](Fail2ban.md) —solo mira la frecuencia de conexión, no si el intento falló— pero es una línea que frena el escaneo masivo, y se combina bien con él.
- **Revisa qué está escuchando de verdad, no solo qué permite el firewall.** `ss -tulpn` lista los puertos abiertos con el proceso que los ocupa. Es la comprobación que destapa la trampa de Docker y los servicios que alguien levantó "un momento para probar" hace tres meses. Contrasta esa lista con `ufw status` cada vez que despliegues algo nuevo.
- **Activa el registro y luego bájalo.** `ufw logging medium` escribe en `/var/log/ufw.log` cada paquete bloqueado, lo que es utilísimo mientras depuras por qué un servicio no conecta. Déjalo así unos días y vuelve a `low`: en un servidor público, el nivel alto llena el disco a una velocidad que sorprende.
- **Documenta cada regla con un comentario.** `ufw allow 9100/tcp comment "métricas node_exporter"` guarda el motivo junto a la regla. Dentro de seis meses, un puerto abierto sin explicación no se cierra —por miedo a romper algo— y se queda ahí para siempre. El comentario es lo que permite limpiar.

## Recursos didácticos

- [Shields Up! de GRC](https://www.grc.com/shieldsup) — escanea desde fuera los puertos de la IP desde la que navegas y te dice cuáles responden. Es la forma más rápida de comprobar si el firewall hace lo que crees, especialmente para destapar la trampa de Docker.
- [Explainshell](https://explainshell.com/) — desmenuza los comandos de `ufw` y `ss` opción por opción.
- [Guía de UFW de Ubuntu](https://help.ubuntu.com/community/UFW) — la referencia oficial, con la sintaxis completa de reglas por origen, interfaz y rango de puertos.

---

*En resumen: UFW cierra todo lo que no has abierto a propósito — pero comprueba siempre qué escucha de verdad, porque Docker abre puertos por su cuenta y por debajo.*
