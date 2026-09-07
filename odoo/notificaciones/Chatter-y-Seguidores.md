# Chatter, mensajes y seguidores

## ¿Qué es?

El *chatter* es el bloque de conversación e historial que cuelga de la ficha de un registro en Odoo: mensajes, notas internas, cambios de campos, adjuntos y la lista de seguidores. Por debajo es un *mixin* llamado `mail.thread`, y su API central es un método: `message_post`. Los mensajes que se publican ahí **se guardan en la base de datos y se entregan a los seguidores del registro**, cada uno por su canal preferido: la bandeja interna de Odoo o el correo.

## ¿Por qué existe?

Porque en un ERP la pregunta "¿por qué está esto así?" se hace todo el tiempo, y sin historial no tiene respuesta. Quién cambió el precio del pedido SO0042, si al cliente se le avisó del retraso, qué dijo cuando contestó, quién decidió aplicar el descuento.

Antes de que existiera el chatter, esa información vivía en cadenas de correo dentro de los buzones personales de cada uno. El chatter la mueve **al lado del registro**: la conversación sobre el pedido está en el pedido, y quien abra la ficha dentro de dos años la encuentra sin depender de que alguien conserve un correo.

Además, resuelve el problema de la distribución: cada registro tiene una lista de **seguidores**, y quien publica un mensaje no tiene que saber a quién avisar. Escribes en el pedido y Odoo decide quién debe enterarse y por qué canal.

> Si has usado los comentarios de una incidencia de GitHub o de Jira, el chatter es lo mismo llevado a todos los modelos del sistema: hilo de comentarios, lista de *watchers*, notificaciones a quien sigue el hilo y registro automático de los cambios de campos.

## ¿Cuándo y para qué se usa?

Siempre que el aviso deba **quedar registrado** o llegar a **más de una persona**:

- Documentar decisiones y comunicaciones: "cliente avisado del retraso por teléfono".
- Registrar automáticamente lo que hace el sistema: "pedido confirmado", "envío generado".
- Conversar con el cliente: en un chatter, un mensaje enviado sale por correo y **la respuesta vuelve al mismo hilo**.
- Dejar constancia auditable de un cambio de estado.
- Avisar a quien sigue el documento sin tener que saber quién es.

Seguimos con el modelo `shop.order` de la tienda online, el pedido **SO0042** del cliente Marina Costa y el comercial Jordi Vidal.

---

## Activar el chatter en un modelo propio

Son tres pasos: heredar el *mixin*, añadir el bloque a la vista y —esto se olvida— dar permiso de lectura a `mail.message` (que ya viene con el módulo `mail`, así que basta con declarar la dependencia).

**1. Heredar `mail.thread`** en el modelo:

```python
class ShopOrder(models.Model):
    _name = 'shop.order'
    _description = 'Shop Order'
    _inherit = ['mail.thread', 'mail.activity.mixin']

    name = fields.Char(string="Reference", required=True, copy=False, default="New")
    partner_id = fields.Many2one('res.partner', string="Customer", required=True, tracking=True)
    user_id = fields.Many2one('res.users', string="Salesperson", tracking=True,
                              default=lambda self: self.env.user)
    state = fields.Selection(
        [('draft', "Draft"), ('confirmed', "Confirmed"), ('shipped', "Shipped"), ('cancel', "Cancelled")],
        default='draft', required=True, tracking=True)
```

Con solo eso, el modelo gana los campos `message_ids`, `message_follower_ids`, `message_partner_ids` y una treintena de métodos. Se ha añadido también `mail.activity.mixin`, que aporta las [actividades](Actividades.md): van casi siempre juntos.

**2. Declarar la dependencia** en el `__manifest__.py`, o nada de lo anterior existirá:

```python
{
    'name': "Shop",
    'depends': ['mail'],
    ...
}
```

**3. Añadir el bloque a la vista de formulario**, después de `</sheet>`:

```xml
<form>
    <sheet>
        <group>
            <field name="partner_id"/>
            <field name="user_id"/>
            <field name="state"/>
        </group>
    </sheet>
    <chatter/>
</form>
```

