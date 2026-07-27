# WireGuard

## ¿Qué es?

WireGuard es una VPN moderna integrada en el núcleo de Linux. Crea un túnel cifrado entre tu ordenador y el servidor, de forma que ambos se ven como si estuvieran en la misma red local privada.

## ¿Por qué existe?

Las VPN tradicionales (OpenVPN, IPsec) funcionan, pero arrastran décadas de opciones acumuladas: decenas de miles de líneas de código, una configuración con docenas de parámetros y una superficie de ataque grande precisamente en la pieza que debería ser la más segura del sistema.

WireGuard se escribió desde cero con la premisa contraria: unas 4 000 líneas de código, un solo conjunto de algoritmos criptográficos modernos (sin negociación ni opciones débiles que desactivar) y una configuración que cabe en diez líneas. Además vive dentro del núcleo de Linux, lo que lo hace notablemente más rápido.

Su modelo mental también es distinto. No hay "cliente" y "servidor": hay **pares** (*peers*), cada uno con su par de claves pública y privada, que se autorizan mutuamente. Quien tiene IP pública fija hace de punto de encuentro, pero conceptualmente son iguales.

> Si has configurado claves SSH, el esquema es idéntico: cada extremo genera un par de claves, se intercambian las públicas y a partir de ahí se reconocen. La diferencia es que aquí lo que se establece no es una sesión de terminal, sino una interfaz de red completa.

## ¿Cuándo y para qué se usa?

En el contexto de un VPS, el caso más valioso es **sacar la administración de internet**. Con una VPN funcionando, el firewall puede dejar de aceptar SSH desde el mundo y aceptarlo solo desde el túnel: el puerto 22 desaparece de los escaneos porque, sencillamente, ya no responde a nadie de fuera.

El segundo caso es dar acceso a servicios internos que no deben ser públicos: un panel de administración, un gestor de base de datos, un entorno de preproducción. En lugar de protegerlos con una contraseña y esperar que aguante, no se publican en absoluto y solo son alcanzables desde la VPN.

Si el concepto de VPN te resulta nuevo en general, la colección de redes lo introduce en [VPN](../../redes/redes-y-acceso-remoto/VPN.md). Aquí vamos directos al montaje sobre un VPS.

---

## Instalar y generar las claves del servidor

```bash
apt install wireguard -y
```

WireGuard usa criptografía de clave pública, así que lo primero es generar el par del servidor:

```bash
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
chmod 600 /etc/wireguard/server_private.key
```

La línea encadena tres cosas: `wg genkey` genera la clave privada, `tee` la escribe en un fichero **y** la pasa por la tubería, y `wg pubkey` deriva la pública a partir de ella. El `chmod 600` deja la privada legible solo por su propietario; sin él, cualquier usuario del servidor podría suplantar al servidor entero.

Las claves son cadenas cortas en base64:

```bash
cat /etc/wireguard/server_public.key
```

```
xTIBA7yPGkFR4YQ8mZC5Kd3vNqL2hJs9WbEoU1nRtCg=
```

## Configurar la interfaz del servidor

Toda la configuración vive en un fichero por interfaz. El nombre del fichero **es** el nombre de la interfaz de red que se creará:

```bash
nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address    = 10.10.0.1/24
ListenPort = 51820
PrivateKey = <contenido de server_private.key>
SaveConfig = false
```

Qué significa cada línea:

- **`Address = 10.10.0.1/24`** — la IP del servidor **dentro** de la VPN. Es una red privada nueva que te inventas y que no debe solaparse con ninguna red real que uses: ni la de tu oficina, ni la de tu casa, ni las que Docker asigna a sus contenedores (`172.17.0.0/16` en adelante). El `/24` define el rango disponible para los clientes: de `10.10.0.1` a `10.10.0.254`.
- **`ListenPort = 51820`** — el puerto UDP donde escucha. Es el estándar de facto y hay que abrirlo en el firewall.
- **`PrivateKey`** — se pega el contenido del fichero, no la ruta.
- **`SaveConfig = false`** — importante. Con `true`, WireGuard **reescribe este fichero** al parar el servicio, y se lleva por delante los comentarios y el formato que hayas puesto. Con `false`, el fichero es la fuente de verdad y los cambios en caliente se pierden al reiniciar, que es el comportamiento predecible.

Abre el puerto en el [firewall](UFW.md):

```bash
ufw allow 51820/udp
```

Y arranca la interfaz:

```bash
systemctl enable wg-quick@wg0     # que arranque al iniciar el servidor
systemctl start wg-quick@wg0
```

La sintaxis `wg-quick@wg0` es una unidad de systemd parametrizada: el `wg0` después de la arroba le dice qué fichero de configuración usar. Si tuvieras otra interfaz `wg1.conf`, sería `wg-quick@wg1`.

Comprueba que está en marcha:

```bash
wg show
```

