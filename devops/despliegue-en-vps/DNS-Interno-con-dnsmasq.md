# DNS interno con dnsmasq

## ¿Qué es?

dnsmasq es un servidor DNS ligero que resuelve los nombres que tú le indiques y reenvía el resto a un DNS público. En un VPS con VPN sirve para que un mismo dominio apunte a la IP privada del túnel cuando lo consultas desde dentro, y a la pública cuando lo consulta el mundo.

## ¿Por qué existe?

Un servidor DNS completo como BIND es una pieza pesada: pensada para gestionar zonas enteras, con transferencias entre servidores, ficheros de zona con su propia sintaxis y una configuración considerable. Para "quiero que estos tres nombres resuelvan a estas tres IPs", es desproporcionado.

dnsmasq nace para el caso pequeño: un binario diminuto, un fichero de configuración de texto plano y comportamiento de reenvío por defecto. Lo que no sabe resolver, lo pregunta a otro y devuelve la respuesta.

> Si has editado alguna vez el fichero `hosts` de tu ordenador para que un dominio apunte a `localhost` durante unas pruebas, dnsmasq es esa misma idea pero servida en red: en lugar de tocar el fichero de cada máquina, se configura una vez en el servidor y todas las que lo usen como DNS ven la misma respuesta.

## ¿Cuándo y para qué se usa?

El caso que lo justifica en un VPS es el **DNS de horizonte partido** (*split-horizon DNS*): que `tienda.ejemplo.com` resuelva a la IP pública para los usuarios y a la IP de la [VPN](WireGuard.md) para quien administra.

Suena a rebuscamiento y resuelve un problema muy concreto. Si has restringido el acceso administrativo a la VPN, cuando te conectas al túnel y escribes el dominio en el navegador, el DNS público te devuelve la IP pública — y el firewall bloquea esa ruta, porque el acceso ahora solo se permite por `wg0`. Tendrías que usar la IP privada a mano, lo que rompe los certificados TLS (el certificado es para el dominio, no para `10.10.0.1`), las cookies y cualquier enlace absoluto.

Con dnsmasq resolviendo el nombre a la IP del túnel, todo funciona con el dominio de siempre: el certificado valida, las cookies se envían y los enlaces no se rompen.

---

## Instalar

```bash
apt install dnsmasq -y
```

Un aviso antes de continuar: en muchas distribuciones modernas, `systemd-resolved` ya está escuchando en el puerto 53, y dnsmasq no podrá arrancar. El síntoma es claro:

```bash
systemctl status dnsmasq
```

```
dnsmasq[1832]: failed to create listening socket for port 53: Address already in use
```

La configuración que veremos evita el choque atando dnsmasq **solo** a la interfaz de la VPN, que es donde `systemd-resolved` no escucha. Para comprobar quién ocupa el puerto:

```bash
ss -tulpn | grep :53
```

```
udp   UNCONN  0  0  127.0.0.53%lo:53  0.0.0.0:*  users:(("systemd-resolved",pid=612,fd=13))
```

## Configurar

Toda la configuración está en un fichero:

```bash
nano /etc/dnsmasq.conf
```

El fichero que viene de fábrica tiene cientos de líneas comentadas con documentación. Puedes editarlas o, más limpio, dejar solo lo que necesitas:

```ini
interface=wg0
listen-address=10.10.0.1
bind-interfaces

address=/tienda.ejemplo.com/10.10.0.1

server=1.1.1.1
server=8.8.8.8

domain-needed
bogus-priv
```

Línea por línea:

- **`interface=wg0`** — atiende consultas únicamente por la interfaz del túnel. Es la línea que hace que este servidor DNS no sea accesible desde internet.
- **`listen-address=10.10.0.1`** — la IP concreta en la que escucha, la del servidor dentro de la VPN.
- **`bind-interfaces`** — le dice a dnsmasq que se ate solo a esa interfaz en lugar de escuchar en todas y filtrar después. Es lo que evita el conflicto con `systemd-resolved` y, de paso, lo que impide que un fallo de configuración exponga el resolutor al exterior.
- **`address=/tienda.ejemplo.com/10.10.0.1`** — la regla que da sentido a todo: este dominio resuelve a la IP del túnel. La sintaxis usa barras como separador, y aplica también a todos los subdominios (`api.tienda.ejemplo.com` incluido). Se pueden poner tantas líneas `address=` como haga falta.
- **`server=1.1.1.1` y `server=8.8.8.8`** — a dónde reenviar lo que no sabe resolver. Sin esto, los clientes de la VPN con `DNS = 10.10.0.1` se quedarían sin poder resolver ningún dominio de internet.
- **`domain-needed`** — no reenvía consultas de nombres sin punto (como `servidor1`), que nunca tienen sentido en internet.
- **`bogus-priv`** — no reenvía consultas inversas de rangos privados. Ambas reducen tráfico inútil hacia los DNS públicos y evitan filtrar nombres internos.