En Odoo 18 el chatter se declara con la etiqueta propia `<chatter/>`. (En versiones anteriores a la 17 se escribía como un `<div class="oe_chatter">` con los tres campos dentro; si ves ese patrón en un módulo, es código antiguo.) Acepta algún atributo, como `<chatter reload_on_follower="True"/>` para recargar el registro cuando cambia la lista de seguidores.

Con estos tres pasos, la ficha del pedido SO0042 ya tiene su caja de mensajes, sus botones de "Enviar mensaje" y "Registrar nota", su contador de seguidores y el registro automático de los cambios en `partner_id`, `user_id` y `state` —eso último gracias a `tracking=True`, que se explica más abajo—.

## Las tres piezas de datos

Entender el chatter es entender tres modelos:

| Modelo | Qué guarda |
|---|---|
| `mail.message` | El mensaje: cuerpo, autor, fecha, tipo, subtipo, a qué registro pertenece |
| `mail.followers` | Quién sigue qué registro, y **a qué subtipos** está suscrito |
| `mail.notification` | Una fila por destinatario y mensaje: si se le envió, por qué canal y si lo ha leído |

La separación importa porque explica el comportamiento raro más común: un mensaje **existe** (hay `mail.message`) pero **nadie lo recibió** (no hay `mail.notification`). Eso pasa cuando el registro no tiene seguidores o cuando el subtipo del mensaje no coincide con los que sus seguidores han marcado. El mensaje se ve en el chatter de quien abra la ficha, y no llega a ninguna bandeja.

## `message_post`: el método central

Publica un mensaje en el hilo de **un** registro y devuelve el `mail.message` creado.

```python
def action_confirm(self):
    self.ensure_one()
    self.state = 'confirmed'
    self.message_post(
        body=_("Order confirmed. Estimated delivery within 48 hours."),
        subtype_xmlid='mail.mt_comment',
    )
```

En Odoo 18, **todos los argumentos de `message_post` son de palabra clave**: `self.message_post(_("texto"))` lanza un `TypeError`. Hay que escribir `body=...` siempre.

Los parámetros que se usan de verdad:

| Parámetro | Para qué |
|---|---|
| `body` | El cuerpo. Texto plano o HTML (ver más abajo) |
| `subject` | Asunto. Se usa como asunto del correo cuando el mensaje sale por ahí |
| `subtype_xmlid` | El subtipo, que decide **quién recibe** el mensaje. Por defecto, nota |
| `partner_ids` | Personas concretas a notificar, además de los seguidores |
| `message_type` | `comment` (de una persona) o `notification` (del sistema) |
| `attachment_ids` | Adjuntos ya existentes en `ir.attachment` |
| `attachments` | Adjuntos nuevos, como lista de tuplas `(nombre, contenido)` |
| `author_id` | Publicar en nombre de otro *partner* |
| `email_layout_xmlid` | La plantilla de correo que envuelve el mensaje al enviarlo |

### El cuerpo y el escapado de HTML

`body` acepta HTML, pero desde Odoo 17 una cadena normal **se escapa**: si pasas `"<b>Urgent</b>"`, en el chatter se lee literalmente `<b>Urgent</b>`. Para que el HTML se interprete hay que marcarlo con `Markup`:

```python
from markupsafe import Markup

# Texto plano: seguro, y lo habitual
self.message_post(body=_("Order confirmed."))

# HTML deliberado: hay que envolverlo
self.message_post(body=Markup("<p>Order confirmed. <b>Delivery in 48h.</b></p>"))

# ❌ Nunca así: si el nombre del cliente contiene HTML, se inyecta en el chatter
self.message_post(body=Markup(f"<p>Customer: {self.partner_id.name}</p>"))

# ✅ Con Markup y formateo: los valores interpolados se escapan solos
self.message_post(body=Markup("<p>Customer: %s</p>") % self.partner_id.name)
```

La última forma es la correcta: `Markup` con el operador `%` escapa lo que se interpola y deja intacto el HTML de la plantilla. Es la misma idea que parametrizar una consulta SQL en vez de concatenarla.

### Adjuntar un fichero al mensaje

Útil para dejar el PDF junto a la conversación:

```python
pdf_content, _content_type = self.env['ir.actions.report']._render_qweb_pdf(
    'shop.report_order', self.ids)
self.message_post(
    body=_("Order confirmation attached."),
    attachments=[('%s.pdf' % self.name, pdf_content)],
)
```

