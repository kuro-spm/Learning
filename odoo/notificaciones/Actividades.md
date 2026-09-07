# Actividades (`mail.activity`)

## ¿Qué es?

Una actividad es **una tarea con responsable y fecha límite pegada a un registro**: "llamar al cliente antes del jueves" colgando del pedido SO0042. Se ve en el reloj de la ficha, en la campana de actividades de la barra superior y en las vistas de actividad de cualquier listado. Por debajo es el modelo `mail.activity` y el *mixin* `mail.activity.mixin`, y se crea con una línea: `activity_schedule(...)`.

## ¿Por qué existe?

Porque hay una diferencia enorme entre "enterarse" y "tener que hacer algo", y ningún mecanismo de aviso la recoge. Un mensaje en la campana se lee, se cierra y se olvida; un correo baja en la bandeja de entrada hasta desaparecer. Ninguno de los dos vuelve a insistir, ninguno tiene fecha, y sobre todo: **no hay forma de listar lo que queda pendiente**.

La actividad convierte el aviso en un objeto consultable. Tiene responsable, tipo, resumen y `date_deadline`, así que se puede filtrar por vencidas, agrupar por persona y ver en un calendario. Y no desaparece: se queda ahí, marcando el registro, hasta que alguien la cierra explícitamente.

> Si has usado el campo "assignee + due date" de una incidencia de Jira o de GitHub, es exactamente eso, con la diferencia de que aquí se puede colgar de **cualquier** modelo —un pedido, un contacto, una factura— sin crear un registro de tareas aparte.

## ¿Cuándo y para qué se usa?

Cuando el aviso implica trabajo por parte de una persona concreta:

- Llamar al cliente para confirmar la dirección de entrega.
- Revisar un pedido que ha superado el límite de crédito.
- Subir el justificante de pago que falta.
- Hacer seguimiento a los siete días de enviar un presupuesto.
- Comprobar por qué un pedido lleva cinco días sin salir del almacén.

El criterio para decidir entre una actividad y un [mensaje en el chatter](Chatter-y-Seguidores.md) es directo: **¿alguien tiene que hacer algo, y quiero poder comprobar si lo hizo?** Si la respuesta es sí, es una actividad. Si solo se trata de informar, es un mensaje.

Seguimos con el modelo `shop.order` de la tienda online, el pedido **SO0042** del cliente Marina Costa y el comercial Jordi Vidal.

---

## Activar las actividades en un modelo propio

Como el chatter, es un *mixin*. Casi siempre se añaden los dos a la vez:

```python
class ShopOrder(models.Model):
    _name = 'shop.order'
    _description = 'Shop Order'
    _inherit = ['mail.thread', 'mail.activity.mixin']
```

Con el `_inherit`, el modelo gana estos campos —todos calculados, así que no ocupan espacio, pero sí son filtrables y agrupables—:

| Campo | Qué contiene |
|---|---|
| `activity_ids` | Las actividades abiertas del registro |
| `activity_state` | `overdue`, `today` o `planned`, según la más urgente |
| `activity_user_id` | El responsable de la siguiente actividad |
| `activity_type_id` | El tipo de la siguiente actividad |
| `activity_date_deadline` | La fecha límite más próxima |
| `my_activity_date_deadline` | Igual, pero solo de **mis** actividades |
| `activity_summary` | El resumen de la siguiente actividad |
| `activity_exception_decoration` | Marca visual de actividad excepcional |

`activity_state` es el que hace útil todo el mecanismo, porque permite filtros como "pedidos con actividades vencidas" sin escribir nada:

```xml
<filter name="activities_overdue" string="Late Activities"
        domain="[('activity_state', '=', 'overdue')]"/>
<filter name="activities_my" string="My Activities"
        domain="[('activity_user_id', '=', uid)]"/>
```

Para que aparezca el reloj en el formulario, basta con tener el [chatter](Chatter-y-Seguidores.md) declarado: la etiqueta `<chatter/>` incluye la zona de actividades. Y para tener la vista de actividades en un listado, se añade a las vistas de la acción:

```xml
<field name="view_mode">list,form,activity</field>
```

La vista `activity` es una matriz de registros por tipo de actividad, muy útil para ver de un golpe qué hay pendiente en toda la cartera.

## Los tipos de actividad

Un tipo (`mail.activity.type`) define la clase de tarea y sus valores por defecto. Odoo trae unos cuantos en el módulo `mail`:

| Tipo | XML ID |
|---|---|
| Email | `mail.mail_activity_data_email` |
| Call | `mail.mail_activity_data_call` |
| Meeting | `mail.mail_activity_data_meeting` |
| To-Do | `mail.mail_activity_data_todo` |
| Upload Document | `mail.mail_activity_data_upload_document` |
| Warning | `mail.mail_activity_data_warning` |

Los campos interesantes de un tipo:

| Campo | Qué hace |
|---|---|
| `delay_count` + `delay_unit` + `delay_from` | El plazo por defecto: "5 días desde hoy" |
| `summary` | Resumen por defecto |
| `default_note` | Nota por defecto, en HTML |
| `default_user_id` | Responsable por defecto |
| `icon` | Icono de Font Awesome (`fa-phone`) |
| `res_model` | Si se rellena, el tipo solo aparece en ese modelo |
| `category` | `default`, `upload_file` (o `phonecall` con el módulo de telefonía) |
| `chaining_type` + `triggered_next_type_id` | Qué actividad se crea automáticamente al cerrar esta |
| `keep_done` | Conservar la actividad visible después de hecha |
| `mail_template_ids` | Plantillas ofrecidas al cerrarla |

Definir un tipo propio para el módulo, con un plazo por defecto de tres días:

```xml
<record id="mail_activity_shop_delivery_check" model="mail.activity.type">
    <field name="name">Delivery Check</field>
    <field name="summary">Check the delivery went through</field>
    <field name="icon">fa-truck</field>
    <field name="res_model">shop.order</field>
    <field name="delay_count">3</field>
    <field name="delay_unit">days</field>
    <field name="delay_from">current_date</field>
    <field name="sequence">15</field>
</record>
```

Con `res_model` a `shop.order`, este tipo solo se ofrece en los pedidos, así que no ensucia el desplegable de actividades de los contactos ni de las facturas.

## Crear una actividad: `activity_schedule`

El método del *mixin*. Su primer argumento es el **XML ID del tipo**, no un ID numérico, lo que evita tener un `env.ref` en cada llamada:

```python
from dateutil.relativedelta import relativedelta
from odoo import _, fields, models

class ShopOrder(models.Model):
    _inherit = 'shop.order'

    def action_confirm(self):
        self.ensure_one()
        self.state = 'confirmed'
        self.activity_schedule(
            'shop.mail_activity_shop_delivery_check',
            date_deadline=fields.Date.context_today(self) + relativedelta(days=3),
            summary=_("Check delivery of %s", self.name),
            note=_("Confirm with the carrier that the parcel left the warehouse."),
            user_id=self.user_id.id,
        )
```

Tras confirmar SO0042, a Jordi le aparece una actividad con fecha para dentro de tres días, y **recibe un aviso de "te han asignado esto"** por su canal habitual (campana o correo): la creación de la actividad dispara internamente un `message_notify` en el idioma del destinatario. No hay que programar ese aviso.

Los parámetros:

| Parámetro | Qué hace |
|---|---|
| `act_type_xmlid` | El XML ID del tipo. Si se omite, se usa el tipo por defecto del modelo |
| `date_deadline` | La fecha límite. Si se omite, **hoy** |
| `summary` | El título que se ve en el reloj |
| `note` | El cuerpo, en HTML |
| `user_id` | El responsable. Si se omite, **quien ejecuta el código** |
| `**act_values` | Cualquier otro campo de `mail.activity` |

Dos defectos que hay que conocer porque muerden:

- **Sin `date_deadline`, la fecha es hoy**, así que la actividad nace vencida al día siguiente. Pon siempre una fecha explícita, aunque sea hoy a propósito.
- **Sin `user_id`, el responsable es quien ejecuta.** Esto es un problema en un cron: el responsable acaba siendo el usuario del cron (a menudo el administrador), no la persona que debía actuar. En cualquier código que corra desatendido, `user_id` es obligatorio de facto.

Y una tercera cosa que se nota al leer el código del *mixin*: `activity_schedule` **funciona sobre un conjunto de registros**, creando una actividad por cada uno. No hace falta bucle:

```python
# Una actividad de revisión en cada uno de los pedidos bloqueados
blocked_orders.activity_schedule(
    'mail.mail_activity_data_todo',
    date_deadline=fields.Date.context_today(self),
    summary=_("Unblock this order"),
    user_id=self.env.ref('shop.user_logistics_manager').id,
)
```