```
interface: wg0
  public key: xTIBA7yPGkFR4YQ8mZC5Kd3vNqL2hJs9WbEoU1nRtCg=
  private key: (hidden)
  listening port: 51820
```

Todavía no hay ningún par autorizado, así que no aparece nada más.

## Añadir un cliente

Cada dispositivo que se conecte necesita su propio par de claves. Se pueden generar en el servidor (cómodo) o en el propio dispositivo (más correcto, porque la clave privada nunca viaja). Para el primer cliente, generándolas en el servidor:

```bash
wg genkey | tee /etc/wireguard/portatil_private.key | wg pubkey > /etc/wireguard/portatil_public.key
chmod 600 /etc/wireguard/portatil_private.key
```

Ahora se autoriza en el servidor añadiendo un bloque `[Peer]` a `wg0.conf`:

```ini
[Peer]
# portátil de trabajo
PublicKey  = <contenido de portatil_public.key>
AllowedIPs = 10.10.0.2/32
```

- **`PublicKey`** — la **pública** del cliente. Es lo único que el servidor necesita para reconocerlo.
- **`AllowedIPs = 10.10.0.2/32`** — aquí está el concepto que más confunde de WireGuard, porque **significa dos cosas a la vez** según el lado:
  - En el servidor, es la lista de IPs de origen que acepta de ese par (una lista de control de acceso).
  - En el cliente, es la lista de destinos que se enrutan por el túnel.

  El `/32` significa "exactamente esta IP y ninguna más". Cada cliente lleva la suya, incrementando el último número: `10.10.0.3/32`, `10.10.0.4/32`...

El comentario con el nombre del dispositivo no es decorativo: dentro de seis meses y con cinco pares, es lo único que te dirá cuál revocar cuando alguien pierda el portátil.

Aplica el cambio:

```bash
systemctl restart wg-quick@wg0
```

## Configurar el cliente

