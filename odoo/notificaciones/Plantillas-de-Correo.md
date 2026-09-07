# Plantillas de correo (`mail.template`)

## ¿Qué es?

Una `mail.template` es un registro de la base de datos que guarda **un correo con huecos**: el asunto, el cuerpo HTML, quién lo recibe y qué informes se adjuntan, todo con expresiones que se rellenan con los datos de un registro concreto en el momento de enviarlo. Se define en XML dentro de un módulo, se edita después desde la interfaz sin tocar código, y se dispara desde Python con una línea.

## ¿Por qué existe?

Porque el texto de un correo que ve un cliente cambia mucho más a menudo que el código que lo envía. La persona de marketing quiere otra frase de cierre; el departamento legal, un pie distinto; ventas quiere añadir el enlace de seguimiento. Si ese texto vive dentro de un `.py`, cada cambio es un despliegue.

La segunda razón es la traducción. Marina Costa lee catalán y el proveedor de Hamburgo lee alemán, y el correo tiene que salir en el idioma **de quien lo recibe**, no en el de quien pulsa el botón. Una plantilla sabe resolver eso; una cadena en el código, no.

Y la tercera es que un correo es más que texto: destinatarios, remitente, servidor de salida, adjuntos, dirección de respuesta, programación. Reunir todo eso en un registro editable evita repetirlo en cada punto del código que manda correo.

> Si has usado Handlebars, Jinja o Razor para generar correos, es lo mismo con dos particularidades: la plantilla se guarda en la base de datos (no en un fichero) y el motor es el de Odoo, QWeb.

## ¿Cuándo y para qué se usa?

Siempre que el destinatario esté **fuera** de Odoo, o cuando el texto deba poder editarse sin tocar código:

- Confirmaciones al cliente: pedido recibido, pedido enviado, factura adjunta.
- Recordatorios de vencimiento.
- Avisos a proveedores.
- Invitaciones y altas de usuarios.
- Informes periódicos por correo.

Cuando el destinatario es un usuario interno y el aviso debe quedar en el historial del registro, el mecanismo correcto no es una plantilla suelta sino el [chatter](Chatter-y-Seguidores.md) —que, por dentro, también acaba enviando correos, pero además deja constancia y respeta las preferencias de cada persona—.

Seguimos con el modelo `shop.order` de la tienda online, el pedido **SO0042** y el cliente Marina Costa.

---

## Anatomía de una plantilla

Los campos que se usan de verdad:

| Campo | Qué guarda |
|---|---|
| `name` | El nombre con el que aparece en las listas. No sale en el correo |
| `model_id` | El modelo al que se aplica: `shop.order` |
| `subject` | El asunto, con expresiones `{{ }}` |
| `email_from` | El remitente. Si se deja vacío, el de la compañía |
| `partner_to` | Destinatarios como **IDs de `res.partner`**, separados por comas |
| `email_to` | Destinatarios como direcciones de correo sueltas |
| `email_cc` | Copia |
| `reply_to` | Dónde van las respuestas |
| `body_html` | El cuerpo, en HTML con QWeb |
| `lang` | Expresión que decide en qué idioma se renderiza |
| `attachment_ids` | Adjuntos fijos, iguales en todos los envíos |
| `report_template_ids` | Informes que se generan y se adjuntan en cada envío (el PDF del pedido) |
| `email_layout_xmlid` | La plantilla envolvente (cabecera, pie, botón) |
| `mail_server_id` | Servidor de salida concreto, si no vale el de por defecto |
| `scheduled_date` | Expresión de fecha: no enviar antes de |
| `auto_delete` | Si se borra el rastro del correo tras enviarlo |

`partner_to` y `email_to` conviven, y la diferencia importa: `partner_to` apunta a contactos de Odoo, así que el correo se puede vincular al *partner*, respeta su idioma y su dirección actual; `email_to` es una dirección en crudo, útil para buzones que no son contactos (`facturacion@empresa.com`). Cuando el destinatario existe como contacto, `partner_to` es siempre mejor.