### Fechas: `fields.Date.context_today`, no `date.today()`

El plazo de una actividad es una fecha, y la fecha "de hoy" depende de la zona horaria de quien mira:

```python
# ✅ La fecha de hoy en la zona horaria del usuario
date_deadline = fields.Date.context_today(self) + relativedelta(days=3)

# ❌ La fecha de hoy en UTC: a las 00:30 en España es todavía ayer
date_deadline = fields.Date.today() + relativedelta(days=3)
```

El error solo se ve en las horas límite del día, lo que lo convierte en uno de esos fallos que "no se reproducen" y que en realidad ocurren cada noche.

## Cerrar, reprogramar y buscar actividades

El *mixin* trae métodos para gestionar actividades **por tipo**, que son los que se usan cuando el propio sistema debe cerrar lo que él mismo abrió.

```python
# Marcar como hecha la actividad de comprobación cuando el pedido se entrega
def action_deliver(self):
    self.ensure_one()
    self.state = 'shipped'
    self.activity_feedback(
        ['shop.mail_activity_shop_delivery_check'],
        feedback=_("Delivery confirmed automatically by the carrier webhook."),
    )
```

`activity_feedback` cierra la actividad **y deja el texto del `feedback` como mensaje en el chatter**, así que queda constancia de cómo se resolvió. Es la diferencia con borrarla.

Los tres métodos y cuándo usar cada uno:

| Método | Qué hace | Cuándo |
|---|---|---|
| `activity_feedback(xmlids, feedback=...)` | La marca como hecha y registra el comentario | La tarea **se hizo** (aunque la haya hecho el sistema) |
| `activity_unlink(xmlids)` | La borra sin dejar rastro | La tarea **ya no procede**: el pedido se canceló |
| `activity_reschedule(xmlids, date_deadline=..., new_user_id=...)` | Cambia fecha o responsable | El plazo o la persona cambian |

La distinción entre `feedback` y `unlink` no es cosmética. Si un pedido se cancela, la actividad "llamar al cliente" no se hizo: borrarla es honesto. Si el transportista confirma la entrega, la comprobación sí se resolvió: cerrarla con `feedback` deja la explicación en el historial, y quien mire el pedido dentro de un mes sabrá por qué esa tarea no la hizo nadie a mano.

Los tres métodos aceptan además `user_id` para limitar la acción a las actividades de una persona:

```python
# Cerrar solo las mías, no las de mis compañeros
self.activity_feedback(['mail.mail_activity_data_todo'], user_id=self.env.uid)
```

Y para consultar, `activity_search`:

```python
pending = order.activity_search(['shop.mail_activity_shop_delivery_check'])
if pending:
    ...
```

Es lo que evita duplicar actividades: comprobar si ya existe una abierta del mismo tipo antes de crear otra. Sin esa comprobación, un cron que se ejecute cada hora deja treinta actividades idénticas en el mismo pedido en un día.

## Actividades en cadena

Un tipo de actividad puede disparar el siguiente automáticamente al cerrarse. Se configura en el propio tipo, sin código:

```xml
<record id="mail_activity_shop_delivery_check" model="mail.activity.type">
    <field name="name">Delivery Check</field>
    <field name="chaining_type">trigger</field>
    <field name="triggered_next_type_id" ref="shop.mail_activity_shop_satisfaction_call"/>
</record>
```

Al marcar como hecha la comprobación de entrega, Odoo crea sola la llamada de satisfacción, con el plazo por defecto de ese segundo tipo. `chaining_type` admite dos valores: `suggest` (se ofrece, y la persona decide) y `trigger` (se crea sin preguntar).

Es un mecanismo potente para modelar procesos —cada paso abre el siguiente— y conviene usarlo con moderación: una cadena de cinco pasos que nadie puede parar genera trabajo que nadie pidió.

## Cómo llega la actividad a su responsable

Tres vías, y ninguna hay que programarla:

1. **El aviso de asignación.** Al crearse (o al cambiar de responsable), la actividad manda un `message_notify` a la persona asignada, renderizado en su idioma, con el resumen, el tipo y la fecha límite. Llega a su campana o a su correo según su `notification_type`.
2. **La campana de actividades de la barra superior.** El icono de reloj lista las actividades pendientes agrupadas por modelo, con contador de vencidas. Se actualiza en tiempo real: crear una actividad envía un mensaje por el bus (`bus.bus`) al responsable, así que el contador cambia sin recargar la página.
3. **Las vistas y filtros.** Con `activity_state` y `activity_user_id` en el modelo, cualquier listado puede filtrar "mis actividades vencidas". Es la vía que de verdad se usa para trabajar: el aviso avisa una vez, el filtro está siempre.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `AttributeError: 'shop.order' object has no attribute 'activity_schedule'` | Falta `mail.activity.mixin` en `_inherit` (o `mail` en `depends`) |
| La actividad nace vencida | No se pasó `date_deadline`: el valor por defecto es hoy |
| El responsable de todas las actividades es el administrador | Falta `user_id` en un `activity_schedule` que corre en un cron |
| Se acumulan actividades duplicadas | Falta comprobar con `activity_search` antes de crear |
| El reloj no aparece en el formulario | Falta `<chatter/>` en la vista |
| El tipo de actividad no sale en el desplegable | Su `res_model` apunta a otro modelo |
| `ValueError: External ID not found` en `activity_schedule` | El primer argumento es un XML ID (`'shop.mail_activity_...'`), no un ID numérico |
| Al cancelar un pedido quedan actividades huérfanas abiertas | Falta `activity_unlink` en el método de cancelación |
| El plazo se desplaza un día en la franja nocturna | Se usó `fields.Date.today()` (UTC) en vez de `context_today(self)` |
| La actividad se cerró y no hay explicación de cómo | Se usó `activity_unlink` donde correspondía `activity_feedback` con texto |
| Nadie recibió el aviso de asignación | El responsable no tiene usuario activo, o el `user_id` apuntaba a la persona equivocada |

## Ejemplo completo: el ciclo de vida de una actividad

Un pedido que abre su tarea de seguimiento al confirmarse, la cierra si el transportista confirma la entrega y la retira si el pedido se cancela.

```python
from dateutil.relativedelta import relativedelta
from odoo import _, fields, models

class ShopOrder(models.Model):
    _inherit = 'shop.order'

    ACTIVITY_DELIVERY = 'shop.mail_activity_shop_delivery_check'

    def action_confirm(self):
        for order in self:
            order.state = 'confirmed'

            # 1. No duplicar: si ya hay una comprobación abierta, no se crea otra.
            if order.activity_search([self.ACTIVITY_DELIVERY]):
                continue

            # 2. Responsable y fecha explícitos: esto puede ejecutarse desde un cron.
            order.activity_schedule(
                self.ACTIVITY_DELIVERY,
                date_deadline=fields.Date.context_today(order) + relativedelta(days=3),
                summary=_("Check delivery of %s", order.name),
                note=_("Confirm with the carrier that the parcel left the warehouse."),
                user_id=order.user_id.id or self.env.uid,
            )

    def action_deliver(self):
        """Called by the carrier webhook: the task did get done."""
        for order in self:
            order.state = 'shipped'
            order.activity_feedback(
                [self.ACTIVITY_DELIVERY],
                feedback=_("Delivery confirmed by the carrier on %s.",
                           fields.Date.context_today(order)),
            )

    def action_cancel(self):
        """The task no longer applies: remove it instead of closing it."""
        for order in self:
            order.state = 'cancel'
            order.activity_unlink([self.ACTIVITY_DELIVERY])
            order.message_post(
                body=_("Order cancelled; the pending delivery check was removed."),
                subtype_xmlid='mail.mt_note',
            )
```

Las tres decisiones que hacen que esto funcione en producción:

- **El XML ID vive en una constante.** Aparece en tres métodos; escribirlo tres veces es garantía de que algún día uno de ellos se queda desincronizado y deja de encontrar las actividades que él mismo creó.
- **La comprobación de duplicados va antes de crear.** Confirmar dos veces un pedido es algo que pasa, y sin ese `activity_search` el reloj acaba con dos tareas idénticas.
- **La cancelación deja constancia de que se retiró la actividad.** `activity_unlink` borra sin rastro, así que el `message_post` es lo que evita la pregunta "¿alguien comprobó esta entrega?" cuando la respuesta correcta es "no había nada que comprobar".

## Buenas prácticas avanzadas