En el dispositivo se instala el cliente oficial ([wireguard.com/install](https://www.wireguard.com/install/), disponible para Windows, macOS, Linux, iOS y Android) y se importa esta configuración:

```ini
[Interface]
Address    = 10.10.0.2/32
PrivateKey = <contenido de portatil_private.key>
DNS        = 10.10.0.1

[Peer]
PublicKey           = <contenido de server_public.key>
Endpoint            = 203.0.113.10:51820
AllowedIPs          = 10.10.0.0/24
PersistentKeepalive = 25
```

Las líneas que necesitan explicación:

- **`Endpoint`** — la IP pública y el puerto del servidor. Es la única dirección real de todo el fichero; el resto son IPs del túnel.
- **`AllowedIPs = 10.10.0.0/24`** — como decíamos, aquí significa "qué tráfico va por el túnel". Con este valor, **solo** el tráfico hacia la red de la VPN pasa por el servidor; navegar por internet sigue saliendo por tu conexión normal. Es lo que quieres para administrar un servidor. Si pusieras `0.0.0.0/0`, **todo** tu tráfico saldría por el VPS: útil para navegar desde una wifi pública, innecesario y más lento para el uso habitual.
- **`DNS = 10.10.0.1`** — usa el servidor como resolutor de nombres. Solo tiene sentido si has montado [dnsmasq](DNS-Interno-con-dnsmasq.md); si no, quita la línea o el cliente perderá la resolución de nombres al conectar.
- **`PersistentKeepalive = 25`** — envía un paquete vacío cada 25 segundos. Hace falta cuando el cliente está detrás de un router con NAT: sin tráfico, el router cierra la asociación al cabo de un minuto y el servidor deja de poder iniciar conexiones hacia el cliente. Con esta línea, el túnel se mantiene abierto en ambos sentidos.

Al conectar, verifica desde el cliente:

```bash
ping 10.10.0.1
ssh deployer@10.10.0.1
```

Y en el servidor, `wg show` ahora muestra la conexión:

```
interface: wg0
  public key: xTIBA7yPGkFR4YQ8mZC5Kd3vNqL2hJs9WbEoU1nRtCg=
  listening port: 51820

peer: 9pQm2LwXvR8sTzN4kHc7YbJd1FgA5eUo3ViK0nMxSrE=
  endpoint: 198.51.100.25:54219
  allowed ips: 10.10.0.2/32
  latest handshake: 12 seconds ago
  transfer: 1.24 MiB received, 8.91 MiB sent
```

**`latest handshake`** es el indicador que importa. Si aparece con un valor reciente, el túnel funciona. Si en su lugar no hay línea de handshake, la conexión no se ha establecido nunca: revisa el puerto UDP en el firewall y que las claves públicas no estén cruzadas.

## Restringir la administración a la VPN

Con el túnel funcionando, llega el paso que justificaba todo esto. En el [firewall](UFW.md), primero se añade la regla que permite SSH por la interfaz del túnel:

```bash
ufw allow in on wg0 to any port 22
ufw reload
```

**Comprueba ahora que puedes entrar por la VPN** (`ssh deployer@10.10.0.1`) antes de tocar nada más. Solo entonces se retira el acceso público:

```bash
ufw delete allow 22/tcp
ufw reload
```

Tras esto, un escaneo desde internet no encuentra el puerto 22. No es que rechace la conexión: no responde. El servicio sigue ahí, alcanzable solo por quien tenga una clave de WireGuard autorizada.

El mismo patrón sirve para exponer un servicio interno solo por VPN: en lugar de conectarlo a `proxy-net` con un `VIRTUAL_HOST` público, se publica en la IP del túnel:

```yaml
services:
  panel-admin:
    image: registro.ejemplo.com/panel-admin:1.4.0
    ports:
      - "10.10.0.1:8080:8080"    # solo alcanzable desde la VPN
```

Atar la publicación a `10.10.0.1` en lugar de a todas las interfaces es lo que hace que el puerto no exista desde fuera. Es la misma técnica que el `127.0.0.1:` de [Redes Docker y exposición de servicios](Redes-Docker-y-Exposicion-de-Servicios.md), con la IP del túnel en lugar de localhost.

## Si necesitas enrutar tráfico a través del servidor

El montaje anterior conecta cliente y servidor, pero no permite alcanzar **otras** redes a través de él. Para eso hay que habilitar el reenvío de paquetes, que Linux trae desactivado por defecto:

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

```
net.ipv4.ip_forward = 1
```

`sysctl -p` aplica el cambio inmediatamente; la línea en `/etc/sysctl.conf` hace que sobreviva a los reinicios. Si solo ejecutas el `sysctl -w`, funciona hasta el siguiente arranque y luego deja de hacerlo sin motivo aparente.

Con UFW hay un ajuste adicional, porque bloquea el tráfico reenviado por defecto. En `/etc/default/ufw`:

```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Y en `/etc/ufw/sysctl.conf`, descomentar:

```
net/ipv4/ip_forward=1
```

Ese fichero usa barras en lugar de puntos —es la sintaxis propia de UFW— y sus valores **sobrescriben** los de `/etc/sysctl.conf` cuando UFW está activo. Es el motivo por el que el reenvío "no funciona" pese a haberlo habilitado correctamente en el sitio estándar.

## Buenas prácticas avanzadas

- **Genera las claves en el dispositivo, no en el servidor.** Es más incómodo (hay que pedirle a cada persona su clave pública) y es la única forma de que la clave privada no viaje nunca por correo, chat o SCP. Si la generas tú en el servidor, existe al menos una copia fuera del dispositivo y no puedes garantizar que se haya borrado.
- **Un par por dispositivo, no por persona.** Compartir una configuración entre el portátil y el móvil parece práctico, pero WireGuard asocia una IP del túnel a cada clave y dos dispositivos con la misma clave se pisan mutuamente: la conexión salta de uno a otro de forma errática. Además, revocar el acceso de un dispositivo perdido obliga a reconfigurar todos.
- **Comprueba que la subred de la VPN no colisiona.** Elegir `10.0.0.0/24` o `192.168.1.0/24` es pedir un conflicto: son los rangos que usan la mitad de los routers domésticos, y un cliente en esa red no podrá enrutar hacia la VPN. Lo mismo con `172.17.0.0/16` y siguientes, que es lo que Docker reparte. Un rango poco común dentro de `10.x` evita el problema para siempre.
- **Vigila el `latest handshake`, no el estado del servicio.** `systemctl status wg-quick@wg0` dice que la interfaz existe, no que nadie esté conectado ni que el tráfico fluya. WireGuard no mantiene "sesiones": si el handshake tiene más de tres minutos, ese par está desconectado aunque todo parezca correcto. Es la única métrica fiable.
- **Ten siempre una vía alternativa antes de cerrar SSH público.** El día que WireGuard no arranque tras una actualización del núcleo —pasa—, la VPN es tu única puerta y estará cerrada. Localiza la consola web del proveedor y **pruébala** antes de ejecutar `ufw delete allow 22/tcp`, no después.

## Recursos didácticos

- [WireGuard Config Generator](https://www.wireguardconfig.com/) — genera las claves y los ficheros de configuración de servidor y clientes en el navegador, mostrando cómo encaja cada pieza. Muy útil para entender la simetría entre ambos lados; para producción, genera tus propias claves localmente.
- [Subnet Calculator](https://www.subnet-calculator.com/cidr.php) — para elegir el rango de la VPN comprobando que no solapa con las redes que ya usas, y para entender qué significa exactamente cada `/24` o `/32` de `AllowedIPs`.
- [wireguard.com/quickstart](https://www.wireguard.com/quickstart/) — la guía oficial: corta, precisa y con el modelo conceptual de pares bien explicado.

---

*En resumen: WireGuard son diez líneas de configuración y un par de claves por dispositivo — y su mejor uso en un VPS es hacer que el puerto de administración deje de existir para internet.*
