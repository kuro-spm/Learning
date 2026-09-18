# Mosquitto

## ¿Qué es?

Eclipse Mosquitto es un *broker* de código abierto que implementa el protocolo [MQTT](MQTT.md): el software que de verdad recibe, guarda y reenvía los mensajes que los clientes MQTT publican y consumen.

## ¿Por qué existe?

MQTT es solo una especificación —igual que HTTP describe un protocolo pero no un servidor web concreto—, así que hace falta un programa que lo hable. Mosquitto nació en 2009 con un objetivo muy concreto: ser esa implementación ligera. Está escrito en C, arranca en milisegundos, consume unos pocos megabytes de memoria y corre sin problemas en una Raspberry Pi o en el contenedor más pequeño de un clúster.

Esa ligereza es también su límite: Mosquitto no está pensado para repartir millones de conexiones concurrentes entre varios nodos, que es el terreno de brokers como EMQX o HiveMQ, con clustering y paneles de administración incorporados. Mosquitto asume que quien lo opera va a configurar la persistencia, la autenticación y el TLS a mano, con ficheros de texto.

> Piensa en Mosquitto como nginx sirviendo HTTP: hace una cosa —hablar el protocolo— de forma ligera y fiable, sin intentar ser una plataforma completa. Para servir esa misma comparación, EMQX o HiveMQ serían más parecidos a una plataforma de API management con clustering incluido.

## ¿Cuándo y para qué se usa?

En proyectos MQTT propios de tamaño pequeño o mediano: la pasarela de sensores de una casa inteligente, un sistema de domótica como Home Assistant, un gateway que recoge la telemetría de una flota de vehículos antes de reenviarla a la nube, o simplemente para desarrollar y probar cualquier sistema MQTT antes de decidir si hace falta algo con clustering. Sigue el mismo ejemplo que la [ficha de MQTT](MQTT.md): una casa con sensores de temperatura en el salón y la cocina, y un controlador que enciende la luz del salón.

Deja de encajar cuando el volumen de dispositivos conectados simultáneamente exige repartir la carga entre varios nodos con failover automático: ahí Mosquitto puede seguir siendo la pieza de cada nodo, pero coordinarlos en clúster no es algo que traiga de serie.

---

## Instalación y primer arranque

La forma más rápida de tener un broker en marcha es Docker:

```bash
docker run -d --name mosquitto \
  -p 1883:1883 \
  -v "$(pwd)/mosquitto.conf:/mosquitto/config/mosquitto.conf" \
  eclipse-mosquitto
```

Con un `mosquitto.conf` mínimo para pruebas locales:

```conf
listener 1883
allow_anonymous true
```

`allow_anonymous true` es imprescindible para este primer arranque —sin autenticación configurada, Mosquitto rechaza toda conexión por defecto desde la versión 2— pero **no** es una opción para nada que no sea tu propia máquina; se retoma más abajo en autenticación.

Con el broker escuchando, dos terminales bastan para comprobar que funciona:

```bash
# Terminal 1
mosquitto_sub -h localhost -t "casa/salon/temperatura" -v
```

```bash
# Terminal 2
mosquitto_pub -h localhost -t "casa/salon/temperatura" -m "21.5"
```

`mosquitto_sub` y `mosquitto_pub` vienen con el paquete `mosquitto-clients` (o dentro de la imagen Docker, ejecutándolos con `docker exec mosquitto ...`) y son la herramienta de diagnóstico más rápida para cualquier problema: si algo no llega por código, el primer paso siempre es reproducirlo con estos dos comandos.

## `mosquitto.conf`: las opciones que de verdad se tocan

El fichero de configuración es texto plano, una opción por línea. Estas son las que aparecen en cualquier despliegue real:

| Opción | Para qué |
|---|---|
| `listener <puerto> [dirección]` | El puerto y la interfaz donde escucha. Se puede repetir para tener varios (por ejemplo, 1883 sin TLS y 8883 con TLS a la vez). |
| `persistence true` | Guarda en disco las suscripciones, los mensajes retenidos y los mensajes con QoS 1/2 pendientes, para que sobrevivan a un reinicio del broker. |
| `persistence_location /mosquitto/data/` | Dónde escribe el fichero de persistencia (`mosquitto.db`). |
| `allow_anonymous false` | Exige autenticación a toda conexión. El valor correcto fuera de pruebas locales. |
| `password_file /mosquitto/config/passwd` | El fichero de usuarios y contraseñas, ver más abajo. |
| `acl_file /mosquitto/config/acl` | El fichero de control de acceso por topic, ver más abajo. |
| `log_dest stdout` | Envía el log a la salida estándar en lugar de a un fichero; cómodo dentro de un contenedor, donde `docker logs` ya lo recoge. |
| `max_queued_messages 1000` | Cuántos mensajes con QoS 1/2 guarda el broker para un cliente desconectado antes de empezar a descartarlos. |