- **Todo `activity_schedule` que corra desatendido lleva `user_id` explícito.** Es el fallo silencioso más común del mecanismo: en un cron, el responsable por omisión es el usuario que ejecuta la tarea programada, normalmente el administrador. El resultado es un administrador con 400 actividades vencidas que nadie mira y un equipo que no se enteró de nada. Y como el código funciona perfectamente cuando se prueba a mano desde la interfaz, el error no aparece hasta que está en producción.
- **Cierra con `activity_feedback` lo que se hizo y borra con `activity_unlink` lo que dejó de proceder.** No es una preferencia estética: el histórico de actividades cerradas es el rastro de que un proceso se siguió, y borrar en lugar de cerrar destruye esa evidencia. La regla se puede aplicar mecánicamente preguntando "¿esta tarea se hizo?": si sí, `feedback` con una frase que explique cómo; si no, `unlink`.
- **Comprueba con `activity_search` antes de crear, siempre que el código pueda repetirse.** Cualquier método llamado desde un botón, un cron o un *webhook* puede ejecutarse dos veces —doble clic, reintento de la pasarela, cron solapado—. Las actividades no tienen restricción de unicidad, así que se acumulan sin protestar. Cinco líneas de comprobación evitan un reloj con veinte tareas idénticas que la gente aprende a ignorar en bloque.
- **Un tipo de actividad propio por proceso, con su `res_model` y su plazo.** Reutilizar "To-Do" para todo funciona el primer mes y luego impide lo único que hace valiosas las actividades: filtrar y medir. Con tipos propios se puede responder a "¿cuántas comprobaciones de entrega llevamos vencidas?", que es la pregunta por la que existe el mecanismo. Y `res_model` mantiene los desplegables limpios en el resto de modelos.
- **Añade `activity_state` a la vista de búsqueda del modelo, no solo al formulario.** El aviso de asignación llega una vez y se pierde entre lo demás; el filtro "Late Activities" está disponible todos los días. Los equipos que trabajan bien con actividades no lo hacen porque reciban los avisos, sino porque tienen un listado filtrado por vencimiento como pantalla de inicio.
- **No conviertas las actividades en un gestor de proyectos.** Una actividad es un recordatorio pegado a un documento: sin subtareas, sin dependencias, sin estimaciones. Cuando el trabajo empieza a necesitar eso, el mecanismo correcto es un modelo de tareas propio, y forzar cadenas largas de actividades encadenadas solo produce procesos rígidos que nadie puede detener a mitad.

## Documentación oficial

- [Odoo Developer Documentation — Mixins y modelos útiles](https://www.odoo.com/documentation/18.0/developer/reference/backend/mixins.html) — la sección de `mail.activity.mixin` con los campos que aporta y la firma de `activity_schedule`. Es el punto de partida.
- [Código de `mail_activity_mixin.py`](https://github.com/odoo/odoo/blob/18.0/addons/mail/models/mail_activity_mixin.py) — el *mixin* completo: aquí se ve que `activity_schedule` opera sobre conjuntos, cuáles son exactamente los valores por defecto y cómo funcionan `activity_feedback`, `activity_unlink` y `activity_reschedule`.
- [Código de `mail_activity_type.py`](https://github.com/odoo/odoo/blob/18.0/addons/mail/models/mail_activity_type.py) — la definición de los tipos, con el significado de `delay_from`, `chaining_type` y `keep_done`. Necesario para configurar plazos y cadenas sin ir a tientas.
- [Odoo Documentation — Activities](https://www.odoo.com/documentation/18.0/applications/essentials/activities.html) — la vista de usuario: cómo se planifican y cierran desde la interfaz, y cómo se configuran los tipos sin programar. Útil para saber qué se puede delegar a quien administra el sistema.

## Recursos didácticos

- [Runbot de Odoo](https://runbot.odoo.com/) — una instancia desechable donde planificar una actividad en un contacto, cambiarle el responsable y ver llegar el aviso de asignación y actualizarse el contador del reloj en tiempo real. El mecanismo se entiende de golpe al verlo.
- [Font Awesome 4 (iconos)](https://fontawesome.com/v4/icons/) — el catálogo de iconos que acepta el campo `icon` de un tipo de actividad (`fa-truck`, `fa-phone`, `fa-file-text-o`). Odoo usa esta versión, así que buscar en la última no sirve.

---

*En resumen: una actividad es un aviso con responsable, fecha y capacidad de ser listado —y eso, no la notificación, es lo que hace que el trabajo pendiente se vea y no se olvide.*
