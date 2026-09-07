# Tipos de notificación en Odoo

## ¿Qué es?

En Odoo, "notificar" no es una función: es **una familia de mecanismos distintos** —un aviso flotante en la esquina, un mensaje en el historial de un registro, un correo, una actividad pendiente, un empujón en tiempo real al navegador— cada uno con su API, su destinatario y su momento. Esta ficha es el mapa: qué mecanismos existen, en qué se diferencian y cuál elegir.

## ¿Por qué existe?

Porque "avisar a alguien" esconde preguntas muy distintas, y la respuesta a cada una lleva a un mecanismo diferente:

- ¿El aviso es **para quien acaba de pulsar el botón**, o para otra persona que ni está conectada?
- ¿Tiene que **quedar registrado** para siempre, o basta con que se vea tres segundos?
- ¿Debe **impedir** que la operación continúe, o solo informar?
- ¿Se entrega **ahora mismo**, o cuando la persona vuelva a abrir Odoo?

Un aviso de "guardado correctamente" y un correo al cliente con el número de seguimiento no se parecen en nada, aunque los dos sean "notificaciones". Odoo no intentó unificarlos bajo una sola API: construyó una pieza para cada caso. El precio es que hay que conocerlas para no usar la equivocada.

> Si vienes de una aplicación web hecha a mano, ya has escrito estas piezas por separado: un *toast* en el frontend, un `throw` que devuelve un 400, una tabla de comentarios, un `sendmail`, un WebSocket. Odoo trae las cinco de serie y espera que elijas.

## ¿Cuándo y para qué se usa?

Esta ficha se usa **antes** de escribir el código: para decidir qué mecanismo toca. El resto de la colección desarrolla cada uno.

El ejemplo que recorre toda la colección es una **tienda online** con un modelo propio `shop.order` (pedido), en un módulo llamado `shop`. Sus campos relevantes:

```python
class ShopOrder(models.Model):
    _name = 'shop.order'
    _description = 'Shop Order'

    name = fields.Char(string="Reference", required=True, copy=False, default="New")
    partner_id = fields.Many2one('res.partner', string="Customer", required=True)
    user_id = fields.Many2one('res.users', string="Salesperson", default=lambda self: self.env.user)
    state = fields.Selection(
        [('draft', "Draft"), ('confirmed', "Confirmed"), ('shipped', "Shipped"), ('cancel', "Cancelled")],
        default='draft', required=True)
    amount_total = fields.Monetary(string="Total")
    carrier_tracking_ref = fields.Char(string="Tracking Reference")
```

El pedido que iremos siguiendo es el **SO0042**, del cliente Marina Costa, con el comercial Jordi Vidal asignado.

---

## Los cuatro ejes que separan un mecanismo de otro

Antes del catálogo, los cuatro criterios que de verdad discriminan. Cuando dudes entre dos mecanismos, casi siempre es porque no has contestado a uno de estos:

| Eje | Extremos |
|---|---|
| **Destinatario** | Quien ejecuta la acción ↔ otras personas (usuarios internos, clientes externos) |
| **Persistencia** | Efímero, no queda rastro ↔ registrado en la base de datos y auditable |
| **Interrupción** | Solo informa ↔ **aborta la operación** y deshace los cambios |
| **Momento de entrega** | Dentro de la respuesta a la petición actual ↔ más tarde, cuando la persona esté disponible |

El eje que más errores causa es el de **interrupción**, porque en Odoo el mecanismo de aviso bloqueante es *lanzar una excepción*, y una excepción **deshace la transacción entera**. Es decir: el aviso y el `rollback` son la misma decisión. Se desarrolla en [Errores y avisos bloqueantes](Errores-y-Avisos.md).

## El catálogo

### 1. Notificación de interfaz (*toast*)

El rectángulo que aparece arriba a la derecha y se va solo en cuatro segundos. Es **efímero, para quien acaba de actuar**, y no deja rastro en ninguna parte.

Desde Python se devuelve como acción de cliente:

```python
def action_confirm(self):
    self.state = 'confirmed'
    return {
        'type': 'ir.actions.client',
        'tag': 'display_notification',
        'params': {
            'type': 'success',
            'message': _("Order %s confirmed.", self.name),
            'next': {'type': 'ir.actions.act_window_close'},
        },
    }
```

Al pulsar el botón, el pedido pasa a `confirmed` y aparece un aviso verde con "Order SO0042 confirmed.". Nadie más lo ve, y mañana no hay forma de saber que se mostró. → [Notificaciones de interfaz](Notificaciones-de-Interfaz.md)

### 2. Excepción (diálogo bloqueante)