Sin `persistence true`, todo lo anterior —mensajes retenidos incluidos— desaparece al reiniciar el contenedor. Es una sorpresa habitual la primera vez que se reinicia un Mosquitto de pruebas y el estado de los dispositivos "se olvida".

## Autenticación con usuario y contraseña

Con `allow_anonymous false`, cada cliente necesita credenciales. Se crean con `mosquitto_passwd`, que guarda las contraseñas cifradas, nunca en texto plano:

```bash
mosquitto_passwd -c /mosquitto/config/passwd sensor-salon
# pide la contraseña de forma interactiva

mosquitto_passwd /mosquitto/config/passwd controlador-luces
# sin -c: añade un usuario más sin machacar los anteriores
```

Y en `mosquitto.conf`:

```conf
allow_anonymous false
password_file /mosquitto/config/passwd
```

Los clientes pasan las credenciales al conectar:

```bash
mosquitto_pub -h localhost -t "casa/salon/temperatura" -m "21.5" \
  -u sensor-salon -P la-contraseña
```

```csharp
var opciones = new MqttClientOptionsBuilder()
    .WithClientId("sensor-salon")
    .WithTcpServer("localhost", 1883)
    .WithCredentials("sensor-salon", "la-contraseña")
    .Build();
```

## Control de acceso por topic (ACL)

Autenticarse dice **quién** eres; el fichero de ACL dice **qué topics** puedes leer o escribir. Sin él, cualquier usuario autenticado puede publicarse y suscribirse a lo que quiera, incluidos los topics de otros dispositivos.

```conf
# /mosquitto/config/acl

user sensor-salon
topic write casa/salon/temperatura

user controlador-luces
topic readwrite casa/salon/luz/#
topic read casa/salon/temperatura
```

Cada bloque `user` define lo que puede hacer ese usuario a partir de ahí, hasta el siguiente `user`. `topic read`, `topic write` o `topic readwrite` aceptan los mismos comodines `+` y `#` que una suscripción MQTT normal.

Para no mantener una entrada por dispositivo, `acl_file` admite un patrón con el usuario incrustado en el propio topic:

```conf
pattern readwrite casa/%u/#
```

`%u` se sustituye por el nombre de usuario de la conexión, así que un cliente llamado `salon` solo puede leer y escribir bajo `casa/salon/#`, y esa misma línea sirve para todos los dispositivos sin tocar el fichero al añadir uno nuevo.

Una conexión rechazada por ACL **no da ningún error visible al publicador**: el `PUBLISH` se acepta a nivel de conexión y el mensaje simplemente no llega a ningún suscriptor, porque el propio suscriptor tampoco puede suscribirse a un topic que su ACL no permite. Es el mismo tipo de fallo silencioso que un exchange de RabbitMQ sin bindings: todo parece funcionar y nada llega.

## TLS: cifrar el tráfico

Sin TLS, las credenciales y los mensajes viajan en texto plano por la red; aceptable para pruebas en `localhost`, no para nada que cruce internet.

```conf
listener 8883
cafile /mosquitto/config/certs/ca.crt
certfile /mosquitto/config/certs/server.crt
keyfile /mosquitto/config/certs/server.key
```

El cliente valida el certificado del servidor contra la misma autoridad certificadora:

```bash
mosquitto_pub -h broker.ejemplo.com -p 8883 --cafile ca.crt \
  -t "casa/salon/temperatura" -m "21.5" -u sensor-salon -P la-contraseña
```

Para autenticación mutua —que cada dispositivo se identifique con su propio certificado en lugar de usuario y contraseña— se añaden `require_certificate true` y `use_identity_as_username true` al listener, y cada cliente presenta su certificado con `--cert` y `--key`. Es lo habitual en flotas de dispositivos donde revocar el acceso de uno solo tiene que ser posible sin tocar a los demás.

## Bridge: conectar dos Mosquitto

Un *bridge* conecta dos brokers Mosquitto entre sí, de modo que uno reenvía al otro los mensajes de los topics que le interesan. Es el patrón habitual en IoT para tener un Mosquitto ligero en cada instalación (el *edge*) y reenviar solo lo relevante a un Mosquitto central en la nube:

```conf
# En el Mosquitto de la instalación local
connection puente-a-la-nube
address broker-central.ejemplo.com:8883
bridge_cafile /mosquitto/config/certs/ca.crt

topic casa/# out 1
```