## Definir una plantilla en XML

Vive en un fichero de datos del módulo —por ejemplo `data/mail_template_data.xml`— declarado en el `__manifest__.py`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data noupdate="1">
        <record id="mail_template_order_shipped" model="mail.template">
            <field name="name">Shop: Order Shipped</field>
            <field name="model_id" ref="shop.model_shop_order"/>
            <field name="subject">Your order {{ object.name }} has been shipped</field>
            <field name="email_from">{{ (object.user_id.email_formatted or object.company_id.email_formatted) }}</field>
            <field name="partner_to">{{ object.partner_id.id }}</field>
            <field name="lang">{{ object.partner_id.lang }}</field>
            <field name="description">Sent to the customer when the order leaves the warehouse</field>
            <field name="body_html" type="html">
<div style="margin: 0px; padding: 0px;">
    <p>
        Hello <t t-out="object.partner_id.name or ''">Marina Costa</t>,
        <br/><br/>
        Your order <t t-out="object.name or ''">SO0042</t> has been shipped.
        <t t-if="object.carrier_tracking_ref">
            You can follow it with the reference
            <span style="font-weight: bold;" t-out="object.carrier_tracking_ref or ''">TRK-9F3E</span>.
        </t>
        <br/><br/>
        Thank you for your purchase.
    </p>
</div>
            </field>
        </record>
    </data>
</odoo>
```

Tres cosas de este XML que hay que entender bien:

- **`noupdate="1"`** significa "no sobrescribir al actualizar el módulo". Es lo correcto en una plantilla: si alguien ha adaptado el texto desde la interfaz, un `-u shop` no debe devolverlo a la versión original. La contrapartida es que tus mejoras del texto tampoco llegan a las bases de datos ya instaladas: hay que aplicarlas a mano o forzar la actualización.
- **`ref="shop.model_shop_order"`** es la forma de apuntar al modelo. Odoo crea un registro de `ir.model` por cada modelo, con el XML ID `<módulo>.model_<nombre_con_guiones_bajos>`: para `shop.order`, `shop.model_shop_order`.
- **`type="html"`** en `body_html` evita tener que escapar cada `<` del cuerpo.

## El renderizado: dos motores en la misma plantilla

Este es el punto donde más gente se atasca, porque **el asunto y el cuerpo no usan la misma sintaxis**.

**En los campos de una línea** (`subject`, `email_from`, `partner_to`, `email_to`, `lang`, `scheduled_date`) el motor es el de plantillas *inline*, y las expresiones van entre dobles llaves:

```xml
<field name="subject">Your order {{ object.name }} has been shipped</field>
```

**En el cuerpo** (`body_html`) el motor es QWeb, el mismo de las vistas e informes de Odoo, y las expresiones son atributos de etiqueta:

```xml
<t t-out="object.partner_id.name or ''">Marina Costa</t>
<t t-if="object.carrier_tracking_ref">…</t>
<t t-set="doc_name" t-value="'order' if object.state == 'confirmed' else 'draft'"/>
```

Escribir `{{ object.name }}` en el cuerpo **no funciona**: sale literalmente en el correo. Y escribir `t-out` en el asunto tampoco: sale la etiqueta. Es el error número uno con plantillas, y ahora se identifica de un vistazo.

Dos detalles del QWeb del cuerpo que se ven en todas las plantillas de Odoo y conviene copiar:

- **El `or ''` de `t-out="object.name or ''"`** evita que un valor falso se imprima como `False`.
- **El texto dentro de la etiqueta** (`>Marina Costa<`) es un *placeholder* de previsualización: se sustituye al renderizar, y solo se ve en el editor de plantillas. Poner ahí un valor realista hace la plantilla mucho más fácil de editar para quien no programa.

### Las variables disponibles

Dentro de una plantilla hay unas pocas variables, y son siempre las mismas:

| Variable | Qué es |
|---|---|
| `object` | El registro sobre el que se renderiza: el pedido SO0042 |
| `user` | El usuario que está ejecutando el envío |
| `ctx` | El contexto que se haya pasado al renderizar |
| `format_amount` | Ayudante para formatear importes con su moneda |
| `format_date`, `format_datetime`, `format_time` | Ayudantes de fecha en el idioma y zona horaria correctos |

```xml
<p>
    Total: <t t-out="format_amount(object.amount_total, object.currency_id) or ''">€ 149.90</t>
    <br/>
    Shipped on <t t-out="format_date(object.shipping_date) or ''">05/09/2026</t>
