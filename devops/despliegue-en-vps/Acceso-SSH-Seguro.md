# Acceso SSH seguro

## ¿Qué es?

Endurecer el acceso SSH consiste en cambiar la forma en que un servidor acepta conexiones remotas: pasar de "entra quien sepa la contraseña de `root`" a "entra quien tenga una clave criptográfica concreta, y nunca como `root`".

## ¿Por qué existe?

Una contraseña se puede adivinar. Los servidores con el puerto 22 abierto reciben miles de intentos automatizados al día probando combinaciones de usuario y contraseña comunes, y `root` es el usuario que prueban siempre porque existe en todas las máquinas Linux. Basta con que una contraseña sea mediocre para que el servidor cambie de dueño.

Una clave SSH resuelve el problema por otra vía: en lugar de un secreto que se teclea, hay un par de claves. La **pública** se copia al servidor y no es secreta; la **privada** se queda en tu ordenador y nunca viaja. Al conectar, el servidor lanza un reto matemático que solo se puede resolver con la privada. No hay nada que adivinar: una clave Ed25519 no se rompe por fuerza bruta.

> Si te suena la firma de tokens JWT, es exactamente la misma idea de criptografía asimétrica: quien tiene la clave privada demuestra su identidad, y con la pública cualquiera puede verificarlo sin poder suplantarlo.

## ¿Cuándo y para qué se usa?

En cualquier servidor accesible desde internet, sin excepciones. Es el primer cambio que se hace tras crear un VPS y antes de desplegar nada. También es lo que permite que un pipeline de CI/CD despliegue solo: un proceso automático no puede teclear una contraseña, pero sí puede usar una clave.

Este documento cubre el **endurecimiento** del acceso. Si SSH como protocolo te resulta nuevo (qué es, cómo se conecta, qué es `known_hosts`), empieza por la ficha de [SSH](../../redes/redes-y-acceso-remoto/SSH.md) en la colección de redes.

---

## Generar el par de claves

Las claves se generan **en tu máquina**, no en el servidor. La privada no debería viajar nunca por la red, ni siquiera para copiarla a un sitio "de confianza".

```bash
ssh-keygen -t ed25519 -C "portatil-de-trabajo" -f ~/.ssh/tienda_vps
```

Qué hace cada opción:

- `-t ed25519` elige el algoritmo. Ed25519 es el recomendado hoy: claves cortas, rápidas y muy seguras. La alternativa clásica es RSA, y si la necesitas por compatibilidad con algún sistema antiguo, usa al menos `-t rsa -b 4096`.
- `-C "portatil-de-trabajo"` es un comentario libre que queda escrito dentro de la clave pública. No afecta a la seguridad, pero es lo que te permitirá saber, dentro de un año y con cinco claves en el servidor, cuál corresponde a qué máquina. Ponlo siempre.
- `-f ~/.ssh/tienda_vps` fija el nombre. Sin esto sobrescribe `~/.ssh/id_ed25519`, que puede ser la clave que ya usas para otras cosas.

El comando pregunta por una *passphrase*. Es una contraseña que cifra el fichero de la clave privada en disco: si alguien te roba el portátil, la clave sigue siendo inservible sin ella. Ponla.

El resultado son dos ficheros:

```
~/.ssh/tienda_vps       ← privada. Nunca sale de aquí.
~/.ssh/tienda_vps.pub   ← pública. Esta es la que se copia al servidor.
```

La pública es una sola línea de texto, y se puede compartir sin riesgo:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIL8Xk2mQ... portatil-de-trabajo
```

## Instalar la clave pública en el servidor

El servidor guarda las claves autorizadas de cada usuario en `~/.ssh/authorized_keys` de ese usuario: una clave por línea. Cualquier clave que aparezca ahí puede entrar como ese usuario.

La forma cómoda, si todavía tienes acceso por contraseña, es dejar que `ssh-copy-id` lo haga:

```bash
ssh-copy-id -i ~/.ssh/tienda_vps.pub deployer@203.0.113.10
```

Copia la clave, crea `~/.ssh` si no existe y ajusta los permisos correctamente. Si el comando no está disponible (por ejemplo en Windows sin Git Bash), el equivalente manual es:

```bash
cat ~/.ssh/tienda_vps.pub | ssh deployer@203.0.113.10 \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Fíjate en el `>>` (añadir) en lugar de `>` (sobrescribir): con `>` borrarías las claves que ya hubiera y podrías dejar fuera a otras personas del equipo.

Los `chmod` no son opcionales. **SSH ignora silenciosamente `authorized_keys` si los permisos son demasiado abiertos**, por diseño: si otro usuario del sistema pudiera escribir en ese fichero, podría concederse acceso a sí mismo. Es la causa número uno de "he copiado la clave y me sigue pidiendo la contraseña".

