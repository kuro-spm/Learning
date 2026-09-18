# MQTT

## ¿Qué es?

MQTT (*Message Queuing Telemetry Transport*) es un protocolo de mensajería **publish/subscribe** diseñado para funcionar con muy poco ancho de banda y en dispositivos con pocos recursos: un mensaje de control puede pesar solo 2 bytes de cabecera.

## ¿Por qué existe?

MQTT nació en 1999 para monitorizar oleoductos a través de un enlace de satélite: caro por byte, con cortes frecuentes y sin margen para protocolos verbosos. HTTP no encajaba —cada lectura habría exigido abrir una conexión, montar cabeceras de texto y esperar una respuesta—, y los protocolos de mensajería empresarial como [AMQP](RabbitMQ.md), el que habla RabbitMQ, están pensados para servidores con memoria y CPU de sobra, no para un sensor con un microcontrolador de gama baja.

MQTT resuelve ese problema con un diseño minimalista: una conexión persistente, mensajes binarios muy compactos y un modelo en el que el emisor de un dato nunca necesita saber quién lo va a leer. Esa misma propiedad de desacoplamiento es el eje de toda la [mensajería asíncrona](Mensajeria-Asincrona.md); MQTT es una implementación concreta del patrón, optimizada para el caso en que "el otro lado" es un dispositivo con batería y una conexión inestable.

> Si ya conoces RabbitMQ o Service Bus, piensa en MQTT como su primo minimalista: el mismo desacoplamiento productor/consumidor a través de un intermediario, pero sin exchanges, sin colas con nombre propio y con una cabecera de protocolo diseñada para caber en un paquete de radio, no en una petición HTTP.

## ¿Cuándo y para qué se usa?

En cualquier escenario con muchos dispositivos que envían datos pequeños con frecuencia y una conectividad que no es de fiar: sensores de una casa inteligente que reportan temperatura cada pocos segundos, la telemetría de una flota de vehículos, o una app móvil que quiere enterarse al instante de que ha llegado un mensaje sin mantener HTTP abierto. El caso de uso que seguiremos en esta ficha es el primero: una casa inteligente con sensores de temperatura en el salón y la cocina, y un controlador que enciende y apaga la luz del salón.

Fuera de ese perfil —servicios backend con buena conectividad que necesitan enrutado complejo, transacciones o colas con reparto de carga entre varios procesos— RabbitMQ o Service Bus encajan mejor: MQTT no tiene el concepto de exchange ni de reparto exclusivo de un mensaje entre varios consumidores.

---

## Publicar y suscribirse: el modelo con el broker en medio

Como en cualquier mensajería con intermediario, **nadie habla directamente con nadie**. Un sensor publica su lectura en un *topic*, un broker MQTT la recibe y la reenvía a quien esté suscrito a ese topic. El sensor no sabe si hay cero, uno o cien suscriptores.

```
sensor-salon  --publica-->  broker MQTT  --reenvía-->  controlador-luces
                                        --reenvía-->  panel-movil
```

Para probarlo hace falta un broker en marcha. La [ficha de Mosquitto](Mosquitto.md) cubre cómo instalarlo y configurarlo; para los ejemplos de aquí basta con tenerlo escuchando en `localhost:1883` (o usar el broker público de pruebas `test.mosquitto.org`, sin instalar nada). Con el broker `mosquitto` vienen dos herramientas de línea de comandos que sirven para experimentar sin escribir código:

```bash
# Terminal 1: queda escuchando
mosquitto_sub -h localhost -t "casa/salon/temperatura" -v
```

```bash
# Terminal 2: publica una lectura
mosquitto_pub -h localhost -t "casa/salon/temperatura" -m "21.5"
```

El primer terminal imprime `casa/salon/temperatura 21.5` en cuanto se ejecuta el segundo. Ese es el protocolo completo en su forma más simple: un topic y un valor.