</p>
```

Sin `format_amount` saldría `149.9`, sin símbolo ni separador de miles y sin respetar la configuración regional de quien lo recibe. Usar los ayudantes no es cosmética: es lo que hace que un correo se lea igual de bien en `es_ES` y en `de_DE`.

## Enviar la plantilla desde Python

Dos métodos, según cuántos registros:

```python
# Un registro
template = self.env.ref('shop.mail_template_order_shipped')
template.send_mail(self.id)                     # devuelve el ID del mail.mail creado

# Varios registros: un correo por cada uno, en una sola pasada
template.send_mail_batch(self.ids)              # devuelve el conjunto de mail.mail
```

`send_mail_batch` es la versión por lotes que se introdujo en Odoo 18 y es la que hay que usar en un bucle sobre muchos registros: renderiza y crea los correos por bloques, en vez de dar una vuelta completa por cada uno.

Los parámetros que importan:

| Parámetro | Qué hace |
|---|---|
| `force_send` | `True` envía ya; `False` (por defecto) deja el correo en cola |
| `email_values` | Diccionario para sobrescribir cualquier valor del correo generado |
| `email_layout_xmlid` | La plantilla envolvente, si quieres otra que la de la plantilla |
| `raise_exception` | Si `True`, un fallo de envío lanza una excepción en vez de marcar el correo como fallido |

`email_values` es la vía de escape para lo que la plantilla no puede saber:

```python
template.send_mail(self.id, email_values={
    'email_cc': 'logistics@example.com',
    'attachment_ids': [attachment.id],
    'auto_delete': False,
})
```

## La cola de correo: `mail.mail` y su cron

Enviar un correo no es escribirlo: es entregarlo a un servidor SMTP, una operación lenta que puede fallar. Odoo la desacopla en dos pasos.

`send_mail` **no envía**: crea un registro `mail.mail` en estado `outgoing`. Quien envía de verdad es la tarea programada *Mail: Email Queue Manager*, que recorre la cola y va marcando cada correo.

Los estados de un `mail.mail`:

| Estado | Significado |
|---|---|
| `outgoing` | En cola, pendiente de enviar |
| `sent` | Entregado al servidor SMTP |
| `exception` | El envío falló. `failure_type` y `failure_reason` dicen por qué |
| `cancel` | Cancelado a mano |
| `received` | Entrante (para correo de llegada) |

Que esté `sent` significa "el servidor SMTP lo aceptó", **no** "llegó a la bandeja de Marina". Un rechazo posterior del servidor de destino se ve como devolución, no como error de Odoo.

### `force_send`: cuándo saltarse la cola

```python
# Recomendado: en cola, sale con el siguiente ciclo del cron
template.send_mail(self.id, force_send=False)