`topic casa/# out 1` reenvía hacia el broker remoto (`out`) todo lo que se publique bajo `casa/#`, con QoS 1. La dirección puede ser `out` (local → remoto), `in` (remoto → local) o `both`. Esto evita exponer directamente a internet el broker de cada instalación: solo el broker central necesita ser alcanzable desde fuera, y cada bridge se conecta hacia él, no al revés.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `Connection Refused: not authorised` | Las credenciales son correctas pero el ACL no permite la acción, o el `ClientId` está bloqueado |
| `Connection Refused: bad user name or password` | Usuario o contraseña incorrectos, o `password_file` mal referenciado en `mosquitto.conf` |
| El cliente se desconecta solo cada pocos segundos, con `Client <id> already connected, closing old connection` | Dos procesos están usando el mismo `ClientId`; el broker solo permite una conexión activa por identificador y cierra la más antigua |
| Un mensaje se publica sin error y nadie lo recibe | El suscriptor no tiene permiso de lectura en el ACL para ese topic, o simplemente no hay nadie suscrito en ese momento |
| El estado retenido no cambia tras publicar un valor nuevo con `-r` | Se publicó a un topic distinto (revisar mayúsculas y niveles exactos); MQTT distingue topics carácter a carácter |
| Al reiniciar el contenedor desaparecen los mensajes retenidos y las suscripciones persistentes | Falta `persistence true` en `mosquitto.conf`, o el volumen de `persistence_location` no está montado fuera del contenedor |
| `SSL Error: unable to get local issuer certificate` | El cliente no tiene el certificado de la autoridad certificadora (`--cafile`) que validaría el certificado del broker |

## Buenas prácticas avanzadas

- **Un usuario o un certificado por dispositivo, nunca credenciales compartidas.** Con una contraseña común para todos los sensores, revocar el acceso de un dispositivo comprometido obliga a rotarla en todos a la vez. Con una credencial por dispositivo —reforzada con `pattern` en el ACL— revocar uno es borrar una línea.
- **Suscríbete a `$SYS/#` para vigilar el broker, no lo adivines desde fuera.** Mosquitto publica sus propias métricas como mensajes MQTT normales bajo `$SYS/broker/...`: clientes conectados, mensajes por segundo, uso de memoria. `mosquitto_sub -t '$SYS/#' -v` durante un minuto da más información real que cualquier suposición sobre si el broker está saturado.
- **Ajusta `max_queued_messages` y `max_inflight_messages` para clientes que se desconectan a menudo.** Sin límite, un dispositivo con QoS 1 que pasa horas sin conexión acumula mensajes en el broker hasta agotar su memoria, y eso afecta a todos los demás clientes conectados, no solo al dispositivo desconectado. Es el mismo error que un `x-max-length` ausente en una cola de RabbitMQ.
- **No expongas el broker de cada instalación directamente a internet: usa bridges hacia un broker central.** Un Mosquitto de borde solo necesita salir hacia fuera, nunca aceptar conexiones entrantes desde internet. Eso reduce la superficie de ataque a un único broker bien vigilado en lugar de uno por instalación.
- **`allow_anonymous true` es solo para el primer arranque en tu propia máquina.** Es la opción que hace que el "hola mundo" funcione a la primera, y también la que deja el broker abierto a cualquiera que encuentre el puerto si se olvida cambiar antes de exponerlo. Trátala como una nota temporal en el fichero de configuración, no como un valor por defecto aceptable.

## Documentación oficial

- [Documentación de Eclipse Mosquitto](https://mosquitto.org/documentation/) — la referencia completa de cada opción de `mosquitto.conf`, con las páginas de manual (`mosquitto.conf(5)`, `mosquitto-tls(7)`) enlazadas desde ahí.
- [Repositorio eclipse/mosquitto](https://github.com/eclipse/mosquitto) — el código fuente y, sobre todo, el `ChangeLog.txt`, útil para saber qué cambió entre la versión que tienes instalada y la última (el paso de `allow_anonymous` a `false` por defecto en la versión 2 es el ejemplo típico que rompe configuraciones antiguas).

## Recursos didácticos

- [test.mosquitto.org](https://test.mosquitto.org/) — un Mosquitto público de Eclipse para hacer pruebas sin instalar nada, con variantes sin TLS, con TLS y con autenticación. Sirve para probar un cliente nuevo en minutos antes de montar el broker propio.

---

*En resumen: Mosquitto es la pieza que hace real el protocolo MQTT — ligera de instalar, pero con la autenticación, el control de acceso por topic y el TLS a cargo de quien la opera, no activados de serie.*