Desde una aplicación .NET, la librería habitual es [MQTTnet](https://github.com/dotnet-iot/MQTTnet):

```bash
dotnet add package MQTTnet
```

```csharp
var factory = new MqttFactory();
using var cliente = factory.CreateMqttClient();

var opciones = new MqttClientOptionsBuilder()
    .WithClientId("controlador-luces")
    .WithTcpServer("localhost", 1883)
    .Build();

await cliente.ConnectAsync(opciones);
```

Suscribirse y reaccionar a los mensajes que lleguen:

```csharp
cliente.ApplicationMessageReceivedAsync += async e =>
{
    var topic = e.ApplicationMessage.Topic;
    var valor = e.ApplicationMessage.ConvertPayloadToString();
    Console.WriteLine($"{topic}: {valor}");
    await Task.CompletedTask;
};

await cliente.SubscribeAsync(
    new MqttTopicFilterBuilder().WithTopic("casa/salon/temperatura").Build());
```

Y publicar un mensaje propio, por ejemplo para encender la luz:

```csharp
var mensaje = new MqttApplicationMessageBuilder()
    .WithTopic("casa/salon/luz/set")
    .WithPayload("ON")
    .Build();

await cliente.PublishAsync(mensaje);
```

## Topics: la jerarquía y sus comodines

Un topic es una cadena de texto con niveles separados por `/`, como una ruta de archivos: `casa/salon/temperatura`. No hace falta crearlo de antemano —el primer `PUBLISH` a un topic nuevo simplemente empieza a existir— y no hay límite de profundidad más allá del tamaño máximo del paquete.

Lo que hace útil esa jerarquía son los **comodines**, que solo se usan al suscribirse (nunca al publicar):

| Comodín | Significa | Ejemplo |
|---|---|---|
| `+` | Un nivel cualquiera, exactamente uno | `casa/+/temperatura` recibe `casa/salon/temperatura` y `casa/cocina/temperatura`, pero no `casa/salon/planta1/temperatura` |
| `#` | Cero o más niveles, solo al final | `casa/salon/#` recibe `casa/salon/temperatura`, `casa/salon/luz/estado` y todo lo que cuelgue de `casa/salon/` |

```bash
# Todas las temperaturas de la casa, sin importar la habitación
mosquitto_sub -h localhost -t "casa/+/temperatura" -v
```

Diseñar bien la jerarquía es lo que hace que esos comodines sirvan de algo después: si el identificador del dispositivo va siempre en la misma posición (`casa/<habitación>/<magnitud>`), un `+` en esa posición cubre todos los dispositivos sin tener que enumerarlos. Un `#` en el nivel raíz de todo el sistema (`#` a secas) recibe absolutamente todo lo que pase por el broker, incluidos los topics internos `$SYS/...` si el broker los incluye explícitamente en la suscripción — normalmente no conviene usarlo así salvo para depurar.

## Los tres niveles de calidad de servicio (QoS)

MQTT deja elegir, mensaje a mensaje, cuánto esfuerzo hace el protocolo por garantizar la entrega. Es una decisión explícita en cada `PUBLISH`, no una configuración global del broker.

| QoS | Garantía | Cómo lo consigue |
|---|---|---|
| **0** | Como mucho una vez | Se envía y no se espera confirmación. Si se pierde el paquete, se pierde el dato. |
| **1** | Al menos una vez | El receptor confirma con `PUBACK`; si no llega a tiempo, el emisor reenvía. Puede llegar duplicado. |
| **2** | Exactamente una vez | Un intercambio de cuatro pasos (`PUBLISH` → `PUBREC` → `PUBREL` → `PUBCOMP`) evita que el mismo mensaje se entregue dos veces en ese salto. |

```csharp
var mensaje = new MqttApplicationMessageBuilder()
    .WithTopic("casa/salon/temperatura")
    .WithPayload("21.5")
    .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)   // QoS 1
    .Build();
```

QoS 0 es el que usan los sensores que publican cada pocos segundos: si se pierde una lectura de temperatura, la siguiente llega enseguida y no importa. QoS 1 es el nivel por defecto razonable para casi todo lo demás —un comando "enciende la luz" que sí quieres que llegue—. QoS 2 tiene un coste real de latencia y de estado guardado en el broker por cada mensaje en vuelo, así que se reserva para lo que de verdad no puede duplicarse, como un comando de facturación o de apertura de una cerradura.

Un matiz que conviene no perder de vista, en línea con lo que ya se explica para la [mensajería asíncrona en general](Mensajeria-Asincrona.md#las-garantías-de-entrega): QoS 2 evita el duplicado **en ese salto concreto** entre dos participantes que mantienen su sesión intacta. Si el cliente se reconecta con la sesión limpia a medio camino, o el mensaje atraviesa un [bridge](Mosquitto.md#bridge-conectar-dos-mosquitto) hacia otro broker, la garantía no se propaga automáticamente de extremo a extremo. Para un dato que de verdad no se puede procesar dos veces, la idempotencia en quien consume sigue siendo la red de seguridad.

## Mensajes retenidos y el testamento de conexión

Dos mecanismos que no tienen equivalente directo en RabbitMQ o Service Bus, y que son casi siempre la razón por la que un proyecto de IoT elige MQTT.

**Mensaje retenido (*retained*).** Un `PUBLISH` marcado como retenido se guarda en el broker como "el último valor conocido" de ese topic. Cualquier cliente que se suscriba **después**, aunque el mensaje se publicara hace horas, lo recibe inmediatamente al conectarse:

```bash
mosquitto_pub -h localhost -t "casa/salon/luz/estado" -m "ON" -r
```

Sin retención, un panel de control que arranca no sabría si la luz está encendida hasta la siguiente vez que alguien la cambie. Con el mensaje retenido, lo sabe en el instante en que se suscribe. Para borrar un valor retenido se publica un payload vacío con el flag de retención activo sobre el mismo topic; publicar sin el flag de retención no lo borra, solo añade un mensaje normal por encima.

**Testamento de conexión (*Last Will and Testament*, LWT).** Al conectarse, un cliente puede dejarle al broker un mensaje de repuesto —topic, payload y QoS— para que lo publique **en su nombre** si la conexión se corta de forma anómala (se agota el *keep-alive*, cae la red) y no de forma ordenada (un `DISCONNECT` explícito no dispara el testamento):

```csharp
var opciones = new MqttClientOptionsBuilder()
    .WithClientId("sensor-salon")
    .WithTcpServer("localhost", 1883)
    .WithWillTopic("casa/salon/sensor-temperatura/estado")
    .WithWillPayload("offline")
    .WithWillRetain(true)
    .Build();
```

Combinado con retención, el testamento es el patrón estándar para saber si un dispositivo está vivo: se suscribe a `casa/+/+/estado`, y cuando un sensor pierde la conexión sin avisar, el propio broker publica `offline` por él.

## Sesiones: qué sobrevive a una desconexión

Un sensor de batería se desconecta y reconecta constantemente, y lo que pase con sus suscripciones y sus mensajes pendientes en ese hueco depende de la **sesión**.

Con `CleanSession: false` (o, en MQTT 5, `CleanStart: false` junto con un `SessionExpiryInterval`), el broker recuerda las suscripciones del cliente y guarda los mensajes con QoS 1 o 2 que le correspondían mientras estaba desconectado, hasta que vuelva a conectarse con el mismo `ClientId`:

```csharp
var opciones = new MqttClientOptionsBuilder()
    .WithClientId("sensor-salon")
    .WithTcpServer("localhost", 1883)
    .WithCleanSession(false)
    .Build();
```

Con `CleanSession: true`, cada conexión empieza de cero: hay que volver a suscribirse y cualquier mensaje acumulado durante la desconexión se pierde. Es la trampa más común al depurar una reconexión automática: el cliente parece funcionar —se reconecta solo tras un corte de red— pero silenciosamente ha perdido todo lo que le tocaba recibir mientras estaba fuera, porque la lógica de reconexión crea una sesión limpia en cada intento.

El otro mecanismo temporal es el ***keep-alive***: el cliente pacta un intervalo (por ejemplo, 60 segundos) y, si no ha enviado nada en ese tiempo, manda un `PINGREQ` vacío solo para mantener la conexión viva. Si el broker no oye nada del cliente en 1,5 veces ese intervalo, lo da por caído y dispara su testamento.

## Buenas prácticas avanzadas

- **Diseña la jerarquía de topics para quien se suscribe, no para quien publica.** Poner el identificador del dispositivo siempre en la misma posición (`casa/<habitación>/<magnitud>`, nunca `<magnitud>/casa/<habitación>` en unos sitios y al revés en otros) es lo que permite que un `+` cubra todos los dispositivos futuros sin tocar código. Cambiar la posición de un nivel después de tener dispositivos en producción obliga a migrar suscripciones y reglas de ACL a la vez.
- **QoS 1 por defecto, QoS 2 solo si el duplicado es inaceptable.** El apretón de manos de cuatro pasos de QoS 2 multiplica el tráfico y el estado que el broker guarda por mensaje en vuelo; en un despliegue con miles de sensores eso se nota en memoria del broker antes que en ningún otro sitio. Casi todo lo que "parece necesitar" QoS 2 se resuelve con QoS 1 y una comprobación de idempotencia barata en quien consume.
- **Un mensaje retenido obsoleto miente para siempre.** Si un dispositivo se retira sin limpiar su último valor retenido, cualquier cliente nuevo que se suscriba seguirá recibiendo ese dato como si fuera actual, sin ningún aviso de que está caducado. Al dar de baja un dispositivo, publica un retenido vacío en sus topics como parte del proceso.
- **Usa suscripciones compartidas (`$share/`) para repartir carga entre varios consumidores, no una suscripción por instancia.** MQTT 5 introduce el prefijo `$share/<grupo>/<topic>`: varias instancias suscritas al mismo grupo se reparten los mensajes en lugar de recibirlos todas, el equivalente MQTT a varios consumidores compitiendo por la misma cola en RabbitMQ. Sin esto, escalar un consumidor horizontalmente significa que cada instancia procesa el mismo mensaje por separado.
- **No confundas testamento con desconexión esperada.** Un despliegue que reinicia el proceso del sensor sin enviar `DISCONNECT` (matarlo con `SIGKILL`, por ejemplo) dispara el testamento igual que una caída de red real. Si el reinicio es rutinario, cerrar la conexión de forma ordenada evita que el sistema de monitorización marque como "offline" un dispositivo que en realidad solo se está actualizando.

## Documentación oficial

- [Especificación MQTT 5.0 (OASIS)](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html) — la fuente normativa: el significado exacto de cada tipo de paquete, cada flag y cada código de razón.
- [mqtt.org](https://mqtt.org/) — la página del protocolo, con una introducción a alto nivel, la lista de implementaciones (brokers y clientes) y enlaces a la especificación de cada versión.

## Recursos didácticos

- [MQTT Essentials, de HiveMQ](https://www.hivemq.com/mqtt-essentials/) — la serie de artículos más citada para aprender el protocolo desde cero, con un capítulo por concepto (topics, QoS, retención, testamento) y diagramas claros de cada intercambio de paquetes.
- [MQTT Explorer](http://mqtt-explorer.com/) — un cliente gráfico que conecta a cualquier broker y muestra el árbol de topics en vivo, con los valores retenidos marcados aparte. Verlo actualizarse en tiempo real mientras publicas desde `mosquitto_pub` es la forma más rápida de que los comodines y la retención dejen de ser abstractos.

---

*En resumen: MQTT desacopla publicador y suscriptor igual que cualquier mensajería con broker, pero recortado hasta caber en un sensor con batería — con topics jerárquicos, tres niveles de QoS explícitos por mensaje y dos mecanismos, la retención y el testamento, pensados para que alguien que se conecta tarde siga sabiendo el estado del mundo.*