# Inmediato: intenta el SMTP dentro de esta petición
template.send_mail(self.id, force_send=True)
```

`force_send=True` tiene dos costes. El primero es el tiempo: quien pulsó el botón espera a que el SMTP responda, y si el servidor tarda cinco segundos, la interfaz se queda congelada cinco segundos —multiplicado por el número de correos—. El segundo es más sutil y es la razón de fondo para no usarlo: **un correo ya entregado al SMTP no se puede deshacer**. Si más adelante en el mismo método salta una excepción, la transacción se deshace, el pedido se queda como estaba... y el cliente ya tiene en su bandeja el correo diciendo que se le ha enviado.

Con la cola, el correo es una fila en la base de datos como cualquier otra: si la transacción se deshace, la fila desaparece y no se envía nada.

Merece la pena saber que el chatter resuelve esto de otra forma. Las notificaciones de `message_post` **sí** se envían enseguida, pero mediante un gancho *post-commit*: se disparan cuando la transacción ya se ha confirmado, en un cursor nuevo. Así consiguen las dos cosas —inmediatez y seguridad ante un `rollback`—. Para el envío de plantillas desde tu propio código, la forma equivalente y sencilla de conseguirlo es dejar el correo en cola.

Un aviso de operación: el cron de la cola viene configurado **cada hora**. En un despliegue donde se espera que los correos salgan en el minuto, hay que bajar ese intervalo desde *Ajustes técnicos → Acciones planificadas*. Es una de las primeras cosas que hay que comprobar cuando alguien dice "los correos de Odoo tardan mucho": muchas veces no tardan, es que esperan al cron.

## Adjuntar un informe PDF

El caso más habitual: mandar el pedido en PDF con el correo. Se declara en la plantilla, no en el código:

```xml
<field name="report_template_ids" eval="[(4, ref('shop.action_report_shop_order'))]"/>
```

En cada envío, Odoo renderiza el informe **para ese registro** y lo adjunta. Es distinto de `attachment_ids`, que son ficheros fijos idénticos en todos los correos (unas condiciones generales, por ejemplo).

Desde Python se puede añadir un adjunto puntual con `email_values`:

```python
pdf_content, _content_type = self.env['ir.actions.report']._render_qweb_pdf(
    'shop.action_report_shop_order', self.ids)
attachment = self.env['ir.attachment'].create({
    'name': '%s.pdf' % self.name,
    'raw': pdf_content,
    'res_model': self._name,
    'res_id': self.id,
    'mimetype': 'application/pdf',
})
template.send_mail(self.id, email_values={'attachment_ids': [attachment.id]})
```

Se usa cuando el adjunto depende de una condición que la plantilla no puede expresar. Si es siempre el mismo informe, `report_template_ids` y a otra cosa.

## El idioma del destinatario

El campo `lang` de la plantilla decide en qué idioma se renderiza, y su valor es una expresión que se evalúa por registro:

```xml
<field name="lang">{{ object.partner_id.lang }}</field>
```

Con esto, el correo del pedido SO0042 sale en el idioma de Marina Costa, aunque quien pulse el botón tenga Odoo en inglés. Sin esto, sale en el idioma de quien envía, que casi nunca es lo que se quiere.

Para que traduzca de verdad hacen falta las dos mitades:

- El `lang` de la plantilla, que selecciona el idioma.
- Las traducciones de los campos `subject` y `body_html`, que son campos traducibles: cada idioma guarda su propia versión, editable desde la plantilla pulsando el indicador de idioma que aparece junto al campo. Lo importante para esta ficha es que ese texto traducido vive **en la plantilla**, no en los ficheros de traducción del código: una plantilla sin la versión catalana se enviará en el idioma original aunque `lang` resuelva a `ca_ES`.

## Las plantillas envolventes (*layouts*)

El correo que sale no es solo tu cuerpo: va dentro de un envoltorio con la cabecera de la compañía, el logo, el pie y a veces un botón "Ver el pedido". Ese envoltorio es otra plantilla QWeb, y se elige con `email_layout_xmlid`:

| XML ID | Cuándo |
|---|---|
| `mail.mail_notification_layout` | El envoltorio estándar |
| `mail.mail_notification_layout_with_responsible_signature` | Añade la firma de la persona responsable del registro |

```python
template.send_mail(self.id, email_layout_xmlid='mail.mail_notification_layout_with_responsible_signature')
```

Con este, el correo de SO0042 termina con la firma de Jordi Vidal. Y como el envoltorio es una vista QWeb normal, se puede heredar para meter la identidad visual propia sin tocar ninguna plantilla de contenido.

## El asistente de envío: dejar que la persona revise antes

Si el correo debe poder repasarse antes de salir, no se llama a `send_mail`: se abre el compositor con la plantilla precargada.

```python
def action_send_by_email(self):
    self.ensure_one()
    template = self.env.ref('shop.mail_template_order_shipped')
    return {
        'type': 'ir.actions.act_window',
        'name': _("Send Order by Email"),
        'res_model': 'mail.compose.message',
        'view_mode': 'form',
        'target': 'new',
        'context': {
            'default_model': self._name,
            'default_res_ids': self.ids,
            'default_template_id': template.id,
            'default_composition_mode': 'comment',
        },
    }