`attachments` recibe el contenido **sin codificar en base64** (Odoo se encarga) y crea los `ir.attachment` ligados al pedido. Si el adjunto ya existe como registro, se usa `attachment_ids` con su ID.

## Los subtipos: quién recibe el mensaje

Esta es la parte que casi nadie explica y la que provoca el 80 % de los "¿por qué no me llega nada?".

Un **subtipo** (`mail.message.subtype`) es una etiqueta de clasificación del mensaje, y cada seguidor elige **a qué subtipos se suscribe**. El cálculo de destinatarios es, simplificando: *seguidores suscritos al subtipo del mensaje* + *lo que pases en `partner_ids`*.

Los dos subtipos que se usan a diario vienen del módulo `mail`:

| Subtipo | XML ID | Significado |
|---|---|---|
| Note | `mail.mt_note` | Nota **interna**. No se manda a contactos externos |
| Discussions | `mail.mt_comment` | Mensaje de la conversación. Se entrega a los seguidores, incluidos los externos |

```python
# Nota interna: queda en el historial, la ven los empleados, NO sale al cliente
self.message_post(body=_("Customer called: wants delivery after the 15th."),
                  subtype_xmlid='mail.mt_note')

# Mensaje: se entrega a los seguidores del hilo, cliente incluido
self.message_post(body=_("Your order has been shipped."),
                  subtype_xmlid='mail.mt_comment')
```

La diferencia es de visibilidad y de entrega, y confundirla tiene consecuencias reales en las dos direcciones: una nota interna que quería ser un aviso al cliente se queda dentro y el cliente nunca se enteró; un comentario que quería ser una nota interna sale por correo a Marina Costa con lo que el equipo comentaba entre sí.

**El valor por defecto de `message_post` es nota** (`mail.mt_note`), que es el conservador. Si quieres que el mensaje llegue a los seguidores externos, hay que pedirlo con `mail.mt_comment` explícitamente.

Un módulo puede definir sus propios subtipos para dar control fino, por ejemplo para que alguien siga los cambios de estado de un pedido pero no la conversación:

```xml
<record id="mt_order_shipped" model="mail.message.subtype">
    <field name="name">Order Shipped</field>
    <field name="res_model">shop.order</field>
    <field name="default" eval="True"/>
    <field name="description">Order shipped</field>
</record>
```

Con `default` a verdadero, quien empiece a seguir el pedido queda suscrito a este subtipo sin tener que marcarlo. Desde Python se usa por su XML ID: `subtype_xmlid='shop.mt_order_shipped'`.

## Los seguidores

La lista de seguidores es lo que hace que publicar un mensaje no requiera saber a quién avisar.

### Añadir y quitar a mano

```python
# Suscribir al cliente y al comercial
self.message_subscribe(partner_ids=[self.partner_id.id, self.user_id.partner_id.id])

# Suscribir solo a ciertos subtipos
subtype = self.env.ref('shop.mt_order_shipped')
self.message_subscribe(partner_ids=[self.partner_id.id], subtype_ids=[subtype.id])

# Dejar de seguir
self.message_unsubscribe(partner_ids=[self.partner_id.id])
```

Los seguidores son **`res.partner`**, no `res.users`. Por eso al suscribir a un usuario se escribe `user.partner_id.id`: es el error de tipo más frecuente al usar esta API, y cuando se cuela no da error —el ID existe, pero apunta a otro contacto— así que el aviso acaba en el buzón de un desconocido.

### La suscripción automática

Odoo suscribe gente por su cuenta en dos situaciones, y conviene conocerlas porque explican seguidores que "aparecen solos":

- **Al asignar un responsable.** Si el modelo tiene un campo `user_id` que apunta a `res.users` y está marcado con `tracking=True`, al cambiarlo Odoo suscribe a la persona asignada **y le manda una notificación** de "te han asignado esto". No hay que programarlo: sale del *mixin*.
- **Por jerarquía de subtipos.** Un subtipo puede declarar un `parent_id` y un `relation_field`, y eso permite que quien sigue un registro padre reciba los mensajes de sus hijos. Es el mecanismo por el que seguir un proyecto entero hace llegar los avisos de sus tareas.