Ahora comprueba que funciona, **sin cerrar la sesión actual**:

```bash
ssh -i ~/.ssh/tienda_vps deployer@203.0.113.10
```

Si entra sin pedir la contraseña del servidor (la *passphrase* de la clave sí la puede pedir), la clave está bien instalada.

## Desactivar contraseñas y el acceso de root

Mientras el servidor siga aceptando contraseñas, la clave no ha servido de nada: el atacante seguirá probando por la puerta vieja. La configuración del servidor SSH está en `/etc/ssh/sshd_config`.

```bash
sudo nano /etc/ssh/sshd_config
```

Las líneas que importan:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
UsePAM yes
```

Qué hace cada una:

- `PermitRootLogin no` — `root` deja de poder conectarse directamente. Se entra como `deployer` y se escala con `sudo`, lo que además deja rastro en los logs de quién hizo qué.
- `PasswordAuthentication no` — el servidor deja de aceptar contraseñas. Este es el cambio que hace desaparecer los ataques de fuerza bruta de golpe.
- `PubkeyAuthentication yes` — habilita explícitamente las claves. Suele venir activo por defecto, pero conviene dejarlo escrito.
- `KbdInteractiveAuthentication no` — cierra una vía alternativa de autenticación interactiva que, en algunas configuraciones, acaba pidiendo una contraseña igualmente. (En versiones antiguas de OpenSSH esta directiva se llamaba `ChallengeResponseAuthentication`.)
- `UsePAM yes` — deja que el sistema siga aplicando sus políticas de sesión (límites, registro de accesos). No debilita nada.

Ojo con un detalle que despista: muchas distribuciones incluyen al final del fichero una línea `Include /etc/ssh/sshd_config.d/*.conf`, y los proveedores de VPS suelen dejar ahí un fichero que **vuelve a activar** `PasswordAuthentication yes`. En SSH gana la primera directiva que aparece, y los `Include` suelen estar al principio. Si tras reiniciar el servicio las contraseñas siguen funcionando, mira ahí:

```bash
sudo grep -r "PasswordAuthentication" /etc/ssh/
```

```
/etc/ssh/sshd_config:PasswordAuthentication no
/etc/ssh/sshd_config.d/50-cloud-init.conf:PasswordAuthentication yes   ← el culpable
```

Antes de reiniciar nada, valida la sintaxis. Un error tipográfico impide que el servicio arranque, y sin SSH no hay forma de entrar a arreglarlo:

```bash
sudo sshd -t          # sin salida = configuración correcta
sudo systemctl restart ssh
```

**Y ahora la regla de oro: no cierres la sesión actual.** Abre una segunda terminal y comprueba que puedes entrar con la clave. Si algo salió mal, la sesión que sigue abierta es tu única vía para deshacerlo. Si además tu proveedor ofrece una consola web (VNC o similar), tenla localizada antes de tocar SSH: es la red de seguridad definitiva.

## Simplificar la conexión con `~/.ssh/config`

Teclear `ssh -i ~/.ssh/tienda_vps deployer@203.0.113.10` cada vez es incómodo y propenso a errores. El fichero `~/.ssh/config` **de tu máquina** permite darle un nombre a esa combinación:

```
Host tienda
    HostName 203.0.113.10
    User deployer
    IdentityFile ~/.ssh/tienda_vps
    IdentitiesOnly yes
```

Con esto, `ssh tienda` equivale al comando largo. `IdentitiesOnly yes` merece una mención: sin él, SSH ofrece al servidor **todas** las claves que tenga cargadas antes de llegar a la correcta, y muchos servidores cortan la conexión tras cinco intentos fallidos con un `Too many authentication failures`. Con esa opción, solo se ofrece la clave indicada.

El alias funciona también para el resto de herramientas que hablan SSH:

```bash
scp copia.sql tienda:/srv/backups/     # copiar ficheros
rsync -av /srv/docker/ tienda:/srv/docker/
```

En Windows el fichero está en `C:\Users\TuUsuario\.ssh\config` y la ruta de la clave se escribe con el formato de Windows (`C:\Users\TuUsuario\.ssh\tienda_vps`).

## Cuando el que se conecta es un pipeline

Un despliegue automático necesita entrar sin que nadie teclee nada, así que su clave **no puede tener passphrase**. Eso la hace más frágil, y por eso se compensa limitando lo que puede hacer.

Genera una clave dedicada al pipeline (nunca reutilices la tuya) y, al añadirla al `authorized_keys` del servidor, restríngela con opciones al principio de la línea:

```
restrict,command="/srv/scripts/deploy.sh" ssh-ed25519 AAAAC3Nza... ci-deploy
```

- `restrict` desactiva de golpe el reenvío de puertos, el reenvío de agente, X11 y la asignación de terminal. Es el punto de partida más seguro.
- `command="..."` fuerza que esa clave ejecute **siempre** ese script, ignorando lo que pida el cliente. Aunque alguien robe la clave del CI, lo único que consigue es lanzar el despliegue, no una shell.

La clave privada se guarda como secreto del sistema de CI, nunca en el repositorio. Sobre por qué eso importa y qué hacer si se filtra, la colección de seguridad lo trata a fondo en [Por qué los secretos no van a Git](../../seguridad/gestion-de-secretos-en-desarrollo/Por-Que-Los-Secretos-No-Van-A-Git.md).

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| `Permission denied (publickey)` | La pública no está en `authorized_keys`, o está en el usuario equivocado |
| Sigue pidiendo contraseña tras copiar la clave | Permisos de `~/.ssh` (700) o `authorized_keys` (600) demasiado abiertos |
| `Too many authentication failures` | El agente ofrece muchas claves; falta `IdentitiesOnly yes` |
| `PasswordAuthentication no` no surte efecto | Un fichero en `sshd_config.d/` lo sobrescribe |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | El servidor se reinstaló (huella nueva) — o alguien se interpone |

Para diagnosticar cualquiera de ellos, el modo verboso del cliente dice exactamente qué clave se ofreció y por qué se rechazó:

```bash
ssh -vvv -i ~/.ssh/tienda_vps deployer@203.0.113.10
```

Entre el ruido, las líneas útiles son las del tipo `Offering public key:` y `Authentications that can continue:`. Y si sospechas del lado del servidor, sus logs son igual de explícitos:

```bash
sudo journalctl -u ssh -n 50 --no-pager
```

```
sshd[1421]: Authentication refused: bad ownership or modes for file /home/deployer/.ssh/authorized_keys
```

Ese mensaje, por ejemplo, es literalmente el problema de permisos: `chmod 600` y resuelto.

## Buenas prácticas avanzadas

- **Una clave por persona y por máquina, nunca una compartida.** Es tentador tener "la clave del servidor" que usa todo el equipo, pero entonces revocar el acceso a alguien que se va obliga a rotar la clave de todos. Con una línea por persona en `authorized_keys`, dar de baja a alguien es borrar su línea. Lo mismo aplica a tus propios dispositivos: si pierdes el portátil, quieres revocar solo esa clave.
- **Cambiar el puerto de SSH reduce el ruido, no el riesgo.** Mover SSH al 2222 elimina la práctica totalidad de los escaneos automáticos y adelgaza mucho los logs, lo que ayuda a que un intento real destaque. Pero no protege frente a nadie que escanee tus puertos a propósito: es higiene, no seguridad. Lo que de verdad cierra la puerta es `PasswordAuthentication no`.
- **Usa el agente SSH y desconfía del reenvío de agente.** `ssh-add` te evita teclear la passphrase en cada conexión. El reenvío (`ssh -A`) permite saltar del servidor a otro usando tu clave local, y suena muy práctico — pero mientras la sesión esté abierta, quien tenga `root` en ese servidor puede usar tu clave para conectarse a cualquier sitio donde tengas acceso. Si necesitas encadenar saltos, usa `ProxyJump` en su lugar: cifra de extremo a extremo y no expone la clave al servidor intermedio.
- **Fija los algoritmos si el servidor es sensible.** Añadir `PubkeyAcceptedAlgorithms ssh-ed25519,rsa-sha2-512` y `KexAlgorithms curve25519-sha256` a `sshd_config` descarta criptografía antigua que sigue habilitada por compatibilidad. Es el tipo de ajuste que una auditoría pide y que casi nadie hace, porque exige comprobar que ningún cliente antiguo se queda fuera.
- **Restringe también desde dónde se acepta la conexión.** Aunque tengas claves, el puerto sigue abierto al mundo. Combinar SSH con un firewall que solo lo permita desde la red interna de una [VPN](WireGuard.md) hace que el servicio deje de ser alcanzable desde internet. Es la capa que convierte "difícil de romper" en "invisible".

## Recursos didácticos

- [SSH Audit](https://www.sshaudit.com/) — pegas la IP o el dominio de tu servidor y te audita la configuración SSH desde fuera: algoritmos débiles, versiones vulnerables y qué línea concreta añadir para corregir cada aviso. Es la forma más directa de comprobar que el endurecimiento ha surtido efecto.
- [Explainshell](https://explainshell.com/) — útil para desmenuzar los `ssh-keygen` y `chmod` de esta guía opción por opción.
- [`man sshd_config`](https://man.openbsd.org/sshd_config) — la referencia oficial de todas las directivas, escrita sorprendentemente bien. Merece la pena leer las entradas de las cinco directivas de esta ficha.

---

*En resumen: una clave SSH no se adivina, así que el endurecimiento consiste en instalar la tuya y luego cerrar todas las demás puertas — sin cerrar la sesión que tienes abierta.*