```

Se abre un diálogo con el asunto, el cuerpo y los destinatarios ya rellenos, editables antes de enviar. El `composition_mode` a `comment` hace algo importante: además de enviar el correo, **lo publica en el chatter del pedido**, así que queda constancia de lo que se le dijo al cliente. Con `mass_mail` se enviaría un correo por registro sin registrarlo en el hilo.

Este es el patrón que usan los botones "Enviar por correo" de Odoo, y casi siempre es preferible a un envío directo: quien atiende al cliente puede añadir una frase antes de mandarlo.

## Depurar un correo que no llega

En orden, porque el 90 % de los casos se resuelve en los tres primeros pasos:

1. **¿Existe el `mail.mail`?** *Ajustes → Técnico → Correo electrónico → Correos electrónicos* (hace falta el [modo desarrollador](../fundamentos/Modo-Desarrollador.md)). Si no hay registro, el problema no es de correo: es que el código no llegó a crearlo.
2. **¿En qué estado está?** Si es `outgoing` y no avanza, el cron no está corriendo o su intervalo es muy largo. Si es `exception`, el campo `failure_reason` tiene el error del SMTP en crudo.
3. **¿Hay destinatario?** Un `partner_to` que se renderiza a nada produce un correo sin destinatario. Suele ser un contacto sin campo `email`.
4. **¿Está configurado el servidor de salida?** *Ajustes → Técnico → Correo electrónico → Servidores de correo saliente*, con su botón de "Probar conexión".
5. **¿Es una base de pruebas neutralizada?** En una copia neutralizada el envío está desactivado a propósito, precisamente para no escribir a clientes reales desde un entorno de pruebas. Ver [Neutralizar la base de datos](../pruebas-seguras/Neutralizar-Base-de-Datos.md).
6. **¿Y el log del servidor?** Los fallos de SMTP se registran ahí con detalle.

Un correo en `exception` se puede reintentar desde su propia ficha, con el botón de reenviar. Y `auto_delete` merece una nota: cuando está activo, el `mail.mail` se borra tras enviarse, así que **no habrá rastro que mirar**. Ahorra espacio y complica la depuración; en una plantilla nueva conviene dejarlo desactivado hasta que el flujo esté rodado.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| En el correo se lee literalmente `{{ object.name }}` | Se usó sintaxis *inline* en `body_html`, que es QWeb. Usa `t-out` |
| En el asunto se lee `<t t-out="...">` | Al contrario: QWeb en un campo de una línea. Usa `{{ }}` |
| El cuerpo sale con `False` donde iba un valor | Falta el `or ''` en el `t-out` |
| El correo se crea sin destinatario | `partner_to` se renderizó a vacío, o el contacto no tiene `email` |
| Sale en inglés a un cliente catalán | Falta `lang` en la plantilla, o falta la traducción del `subject`/`body_html` |
| El correo se queda en `outgoing` para siempre | El cron de la cola está desactivado o con un intervalo muy largo |
| Se envió un correo de una operación que luego falló | Se usó `force_send=True`: salió antes del `rollback` |
| La interfaz se congela al enviar a 200 clientes | `force_send=True` en un bucle. Usa la cola y `send_mail_batch` |
| Los importes salen sin símbolo de moneda | Falta `format_amount(...)` |
| Tras actualizar el módulo, el texto editado por el cliente volvió al original | La plantilla no estaba en un bloque `noupdate="1"` |
| No hay rastro del correo enviado | `auto_delete` activo: el `mail.mail` se borró al enviarse |
| `ValueError: External ID not found: shop.model_shop_order` | El XML ID del modelo se escribe con guiones bajos: `model_shop_order` |
| El adjunto llega vacío o corrupto | Se pasó el contenido en base64 donde se esperaba binario (o al revés) |

## Ejemplo completo: enviar el pedido y dejar constancia

Junta la plantilla, el envío y el registro en el historial. El objetivo es que dentro de un año se pueda saber qué se le mandó a Marina y cuándo.

```python
from odoo import _, models
from odoo.exceptions import UserError