Un modal que corta la operación. Es el único mecanismo que **impide** que algo pase:

```python
from odoo.exceptions import UserError

def action_ship(self):
    if not self.carrier_tracking_ref:
        raise UserError(_("Add a tracking reference before shipping order %s.", self.name))
    self.state = 'shipped'
```

Sin referencia de seguimiento, la persona ve un diálogo con el mensaje y **el pedido no cambia de estado**: la escritura se deshace junto con todo lo demás de la transacción. → [Errores y avisos bloqueantes](Errores-y-Avisos.md)

### 3. Aviso de `onchange` (no bloqueante)

Un aviso mientras se rellena un formulario, antes de guardar. No aborta nada: solo advierte.

```python
@api.onchange('partner_id')
def _onchange_partner_id_overdue(self):
    if self.partner_id and self.partner_id.has_overdue_invoices:
        return {'warning': {
            'title': _("Overdue invoices"),
            'message': _("This customer has overdue invoices."),
        }}
```

En cuanto se elige a Marina Costa como cliente, sale el aviso; el formulario sigue editable y se puede guardar igual. → [Errores y avisos bloqueantes](Errores-y-Avisos.md)

### 4. Mensaje en el *chatter* (`message_post`)

El historial que cuelga de cada registro. Es **persistente, auditable y multi-destinatario**: se guarda como un `mail.message` ligado al pedido y se entrega a sus seguidores, por bandeja interna o por correo según la preferencia de cada uno.

```python
self.message_post(
    body=_("Order confirmed. Estimated delivery in 48 hours."),
    subtype_xmlid='mail.mt_comment',
)
```

Dentro de un año, alguien que abra el pedido SO0042 verá ese mensaje con fecha y autor. → [Chatter, mensajes y seguidores](Chatter-y-Seguidores.md)

### 5. Notificación personal sin documento (`message_notify`)

Un aviso dirigido a personas concretas que **no** se muestra en el historial del registro. Sirve para "oye, mira esto" sin ensuciar la auditoría del documento:

```python
self.message_notify(
    partner_ids=self.user_id.partner_id.ids,
    body=_("Order %s needs your review.", self.name),
    subject=_("Review needed"),
)
```

Jordi recibe el aviso (campana o correo), y el chatter del pedido queda limpio. → [Chatter, mensajes y seguidores](Chatter-y-Seguidores.md)

### 6. Correo con plantilla (`mail.template`)

Cuando el destinatario está **fuera** de Odoo —un cliente que no tiene usuario— el canal es el correo, y el contenido vive en una plantilla editable sin tocar código:

```python
template = self.env.ref('shop.mail_template_order_confirmation')
template.send_mail(self.id, force_send=False)
```

Se crea un `mail.mail` en cola y sale con el siguiente ciclo del cron de correo. → [Plantillas de correo](Plantillas-de-Correo.md)

### 7. Actividad (`mail.activity`)

No es un aviso: es una **tarea asignada con fecha límite**, que aparece en el reloj del registro, en la vista de actividades y en el resumen diario. Es el mecanismo correcto cuando alguien tiene que *hacer* algo, no solo enterarse:

```python
self.activity_schedule(
    'mail.mail_activity_data_todo',
    date_deadline=fields.Date.today() + relativedelta(days=2),
    summary=_("Check payment for %s", self.name),
    user_id=self.user_id.id,
)
```

A Jordi le queda una actividad pendiente que no desaparece hasta que la marca como hecha. → [Actividades](Actividades.md)

### 8. Empujón en tiempo real (`bus.bus`)

Los siete mecanismos anteriores llegan cuando el navegador pregunta. El bus es lo contrario: el servidor **empuja** un mensaje a una sesión abierta ahora mismo, por WebSocket.

```python
self.user_id._bus_send('simple_notification', {
    'type': 'info',
    'title': _("New order"),
    'message': _("Order %s just came in.", self.name),
})
```

Si Jordi tiene Odoo abierto, le sale el aviso sin recargar nada. Si no lo tiene, **se pierde**. (Ficha propia **pendiente de escribir**.)

### 9. Notificación push del navegador

Un aviso del sistema operativo, con Odoo cerrado. Requiere que la persona haya dado permiso en su navegador y algo de infraestructura (claves VAPID, *service worker*). (Ficha propia **pendiente de escribir**.)

### 10. Sin escribir código

Buena parte de lo anterior se configura desde la interfaz con **acciones automatizadas**: "cuando un pedido pase a `shipped`, envía esta plantilla y crea una actividad". (Ficha propia **pendiente de escribir**.)

---

## Tabla de decisión