Y se puede intervenir: el método `_message_auto_subscribe_followers` está pensado para sobreescribirse cuando el criterio de "quién debería seguir esto" es propio del negocio.

```python
def _message_auto_subscribe_followers(self, updated_values, default_subtype_ids):
    """Also subscribe the customer when it is set on the order."""
    res = super()._message_auto_subscribe_followers(updated_values, default_subtype_ids)
    partner_id = updated_values.get('partner_id')
    if partner_id:
        res.append((partner_id, default_subtype_ids, False))
    return res
```

A partir de ahí, cada vez que se rellene el cliente de un pedido, Marina Costa queda como seguidora sin que nadie tenga que acordarse.

## Campana o correo: lo decide quien recibe, no tú

Cuando un mensaje se entrega a un seguidor, el canal sale del campo `notification_type` de su usuario:

- `inbox` — le aparece en la campana de Odoo, dentro de Discuss.
- `email` — le llega un correo.

Cada quien lo configura en sus preferencias, y quienes son usuarios de portal o externos tienen forzosamente `email` (no tienen acceso a la campana). Tu código **no puede** elegir el canal: `message_post` no tiene un parámetro para eso.

Esto es lo primero que hay que mirar ante un "no me llegan los correos de Odoo": muy a menudo no hay ningún problema de correo, simplemente esa persona tiene `inbox` y los avisos están esperando en su campana. Se comprueba en *Ajustes → Usuarios*, campo "Notification" (visible con el [modo desarrollador](../fundamentos/Modo-Desarrollador.md) activado).

Y un detalle que sorprende: **quien publica un mensaje no se lo autonotifica**. Si Jordi escribe en el pedido, Jordi no recibe copia, aunque sea seguidor. Se puede forzar con `notify_author=True`, pero rara vez es lo que se quiere.

## `message_notify`: avisar sin ensuciar el historial

A veces hay que avisar a alguien de algo relacionado con un registro, pero el aviso no forma parte de la historia del documento. Para eso está `message_notify`, que crea un mensaje de tipo `user_notification` **que no se muestra en el chatter**:

```python
def action_request_review(self):
    self.ensure_one()
    self.message_notify(
        partner_ids=self.user_id.partner_id.ids,
        subject=_("Review needed on %s", self.name),
        body=_("The customer changed the delivery address. Please review the order."),
    )
```

Jordi recibe el aviso por su canal (campana o correo), con un enlace al pedido, y el chatter de SO0042 sigue conteniendo solo lo que de verdad le pasó al pedido.

Las diferencias con `message_post`, que son la razón de que existan las dos:

| | `message_post` | `message_notify` |
|---|---|---|
| Aparece en el chatter | Sí | **No** |
| Destinatarios | Seguidores según subtipo (+ `partner_ids`) | **Solo** los `partner_ids` que indiques |
| Requiere que el modelo herede `mail.thread` | Sí | No (acepta `model` y `res_id`) |
| Uso típico | Historia del documento | Empujón personal |

Como `message_notify` no necesita que el modelo sea un hilo, sirve para avisar sobre registros que no tienen chatter:

```python
self.env['mail.thread'].message_notify(
    partner_ids=admin.partner_id.ids,
    model='ir.cron',
    res_id=cron.id,
    subject=_("Nightly job failed"),
    body=_("The order synchronisation cron raised an error."),
)
```

Igual que en `message_post`, en Odoo 18 los argumentos de `message_notify` son **de palabra clave**.

## Registro automático de cambios: `tracking`

Marcar un campo con `tracking=True` hace que cada cambio de valor se anote en el chatter, con el valor anterior y el nuevo:

```python
state = fields.Selection([...], tracking=True)
partner_id = fields.Many2one('res.partner', tracking=True)
```

Al pasar el pedido de `draft` a `confirmed` aparece en el chatter una línea del estilo "Status: Draft → Confirmed", sin que nadie llame a `message_post`. Es la forma más económica de auditar un modelo: dos palabras en la definición del campo.

`tracking` acepta un número para ordenar las líneas cuando cambian varios campos a la vez (`tracking=10`, `tracking=20`); los de número más bajo salen primero.