## Un DNS que no puede arrancar antes que su interfaz

Aquí está la parte que hace fallar este montaje tras el primer reinicio, y el motivo es de orden de arranque.

`bind-interfaces` con `interface=wg0` obliga a dnsmasq a atarse a esa interfaz **en el momento de arrancar**. Pero `wg0` no existe hasta que WireGuard la crea. Si dnsmasq arranca primero —que es lo habitual, porque systemd lanza los servicios en paralelo—, no encuentra la interfaz y falla:

```
dnsmasq: unknown interface wg0
```

Lo desconcertante es que arrancarlo a mano después funciona perfectamente, así que el problema solo se manifiesta tras reiniciar el servidor. Se corrige declarando la dependencia con un fichero de anulación de systemd:

```bash
systemctl edit dnsmasq
```

Ese comando abre un editor sobre un fichero vacío. Se añade:

```ini
[Unit]
After=wg-quick@wg0.service
Requires=wg-quick@wg0.service
```

- **`After=`** fija el orden: no arranques dnsmasq hasta que WireGuard haya terminado.
- **`Requires=`** fija la dependencia: si WireGuard no arranca, dnsmasq tampoco lo intenta. Sin esta línea, `After=` solo ordena, y si WireGuard falla, dnsmasq arrancaría igualmente para fracasar.

`systemctl edit` guarda esto en `/etc/systemd/system/dnsmasq.service.d/override.conf`, un fichero aparte que las actualizaciones del paquete no tocan. Editar directamente el `.service` original tendría el mismo efecto hasta la siguiente actualización, que lo sobrescribiría.

Aplica y activa:

```bash
systemctl daemon-reload           # releer las unidades tras el cambio
systemctl enable dnsmasq
systemctl restart dnsmasq
```

`daemon-reload` es obligatorio tras modificar cualquier unidad; sin él, systemd sigue usando la definición anterior en memoria.

## Comprobar que resuelve

La prueba se hace **desde un cliente conectado a la VPN**, con `DNS = 10.10.0.1` en su configuración de WireGuard:

```bash
nslookup tienda.ejemplo.com
```

```
Server:   10.10.0.1
Address:  10.10.0.1#53

Name:     tienda.ejemplo.com
Address:  10.10.0.1
```

Las dos cosas que confirman que funciona: el `Server` es el servidor de la VPN (no el DNS de tu router), y la dirección devuelta es la del túnel, no la pública.

Y para confirmar que el reenvío también va, consulta cualquier dominio externo:

```bash
nslookup github.com
```

```
Server:   10.10.0.1
Address:  10.10.0.1#53

Non-authoritative answer:
Name:     github.com
Address:  140.82.121.4
```

Sigue respondiendo el servidor de la VPN, pero con la IP real de internet: la consulta se reenvió a `1.1.1.1` y volvió.

Desde el propio servidor, la comprobación hay que dirigirla explícitamente a dnsmasq, porque el servidor usa su propio resolutor:

```bash
dig @10.10.0.1 tienda.ejemplo.com +short
```

```
10.10.0.1
```

## Cuando la caché del cliente se interpone

El problema más frecuente al probar esto no está en el servidor: es que el cliente ya tenía la respuesta antigua guardada y no vuelve a preguntar. Conectas la VPN, la resolución sigue dando la IP pública, y el servidor está bien configurado.

Vaciar la caché DNS según el sistema:

```bash
# Windows
ipconfig /flushdns

# macOS
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder

# Linux con systemd-resolved
sudo resolvectl flush-caches
```

