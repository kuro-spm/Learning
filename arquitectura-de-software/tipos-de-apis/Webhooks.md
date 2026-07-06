# Webhooks

## ¿Qué es?

Un **webhook** es una API "al revés": en lugar de que tú llames a un servicio para preguntar si algo ha pasado, es el **servicio quien te llama a ti** (a una URL que tú le has dado) en el momento en que ocurre el evento. Por eso a veces se les llama "APIs inversas" o *callbacks* HTTP.

## ¿Por qué existe?

Imagina que esperas a que se confirme un pago. Sin webhooks, tendrías que preguntar a la pasarela una y otra vez: "¿ya está pagado?... ¿y ahora?". Eso es ineficiente y siempre llega tarde. El webhook invierte la relación: tú le das al servicio una URL y le dices "avísame tú cuando el pago se confirme". Cuando ocurre, el servicio envía una petición HTTP a tu URL con los datos del evento.

> El polling es llamar al restaurante cada 5 minutos para ver si tu pedido está listo. El webhook es dejar tu número para que **te llamen ellos** cuando esté listo. Dejas de gastar llamadas y te enteras justo a tiempo.

## ¿Cuándo y para qué se usa?

Siempre que necesites reaccionar a eventos que ocurren en **otro sistema**, sin saber cuándo pasarán:

- Una **pasarela de pago** te avisa cuando un cobro se completa o falla.
- Un **repositorio de código** te notifica cuando alguien sube cambios.
- Una **plataforma de envíos** te informa cuando un paquete cambia de estado.
- Un **formulario** externo te manda los datos cada vez que alguien lo rellena.

## Lo mínimo que necesitas saber

**1. Tú expones una URL pública (el *endpoint* del webhook)**

Es una ruta normal de tu servidor preparada para **recibir** peticiones del otro sistema.

```http
POST /webhooks/payments
{ "event": "payment.succeeded", "orderId": 87, "amount": 59.90 }
```

**2. Registras esa URL en el servicio externo**

Le dices al proveedor a dónde llamar y para qué eventos.

```http
POST https://api.pasarela.com/webhooks
{ "url": "https://miapp.com/webhooks/payments", "events": ["payment.succeeded"] }
```

**3. Tu endpoint procesa el evento y responde rápido**

Confirmas la recepción con un `200 OK` cuanto antes; el trabajo pesado, mejor en segundo plano.

```csharp
[HttpPost("/webhooks/payments")]
public IActionResult Recibir(PaymentEvent evento)
{
    encolarProcesamiento(evento);   // procesa después, no bloquees la respuesta
    return Ok();                    // responde ya, para que no te reintenten
}
```

**4. Verifica que el aviso es legítimo (firma)**

Como tu URL es pública, cualquiera podría llamarla. Los proveedores firman cada webhook con un secreto compartido para que compruebes que viene de ellos.

```csharp
if (!FirmaValida(request.Body, request.Headers["X-Signature"], secreto))
    return Unauthorized();   // descarta avisos que no sean del proveedor
```

**5. Prepárate para repeticiones y reintentos**

Si tu servidor no responde `200`, el proveedor suele reintentar. Eso significa que un mismo evento puede llegarte **más de una vez**: hazlo *idempotente* (procesar dos veces el mismo evento no debe duplicar el efecto).

## Lo que NO hace

- **No es algo que tú "llamas"** — es el otro quien te llama; tu papel es exponer y escuchar.
- **No garantiza una sola entrega** — puede llegar duplicado; diseña pensando en ello.
- **No sirve si tu servidor no es accesible** — el proveedor necesita poder alcanzar tu URL pública por internet.
- **No es para tiempo real continuo** — es para eventos puntuales; para un flujo constante usa [WebSockets](WebSockets.md) o [SSE](Server-Sent-Events.md).

## Buenas prácticas avanzadas

- **Verifica el *timestamp* además de la firma** — una firma válida no basta: un atacante que capture un webhook legítimo puede reenviarlo idéntico días después (*replay attack*) y la firma seguirá cuadrando. Los proveedores serios incluyen la marca de tiempo dentro de lo firmado; rechaza avisos con más de unos minutos de antigüedad.
- **No confíes en el orden: consulta el estado real** — los reintentos y el paralelismo hacen que `payment.succeeded` pueda llegarte *antes* que `payment.created`. Si tu lógica asume la secuencia, tendrás bugs fantasma imposibles de reproducir. El patrón robusto es tratar el webhook como un timbre, no como la verdad: al recibirlo, consulta a la API del proveedor el estado actual del recurso y actúa sobre eso.
- **Ten una reconciliación de respaldo** — algún webhook se perderá (caída tuya justo cuando el proveedor agotó sus reintentos, un despliegue en mal momento). Un proceso periódico que compare tu estado con el del proveedor ("pedidos que sigo teniendo como *pendientes de pago* desde hace más de una hora") convierte una pérdida silenciosa de dinero en un retraso de minutos.
- **Guarda los *payloads* reales y reprodúcelos en tests** — los cuerpos que envía el proveedor nunca coinciden del todo con su documentación (campos extra, nulos inesperados, formatos de fecha). Almacena los webhooks crudos recibidos (con datos sensibles enmascarados) y úsalos como casos de test de tu endpoint; en desarrollo, un túnel tipo ngrok te permite recibir los de verdad en tu máquina.

---

*En resumen: un webhook es una API al revés —das tu número y el servicio te llama cuando ocurre el evento— ideal para enterarte de pagos, envíos o cambios sin estar preguntando todo el rato.*