Los cambios rastreados se publican como nota, así que no salen al cliente. Y hay un gancho para ir más allá: `_track_subtype` permite decidir **con qué subtipo** se anota un cambio concreto, lo que convierte un cambio de campo en un aviso a los seguidores:

```python
def _track_subtype(self, initial_values):
    """Post the 'shipped' subtype when the order reaches that state."""
    self.ensure_one()
    if 'state' in initial_values and self.state == 'shipped':
        return self.env.ref('shop.mt_order_shipped')
    return super()._track_subtype(initial_values)
```

Ahora, cuando el pedido pasa a `shipped`, el mensaje automático lleva el subtipo `mt_order_shipped`, y quien esté suscrito a él —el cliente incluido— recibe el aviso. Sin escribir una sola llamada a `message_post`: basta con cambiar el campo.

## Mensajes con plantilla: `message_post_with_source`

Cuando el cuerpo del mensaje merece una plantilla —porque lleva formato, o porque quien lo mantiene no es quien programa— se usa `message_post_with_source`, que renderiza una vista QWeb o una plantilla de correo y publica el resultado:

```python
self.message_post_with_source(
    'shop.mail_order_shipped_body',      # XML ID de una ir.ui.view o mail.template
    render_values={'tracking_url': self._get_tracking_url()},
    subtype_xmlid='mail.mt_comment',
)
```

Si el primer argumento es una vista QWeb, se usa para el cuerpo y el resto de valores los pones tú; si es una `mail.template`, la plantilla aporta también asunto y destinatarios. Este método sustituye al antiguo `message_post_with_view` / `message_post_with_template` de versiones anteriores a la 17: si encuentras esos nombres en un módulo, están obsoletos.

Los detalles de las plantillas están en [Plantillas de correo](Plantillas-de-Correo.md).

## Rendimiento: apagar el chatter cuando estorba

El *mixin* tiene un coste: cada `create` publica un mensaje de creación, cada `write` calcula el rastreo de campos y cada mensaje resuelve la lista de destinatarios. En una importación de 50 000 pedidos, eso multiplica el tiempo por varios enteros y llena la base de datos de mensajes que nadie leerá.

Hay tres claves de contexto para desactivarlo, de menos a más agresivas:

```python
# 1. No registrar el mensaje "Shop Order created" al crear
self.env['shop.order'].with_context(mail_create_nolog=True).create(vals_list)

# 2. Además, no calcular el rastreo de campos
self.env['shop.order'].with_context(mail_notrack=True).create(vals_list)

# 3. Apagar TODO el mixin: sin mensajes, sin rastreo, sin suscripción automática
self.env['shop.order'].with_context(tracking_disable=True).create(vals_list)
```

Para una importación masiva, `tracking_disable=True` es la correcta. Y hay que usarla a sabiendas: al desactivarla se pierde el historial de esos registros, así que no es algo que deba estar puesto en el flujo normal de la aplicación "porque va más rápido".

Existe también `mail_auto_subscribe_no_notify`, que suscribe a la persona asignada pero **sin** mandarle el aviso de "te han asignado esto". Es lo que quieres cuando reasignas 300 pedidos en un traspaso de cartera y no quieres inundar el buzón de nadie.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `TypeError: message_post() takes 1 positional argument but 2 were given` | En Odoo 18 los argumentos son de palabra clave: `body=...` |
| El HTML del mensaje se ve como texto | Falta envolver el cuerpo en `Markup(...)` |
| El mensaje aparece en el chatter pero nadie lo recibe | El registro no tiene seguidores, o el subtipo no coincide con los suyos |
| El cliente no recibe el mensaje | Se publicó con `mail.mt_note` (interno). Usa `mail.mt_comment` |
| El cliente recibió una nota interna del equipo | Se publicó con `mail.mt_comment` cuando debía ser nota |
| El aviso llega a un contacto que no tiene nada que ver | Se pasó un ID de `res.users` donde iba un `res.partner`. Usa `user.partner_id.id` |
| Quien publica el mensaje no recibe copia | Es el comportamiento normal. Fuérzalo con `notify_author=True` si de verdad hace falta |
| "No me llegan los correos de Odoo" | Esa persona tiene `notification_type = inbox`: están en la campana |
| El chatter no aparece en la vista | Falta `<chatter/>` en el formulario, o el modelo no hereda `mail.thread` |
| `AttributeError: 'shop.order' object has no attribute 'message_post'` | Falta `mail` en `depends` del manifiesto, o `mail.thread` en `_inherit` |
| Una importación tarda horas | El *mixin* trabajando por cada fila. Usa `tracking_disable=True` |
| Al reasignar en lote se envían cientos de avisos | La suscripción automática notificando. Añade `mail_auto_subscribe_no_notify=True` |
| El mensaje del chatter desapareció tras un error | Un `raise` posterior deshizo la transacción. Ver [Errores y avisos](Errores-y-Avisos.md) |