Los navegadores tienen además su propia caché, independiente de la del sistema. En Chrome se vacía en `chrome://net-internals/#dns`. Y para descartar el navegador por completo mientras depuras, usa `nslookup` o `dig`, que consultan directamente sin cachés intermedias.

## Registrar las consultas para depurar

Cuando algo no resuelve como esperas, el log de dnsmasq dice exactamente qué preguntó cada cliente y qué se le respondió. Se activa añadiendo al fichero de configuración:

```ini
log-queries
log-facility=/var/log/dnsmasq.log
```

```bash
systemctl restart dnsmasq
tail -f /var/log/dnsmasq.log
```

```
dnsmasq[2104]: query[A] tienda.ejemplo.com from 10.10.0.2
dnsmasq[2104]: config tienda.ejemplo.com is 10.10.0.1
dnsmasq[2104]: query[A] github.com from 10.10.0.2
dnsmasq[2104]: forwarded github.com to 1.1.1.1
```

Cómo leerlo: `config` significa que la respuesta salió de una línea `address=` tuya; `forwarded` que se preguntó fuera. Si esperabas `config` y ves `forwarded`, la regla `address=` no coincide con el nombre consultado — casi siempre por una errata o por un subdominio que no cubre.

**Desactiva `log-queries` cuando termines.** Registra absolutamente todas las consultas de todos los clientes, lo que llena el disco rápido y guarda un historial de navegación que probablemente no quieras conservar.

## Buenas prácticas avanzadas

- **Ata dnsmasq a la interfaz de la VPN y nunca a `0.0.0.0`.** Un resolutor DNS abierto a internet se convierte en amplificador de ataques de denegación de servicio: se envían consultas pequeñas con la IP de la víctima falsificada y tu servidor le manda respuestas grandes. Acabarías participando en un ataque sin saberlo y con tu proveedor cortándote el servicio. `interface=wg0` con `bind-interfaces` es lo que lo impide.
- **Baja el TTL de los nombres internos.** `local-ttl=60` hace que los clientes cacheen las respuestas internas solo un minuto. Con el valor por defecto, un cambio de IP tarda horas en propagarse a las máquinas que ya habían consultado, y el desajuste se diagnostica fatal porque unos clientes ven lo nuevo y otros lo viejo.
- **Usa `address=` para dominios enteros y `host-record=` para nombres sueltos.** `address=/ejemplo.com/10.10.0.1` captura el dominio **y todos sus subdominios**, lo que puede ser más de lo que querías: si un subdominio apunta legítimamente a un servicio externo, dejará de resolver dentro de la VPN. `host-record=tienda.ejemplo.com,10.10.0.1` afecta solo a ese nombre exacto.
- **Documenta que existe un DNS partido.** Es la clase de configuración que provoca horas de desconcierto a quien no sabe que está ahí: el mismo dominio devuelve IPs distintas según quién pregunte, `ping` funciona para unos y no para otros, y nada en la aplicación lo explica. Una nota en el README del despliegue ahorra esa investigación entera.
- **Plantéate si te hace falta antes de montarlo.** Con dos o tres personas administrando, unas líneas en el fichero `hosts` de cada portátil resuelven lo mismo sin añadir un servicio al servidor. dnsmasq compensa cuando hay varios nombres, varias personas o los nombres cambian; para un solo dominio y un solo administrador, es infraestructura que hay que mantener a cambio de poco.

## Recursos didácticos

- [DNS Checker](https://dnschecker.org/) — consulta un dominio desde servidores DNS de todo el mundo a la vez. Sirve para ver la cara pública de tu dominio y contrastarla con lo que devuelve tu resolutor interno, que es justo la diferencia que crea el DNS partido.
- [How DNS works](https://howdns.works/) — un cómic que explica el proceso completo de resolución, de la caché del navegador a los servidores raíz. Corto, divertido y sorprendentemente completo; deja muy claro en qué punto de la cadena se está metiendo dnsmasq.
- [`man dnsmasq`](https://thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html) — la referencia de todas las opciones, incluidas las de DHCP que aquí no se usan pero que explican por qué el programa es tan popular en redes pequeñas.

---

*En resumen: dnsmasq hace que un mismo dominio apunte a la IP del túnel para quien administra y a la pública para todos los demás — a cambio de recordar que arranca después de la VPN y solo escucha en ella.*