class ShopOrder(models.Model):
    _inherit = 'shop.order'

    def action_notify_shipped(self):
        """Send the shipping confirmation to every customer, and log it in each order."""
        template = self.env.ref('shop.mail_template_order_shipped')

        # 1. Comprobar antes de escribir nada: sin correo no hay envío posible.
        without_email = self.filtered(lambda o: not o.partner_id.email)
        if without_email:
            raise UserError(_(
                "These customers have no email address: %s",
                ", ".join(without_email.mapped('partner_id.display_name')),
            ))

        # 2. Envío por lotes y en cola: nada sale del sistema hasta que la transacción cierre.
        mails = template.send_mail_batch(
            self.ids,
            force_send=False,
            email_layout_xmlid='mail.mail_notification_layout_with_responsible_signature',
        )

        # 3. Constancia en el historial de cada pedido, como nota interna.
        for order in self:
            order.message_post(
                body=_("Shipping confirmation queued to %s.", order.partner_id.email),
                subtype_xmlid='mail.mt_note',
            )

        # 4. Resumen efímero para quien pulsó el botón.
        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'type': 'success',
                'message': _("%s confirmation email(s) queued.", len(mails)),
            },
        }
```

El orden vuelve a ser deliberado: la validación va antes de crear nada, el envío queda en cola —así que un fallo posterior no manda correos que no debían salir—, y la nota interna registra **que se encoló**, no que se entregó, porque eso último todavía no ha pasado. Escribir "correo enviado" en el chatter cuando el correo está en cola es una de esas medias verdades que después cuesta desenredar.

## Buenas prácticas avanzadas

- **Deja los correos en cola y no fuerces el envío salvo que sepas por qué.** `force_send=True` cambia dos garantías a la vez: convierte una operación instantánea en una que depende de la latencia de un servidor externo, y saca el correo del alcance del `rollback`. La combinación produce el peor fallo posible en un ERP —el cliente recibe la confirmación de algo que no ocurrió— y no da ningún beneficio si el cron de la cola está bien configurado. Si necesitas inmediatez, baja el intervalo del cron en vez de forzar el envío.
- **`noupdate="1"` en toda plantilla de correo, siempre.** El texto de un correo es contenido que el cliente va a retocar, y una actualización del módulo que lo devuelva a la versión original borra ese trabajo sin avisar. Como contrapartida, tus cambios de texto no llegan solos a las instalaciones existentes: si el cambio es importante, se aplica con un script de migración, no rezando para que nadie hubiera tocado la plantilla.
- **Escribe siempre `lang` en la plantilla, aunque hoy solo tengas un idioma.** Cuesta una línea y evita una reescritura completa el día que entre el primer cliente extranjero. Sin `lang`, el idioma del correo es el de quien pulsa el botón, y ese detalle no se detecta en pruebas: en el entorno de desarrollo todo el mundo tiene el mismo idioma.
- **Usa `format_amount` y `format_date`, nunca interpolación directa.** Un `t-out="object.amount_total"` imprime `149.9`. Los ayudantes aplican el formato de la moneda y de la configuración regional del destinatario, que es justo el tipo de detalle que hace que un correo automático parezca hecho a mano o parezca roto. Y en un importe también es una cuestión de exactitud: sin formato, un total de 1500 puede leerse como 1,5.
- **Aprovecha el texto de previsualización de los `t-out`.** El contenido que dejas dentro de la etiqueta (`<t t-out="object.name">SO0042</t>`) no sale nunca en el correo real, pero es lo que ve quien edita la plantilla desde la interfaz. Poner valores realistas convierte una plantilla ilegible en algo que una persona de marketing puede retocar sin llamarte; poner ahí un `xxx` garantiza que te llamen.
- **Antes de crear una plantilla, comprueba si lo que quieres es el compositor.** Un envío directo con `send_mail` no da ocasión de revisar nada, y en comunicación con clientes esa revisión suele valer más que el ahorro de un clic. El compositor (`mail.compose.message` en modo `comment`) precarga la plantilla, permite añadir una frase y además publica el mensaje en el chatter, con lo que el envío queda auditado sin escribir código extra.

## Documentación oficial

- [Odoo Developer Documentation — Email templates](https://www.odoo.com/documentation/18.0/developer/reference/backend/mixins.html) — la sección sobre el *mixin* de renderizado explica los dos motores (`inline_template` y `qweb`) y qué campos usa cada uno, que es la fuente de la mitad de los errores con plantillas.
- [Código de `mail_template.py`](https://github.com/odoo/odoo/blob/18.0/addons/mail/models/mail_template.py) — la definición de todos los campos con su `help`, y las firmas exactas de `send_mail` y `send_mail_batch`. Es la referencia a consultar antes de pasar un parámetro que no recuerdas.
- [Código de `mail_render_mixin.py`](https://github.com/odoo/odoo/blob/18.0/addons/mail/models/mail_render_mixin.py) — el motor de renderizado: qué variables se inyectan en la plantilla y cómo se resuelve el idioma. Aquí se ve la lista real de ayudantes disponibles (`format_amount`, `format_date`…).
- [Odoo Documentation — Email communication](https://www.odoo.com/documentation/18.0/applications/general/email_communication.html) — la parte de administración: servidores de salida y de entrada, alias, DNS (SPF, DKIM, DMARC) y qué hacer cuando los correos llegan a spam. Es la mitad del problema que no está en el código.

## Recursos didácticos

- [Mail-tester.com](https://www.mail-tester.com/) — envías un correo desde tu Odoo a la dirección que te da y devuelve una puntuación con lo que falta: SPF, DKIM, DMARC, reputación de la IP, HTML mal formado. Es la forma más rápida de descubrir por qué tus correos acaban en spam, un problema que se confunde continuamente con un fallo de Odoo.
- [Can I email…](https://www.caniemail.com/) — el equivalente de "Can I use" para clientes de correo: qué CSS y qué etiquetas HTML soporta Outlook, Gmail o Apple Mail. Explica por qué el cuerpo de las plantillas de Odoo está lleno de estilos *inline* y tablas en vez de flexbox.
- [Plantillas de correo del propio Odoo](https://github.com/odoo/odoo/tree/18.0/addons/sale/data) — las plantillas de los módulos estándar son el mejor material de estudio: enseñan el uso de `t-set`, `t-if`, `format_amount` y los textos de previsualización en casos reales y probados en producción.

---

*En resumen: una plantilla de correo separa el texto del código para que se pueda editar y traducir sin desplegar —y su regla de oro es dejar el envío en la cola, porque un correo ya entregado al SMTP no lo deshace ningún `rollback`.*