## Ejemplo completo: un pedido que se comunica solo

Junta las piezas en el orden en que aparecen en un módulo real.

```python
from markupsafe import Markup
from odoo import _, api, fields, models

class ShopOrder(models.Model):
    _name = 'shop.order'
    _description = 'Shop Order'
    _inherit = ['mail.thread', 'mail.activity.mixin']

    name = fields.Char(string="Reference", required=True, copy=False, default="New")
    partner_id = fields.Many2one('res.partner', string="Customer", required=True, tracking=True)
    user_id = fields.Many2one('res.users', string="Salesperson", tracking=True,
                              default=lambda self: self.env.user)
    state = fields.Selection(
        [('draft', "Draft"), ('confirmed', "Confirmed"), ('shipped', "Shipped"), ('cancel', "Cancelled")],
        default='draft', required=True, tracking=True)
    carrier_tracking_ref = fields.Char(string="Tracking Reference")

    # 1. Quien sea el cliente del pedido sigue el pedido.
    def _message_auto_subscribe_followers(self, updated_values, default_subtype_ids):
        res = super()._message_auto_subscribe_followers(updated_values, default_subtype_ids)
        if updated_values.get('partner_id'):
            res.append((updated_values['partner_id'], default_subtype_ids, False))
        return res

    # 2. El paso a "shipped" se anota con un subtipo propio, así que avisa a los seguidores.
    def _track_subtype(self, initial_values):
        self.ensure_one()
        if 'state' in initial_values and self.state == 'shipped':
            return self.env.ref('shop.mt_order_shipped')
        return super()._track_subtype(initial_values)

    def action_ship(self):
        self.ensure_one()

        # 3. El cambio de campo ya genera el mensaje rastreado; no hace falta message_post para eso.
        self.state = 'shipped'

        # 4. Mensaje a la conversación: sale a los seguidores externos, cliente incluido.
        self.message_post(
            body=Markup(_("Your order is on its way. Tracking reference: <b>%s</b>")) % (
                self.carrier_tracking_ref or _("pending")),
            subject=_("Order %s shipped", self.name),
            subtype_xmlid='mail.mt_comment',
        )

        # 5. Nota interna: información del equipo que el cliente no debe ver.
        self.message_post(
            body=_("Shipped from the secondary warehouse; margin reduced by the express fee."),
            subtype_xmlid='mail.mt_note',
        )

        # 6. Aviso personal a quien tiene que revisar la factura, fuera del historial del pedido.
        invoicing_group = self.env.ref('shop.group_shop_invoicing')
        self.message_notify(
            partner_ids=invoicing_group.users.partner_id.ids,
            subject=_("Invoice pending for %s", self.name),
            body=_("Order %s has been shipped and is ready to invoice.", self.name),
        )
```

Cada bloque contesta a una pregunta distinta, y el conjunto ilustra la decisión de fondo de la ficha: **el rastreo de campos cubre el "qué cambió", el mensaje cubre el "qué le contamos al cliente", la nota cubre el "qué sabemos internamente" y la notificación cubre el "quién tiene que hacer algo ahora"**. Los cuatro son el mismo *mixin* y no son intercambiables.

## Buenas prácticas avanzadas