| Lo que quieres | Mecanismo | Ficha |
|---|---|---|
| Confirmar a quien pulsó el botón que fue bien | *Toast* `display_notification` | [Interfaz](Notificaciones-de-Interfaz.md) |
| Impedir una operación y explicar por qué | `UserError` / `ValidationError` | [Errores](Errores-y-Avisos.md) |
| Impedirla y ofrecer ir a arreglarlo | `RedirectWarning` | [Errores](Errores-y-Avisos.md) |
| Advertir al rellenar un campo, sin bloquear | `warning` en `@api.onchange` | [Errores](Errores-y-Avisos.md) |
| Dejar constancia en el historial del registro | `message_post` | [Chatter](Chatter-y-Seguidores.md) |
| Avisar a alguien sin ensuciar el historial | `message_notify` | [Chatter](Chatter-y-Seguidores.md) |
| Que el aviso llegue a quien siga el documento | `message_post` + subtipo + seguidores | [Chatter](Chatter-y-Seguidores.md) |
| Escribir a un cliente externo | `mail.template` + `send_mail` | [Plantillas](Plantillas-de-Correo.md) |
| Que quien reciba el aviso **tenga que hacer algo** | `activity_schedule` | [Actividades](Actividades.md) |
| Avisar en el mismo segundo, sin recargar | `_bus_send` | Pendiente |
| Avisar con Odoo cerrado | Push web | Pendiente |
| Lo mismo, pero configurable por quien administra | Acción automatizada | Pendiente |

## Los tres errores de elección más comunes

Elegir mal no da error: da un comportamiento raro que nadie relaciona con la decisión original.

- **`UserError` para avisar de algo que no debería bloquear.** Es el más frecuente y el más caro. Un `raise` deshace **toda** la transacción, no solo el aviso. Si validas 200 pedidos en lote y el número 137 tiene un aviso menor, con `UserError` pierdes los 136 anteriores. Lo correcto ahí es procesar todo y devolver un *toast* con el resumen, o dejar el detalle en el chatter.

- **`message_post` para dar *feedback* inmediato.** Funciona, pero deja una línea permanente en el historial del pedido por cada "operación completada". A los seis meses el chatter de SO0042 tiene 400 entradas de ruido y encontrar el mensaje que sí importaba —el cambio de dirección de entrega que pidió Marina— es imposible. El *feedback* efímero va en un *toast*.

- **`_bus_send` para algo que no puede perderse.** El bus entrega solo a sesiones conectadas en ese instante. Es perfecto para "hay un pedido nuevo" y catastrófico para "tu pedido ha sido rechazado": si la persona no tenía la pestaña abierta, ese mensaje no existió nunca. Lo que no puede perderse necesita persistencia: chatter, correo o actividad.

## Cómo se combinan en una operación real

En producción rara vez se usa un mecanismo solo. Confirmar el pedido SO0042 dispara cuatro a la vez, cada uno con su papel:

```python
def action_confirm(self):
    self.ensure_one()
    if self.state != 'draft':
        raise UserError(_("Only draft orders can be confirmed."))   # 1. bloquea si no procede

    self.state = 'confirmed'

    # 2. constancia en el historial, visible para los seguidores
    self.message_post(body=_("Order confirmed."), subtype_xmlid='mail.mt_note')

    # 3. correo al cliente, que no tiene usuario en Odoo
    self.env.ref('shop.mail_template_order_confirmation').send_mail(self.id)

    # 4. tarea con fecha para el comercial
    self.activity_schedule(
        'mail.mail_activity_data_todo',
        summary=_("Confirm stock availability"),
        user_id=self.user_id.id,
    )

    # 5. confirmación efímera para quien pulsó el botón
    return {
        'type': 'ir.actions.client',
        'tag': 'display_notification',
        'params': {'type': 'success', 'message': _("Order %s confirmed.", self.name)},
    }
```

Cada línea contesta a una pregunta distinta: el `raise` protege la máquina de estados, el `message_post` deja auditoría, la plantilla informa a quien está fuera, la actividad reparte trabajo y el `return` cierra el círculo con quien está mirando la pantalla.

## Dónde ve cada cosa la persona que la recibe

Saber en qué parte de la interfaz aterriza cada mecanismo ahorra mucho tiempo al depurar "no me llega nada":