- **Antes de escribir un `message_post`, comprueba si `tracking=True` ya lo hace.** Publicar "Status changed to Confirmed" a mano en un campo que ya está rastreado deja el chatter con la información duplicada, y el mensaje escrito a mano no sabe el valor anterior. El rastreo es más barato de mantener (dos palabras en el campo, y sigue funcionando cuando el cambio venga de otro sitio) y produce mejor información. Reserva `message_post` para lo que no es un cambio de campo: comunicaciones, decisiones y contexto.
- **El subtipo es una decisión de confidencialidad, no de estilo.** `mail.mt_comment` sale por correo a los seguidores externos: un comentario interno publicado con ese subtipo llega al cliente y no hay forma de retirarlo. El hábito de quien lleva tiempo en esto es pasar `subtype_xmlid` **siempre explícito**, incluso cuando el valor por defecto sería el correcto, porque así el revisor del código ve la decisión en la línea en vez de tener que recordar cuál es el defecto.
- **Define subtipos propios en cuanto tu modelo tenga dos clases de aviso.** Sin subtipos propios, quien sigue un pedido recibe todo o nada, y lo que hace la gente entonces es dejar de seguir los pedidos, con lo que el mecanismo entero deja de servir. Con un subtipo por evento relevante (`mt_order_shipped`, `mt_order_cancelled`), cada persona ajusta lo que recibe desde la lista de seguidores sin tocar código.
- **Los seguidores son `res.partner`; escribe `.partner_id.id` como reflejo.** El error de pasar un ID de `res.users` no falla: crea un seguidor válido que apunta a un contacto arbitrario, así que el fallo aparece semanas después como "un proveedor está recibiendo nuestros pedidos internos". Cuando revises código ajeno, `partner_ids=[user.id]` es una señal de alarma inmediata.
- **En procesos masivos, decide explícitamente qué historial quieres perder.** `tracking_disable=True` multiplica la velocidad de una importación, y a cambio esos registros no tendrán historia. Es la decisión correcta para una carga inicial y una pérdida grave si se cuela en el flujo diario, porque entonces los cambios de la aplicación dejan de auditarse sin que nadie lo note. Ponla en el método de importación, nunca en un `create` genérico del modelo.
- **Ojo con `message_post` dentro de un `create` o un `write` sobreescrito.** Es la vía rápida a una recursión: el `message_post` escribe en el registro (actualiza `message_ids`), lo que vuelve a entrar en tu `write`. Si necesitas publicar al crear, hazlo después de llamar a `super()` y sobre los registros ya creados, y valora si el mensaje que quieres no es en realidad un rastreo de campo o el propio mensaje de creación del *mixin*.

## Documentación oficial

- [Odoo Developer Documentation — Mixins y modelos útiles](https://www.odoo.com/documentation/18.0/developer/reference/backend/mixins.html) — la referencia de `mail.thread`: qué campos añade, qué métodos expone y cómo se activa en un modelo. Es el punto de partida obligatorio.
- [Código de `mail_thread.py`](https://github.com/odoo/odoo/blob/18.0/addons/mail/models/mail_thread.py) — el *mixin* completo. Las cabeceras de `message_post`, `message_notify` y `_message_auto_subscribe_followers` documentan cada parámetro con más detalle que la documentación publicada; es el sitio donde comprobar una firma antes de usarla.
- [Código de `mail_message_subtype.py`](https://github.com/odoo/odoo/blob/18.0/addons/mail/models/mail_message_subtype.py) — el modelo de subtipos, con el significado de `internal`, `parent_id`, `relation_field` y `default`. Cien líneas que explican el mecanismo de suscripción por jerarquía mejor que cualquier resumen.

## Recursos didácticos

- [Odoo Tutorials — Discuss (chatter y actividades)](https://www.odoo.com/documentation/18.0/developer/tutorials/discover_js_framework.html) — los tutoriales oficiales incluyen el paso de añadir chatter y actividades a un módulo propio, con el código completo. Útil para hacerlo una vez de principio a fin antes de improvisar.
- [Runbot de Odoo](https://runbot.odoo.com/) — una instancia desechable donde crear un pedido, cambiarle el responsable y ver aparecer la suscripción automática y el mensaje "te han asignado". El mecanismo se entiende en dos minutos viéndolo y en media hora leyéndolo.

---

*En resumen: el chatter no es una caja de comentarios sino un sistema de distribución —mensaje, subtipo y seguidores— y dominarlo consiste en elegir bien el subtipo, dejar que el rastreo de campos haga el trabajo automático y no confundir la historia del documento con un empujón personal.*