| Mecanismo | Dónde aparece |
|---|---|
| *Toast* | Esquina superior derecha, se va solo |
| Excepción | Modal centrado, con botón de cerrar |
| Aviso de `onchange` | *Toast* amarillo (o modal si pides `type: 'dialog'`) |
| `message_post` | Chatter del registro **y** campana o correo de los seguidores |
| `message_notify` | Campana o correo, **no** en el chatter |
| Plantilla de correo | Bandeja de entrada del destinatario, fuera de Odoo |
| Actividad | Reloj del registro, vista de actividades, resumen por correo |
| Bus | *Toast*, o lo que decida el código JavaScript suscrito |
| Push web | Notificación del sistema operativo |

Que un mensaje llegue a la **campana** o al **correo** no lo decide tu código: lo decide el campo `notification_type` de cada usuario (`inbox` o `email`). Es la primera cosa que hay que mirar cuando alguien dice "a mí no me llegan los correos de Odoo". → [Chatter, mensajes y seguidores](Chatter-y-Seguidores.md)

## Buenas prácticas avanzadas

- **Decide la persistencia antes que el canal.** La pregunta útil no es "¿toast o correo?", sino "¿alguien va a necesitar saber dentro de seis meses que esto se avisó?". Si la respuesta es sí, el aviso tiene que pasar por la base de datos (`mail.message` o `mail.activity`) aunque además muestres un *toast*. Los equipos que se saltan este paso acaban reconstruyendo el historial a partir de los logs del servidor de correo, que solo guardan lo que salió, no lo que se decidió.
- **Un aviso que pide una acción sin fecha límite es un aviso que nadie hará.** Es la diferencia real entre `message_notify` y `activity_schedule`: la actividad tiene `date_deadline`, aparece en listados filtrables por vencimiento y bloquea visualmente el registro hasta que se cierra. Un mensaje en la campana se lee, se cierra y se olvida. Cuando el aviso implica trabajo, actividad; cuando es información, mensaje.
- **No notifiques dentro de un bucle sobre `self`.** Un `for record in self: record.message_post(...)` sobre 500 pedidos crea 500 mensajes y dispara 500 veces el cálculo de destinatarios. `message_post` funciona sobre un registro único a propósito, pero las plantillas tienen versión por lotes (`send_mail_batch`) y `activity_schedule` acepta directamente un conjunto de registros. Antes de escribir el bucle, comprueba si el método ya trabaja en lote.
- **Cuidado con notificar y luego lanzar una excepción en el mismo método.** El `raise` deshace la transacción, y con ella el `mail.message` que acababas de crear. Lo que **no** deshace son los efectos que ya salieron de la base de datos: un `send_mail(force_send=True)` ya entregó el correo al servidor SMTP y ese correo no vuelve. De ahí la recomendación de dejar los correos en cola (`force_send=False`, que es el valor por defecto) y no forzar el envío salvo que sepas por qué lo haces.
- **Los mensajes visibles se escriben envueltos en `_()`, y con parámetros, no concatenados.** `_("Order %s confirmed.", self.name)` es traducible; `_("Order ") + self.name + _(" confirmed.")` produce dos fragmentos que ningún idioma puede recomponer en su propio orden. La regla vale para todos los mecanismos de esta colección: `UserError`, `message_post`, el `params` del *toast* y el `summary` de una actividad.

## Documentación oficial

- [Odoo Developer Documentation — Mixins y modelos útiles](https://www.odoo.com/documentation/18.0/developer/reference/backend/mixins.html) — la referencia de los *mixins* `mail.thread` y `mail.activity.mixin`, que es donde vive la mitad de esta colección. Empieza por la tabla de campos que aporta cada uno.
- [Odoo Developer Documentation — Actions](https://www.odoo.com/documentation/18.0/developer/reference/backend/actions.html) — define `ir.actions.client`, la vía por la que Python devuelve un *toast* al navegador.
- [Código de `odoo/exceptions.py`](https://github.com/odoo/odoo/blob/18.0/odoo/exceptions.py) — son cien líneas y contienen la lista completa de excepciones que el cliente sabe interpretar. Cualquier otra excepción se muestra como "Server error" con traza.

## Recursos didácticos

- [Buscar `display_notification` en el código de Odoo](https://github.com/search?q=repo%3Aodoo%2Fodoo+%22display_notification%22+language%3APython&type=code) — buscar un patrón en el código fuente de Odoo es la forma más rápida de ver cómo se usa de verdad, con decenas de variantes reales.
- [Runbot de Odoo](https://runbot.odoo.com/) — instancias públicas y desechables de cada versión. Útil para ver un chatter, una actividad y un *toast* funcionando sin montar nada.

---

*En resumen: en Odoo no hay "una" notificación sino diez mecanismos, y elegir bien es contestar a cuatro preguntas —para quién, cuánto tiene que durar, si debe bloquear y cuándo debe llegar— antes de escribir la primera línea.*
