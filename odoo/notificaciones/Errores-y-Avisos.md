# Errores y avisos bloqueantes

## ¿Qué es?

El mecanismo con el que Odoo dice "no" a quien está usando el sistema: **lanzar una excepción**. Un `raise UserError(...)` en Python se convierte en un diálogo modal con tu mensaje, y —esto es la mitad del tema— **deshace todo lo que la operación había hecho**. Junto a ese mecanismo bloqueante existe un primo pequeño y no bloqueante, el `warning` de `@api.onchange`, que advierte sin abortar nada.

## ¿Por qué existe?

Porque una parte de las reglas de negocio no son "avisos": son condiciones que no se pueden violar. Un pedido no puede enviarse sin dirección de entrega, una factura validada no puede cambiar de importe, un usuario no puede leer los datos de otra compañía. Si el sistema solo mostrase un aviso amable y siguiera adelante, la base de datos acabaría con estados imposibles.

Podría resolverse devolviendo códigos de error y comprobándolos en cada llamada, pero eso obliga a que **todos** los llamantes se acuerden de mirar. Odoo eligió el camino contrario: la excepción sube sola por la pila hasta la capa que atiende la petición, que se encarga de dos cosas de golpe —hacer `rollback` de la transacción y mandar el mensaje al navegador— sin que ninguna capa intermedia tenga que colaborar.

> Si has trabajado con HTTP a mano, `UserError` es el equivalente a devolver un 400 con un mensaje para la persona, y una excepción no controlada es un 500 con traza. La diferencia es que aquí el `rollback` de la base de datos viene incluido.

## ¿Cuándo y para qué se usa?

Cuando la operación **no debe completarse**. Ese es el único criterio, y es más restrictivo de lo que parece: si la operación puede seguir adelante, lo que quieres es un [aviso flotante](Notificaciones-de-Interfaz.md), un [mensaje en el chatter](Chatter-y-Seguidores.md) o un `warning` de `onchange`, no una excepción.

Casos típicos:

- Transiciones de estado inválidas: confirmar un pedido ya cancelado.
- Datos obligatorios que faltan justo cuando se necesitan, no antes.
- Reglas de negocio que cruzan varios registros: un descuento superior al que permite el perfil de quien lo aplica.
- Invariantes de datos: un total negativo, una fecha de entrega anterior a la del pedido.

Seguimos con el modelo `shop.order` de la tienda online y el pedido **SO0042**, del cliente Marina Costa, con el comercial Jordi Vidal.

---

## El catálogo de excepciones

Odoo define un puñado de excepciones en `odoo/exceptions.py`, y el cliente web **solo sabe presentar amablemente esas**. Todo lo demás se muestra como un error de servidor con traza.

| Excepción | Cuándo se usa | Qué ve la persona |
|---|---|---|
| `UserError` | El caso general: la operación no procede | Diálogo con tu mensaje |
| `ValidationError` | Una restricción de datos ha fallado (idiomático en `@api.constrains`) | Diálogo con tu mensaje |
| `AccessError` | No tiene permiso sobre el registro o el modelo | Diálogo con tu mensaje |
| `AccessDenied` | Credenciales incorrectas | Diálogo, **sin traza** |
| `MissingError` | El registro ya no existe | Diálogo con tu mensaje |
| `RedirectWarning` | No procede, **y hay un sitio donde arreglarlo** | Diálogo con un botón que lleva allí |
| Cualquier otra | Un fallo que no habías previsto | "Server error" y traza completa |

Todas menos `RedirectWarning` heredan de `UserError`, que a su vez hereda de `Exception`. La consecuencia práctica: un `except UserError` captura también `ValidationError`, `AccessError` y `MissingError`.

La forma de usarlas es la misma en todos los casos:

```python
from odoo import _, api, fields, models
from odoo.exceptions import UserError

class ShopOrder(models.Model):
    _inherit = 'shop.order'

    def action_ship(self):
        for order in self:
            if order.state != 'confirmed':
                raise UserError(_(
                    "Order %(name)s is in state %(state)s: only confirmed orders can be shipped.",
                    name=order.name, state=order.state,
                ))
            order.state = 'shipped'
```

Si el segundo pedido del conjunto está en borrador, sale el diálogo con "Order SO0043 is in state draft: only confirmed orders can be shipped." Y ahora la parte que importa: **el primer pedido tampoco se ha enviado**, aunque su `write` ya se había ejecutado. La sección siguiente explica por qué.

## Lo que de verdad hace un `raise`

Esto es el corazón de la ficha. En Odoo, cada petición del cliente se atiende dentro de **una transacción de PostgreSQL**. Si el método termina bien, la transacción se confirma (`COMMIT`); si sale una excepción, se deshace (`ROLLBACK`).

Es decir: **lanzar una excepción no es "mostrar un mensaje", es "cancelar todo lo hecho en esta petición y mostrar un mensaje"**. Las dos cosas van juntas y no se pueden separar.

```python
def action_process(self):
    self.write({'state': 'confirmed'})          # 1. se ejecuta
    self.message_post(body=_("Confirmed."))     # 2. se ejecuta
    self.env['shop.log'].create({'note': 'ok'}) # 3. se ejecuta
    raise UserError(_("Something is off."))     # 4. deshace 1, 2 y 3
```

Después de esto, en la base de datos no queda **nada** de las tres primeras líneas: ni el estado, ni el mensaje del chatter, ni el registro de log. Esto es casi siempre lo que quieres —es lo que hace que un error no deje datos a medias— pero hay que tenerlo presente, porque destruye dos patrones que la gente intenta a menudo:

**Patrón roto 1: dejar constancia del error en el chatter.**

```python
# ❌ El mensaje se deshace junto con todo lo demás
self.message_post(body=_("Shipping failed: no tracking reference."))
raise UserError(_("No tracking reference."))
```

Si el mensaje del chatter tiene que sobrevivir, no puede haber `raise` en esa transacción. Las salidas son: no lanzar la excepción y devolver un aviso flotante, o registrar el problema fuera de la base de datos con `_logger.warning(...)`, que escribe en el fichero de log y no depende de la transacción.

**Patrón roto 2: procesar en lote "todo lo que se pueda".**

```python
# ❌ Un solo pedido problemático tira los 199 anteriores
for order in self:
    if not order.carrier_tracking_ref:
        raise UserError(_("Order %s has no tracking reference.", order.name))
    order.state = 'shipped'
```

Sobre 200 pedidos, el que falle deshace el trabajo de todos. La versión correcta separa el grano de la paja **antes** de escribir y usa un aviso flotante para el resumen:

```python
# ✅ Procesa lo válido y resume al final, sin excepciones
shippable = self.filtered(lambda o: o.state == 'confirmed' and o.carrier_tracking_ref)
blocked = self - shippable
shippable.write({'state': 'shipped'})
return {
    'type': 'ir.actions.client',
    'tag': 'display_notification',
    'params': {
        'type': 'warning' if blocked else 'success',
        'message': _("%(ok)s shipped, %(ko)s skipped.", ok=len(shippable), ko=len(blocked)),
        'sticky': bool(blocked),
    },
}
```

### Lo que el `rollback` no puede deshacer

El `rollback` solo alcanza a la base de datos. Todo lo que ya haya salido del proceso sigue su curso propio:

- **Un correo ya entregado al servidor SMTP.** Por eso `send_mail` deja el correo en cola por defecto (`force_send=False`): así el envío ocurre después del `commit`, en otra transacción, y un error posterior no manda correos fantasma. Ver [Plantillas de correo](Plantillas-de-Correo.md).
- **Una llamada a una API externa.** Si has cobrado con la pasarela de pago y luego lanzas un `UserError`, el cobro está hecho y el pedido no. Estas llamadas van al final del método, cuando ya no queda nada que pueda fallar, o se compensan explícitamente.
- **Lo escrito en el fichero de log.** Que es justo por lo que sirve para registrar errores.
- **Un `self.env.cr.commit()` que hayas escrito tú.** No lo escribas. Confirmar a mano dentro de un método parte la transacción en dos y deja la mitad del trabajo hecho cuando algo falla después; el sitio donde eso es legítimo son los crons que procesan colas, y esos ya lo hacen por su cuenta.

### Seguir adelante a pesar de un error: `savepoint`

Cuando de verdad hace falta procesar 200 registros aislando los que fallen, existe el punto de guardado: un `rollback` parcial que solo deshace lo hecho dentro del bloque.

```python
failed = self.env['shop.order']
for order in self:
    try:
        with self.env.cr.savepoint():
            order._ship()          # puede lanzar cualquier cosa
    except UserError:
        failed |= order            # este pedido queda como estaba; los demás siguen
```

Cada iteración se protege por separado: si `_ship()` lanza una excepción, se deshace solo lo de ese pedido y el bucle continúa. Es la herramienta correcta para importaciones y procesos por lotes, y hay que usarla con cabeza: capturar `Exception` a secas aquí esconde errores de programación, así que se captura lo que se espera (`UserError`, `ValidationError`) y no más.

## `UserError` frente a `ValidationError`

Para el cliente web son idénticas: las dos muestran un diálogo con el mensaje. La diferencia es de **convención**, y sirve para que el código se lea:

- `ValidationError` — "los datos de este registro no son válidos". Es la que se usa en `@api.constrains`.
- `UserError` — "esta acción no se puede hacer ahora". Es la de los métodos de negocio.

```python
@api.constrains('amount_total')
def _check_amount_total(self):
    for order in self:
        if order.amount_total < 0:
            raise ValidationError(_(
                "Order %s cannot have a negative total.", order.name))
```

Un `@api.constrains` se ejecuta automáticamente en cada `create` y en cada `write` que toque uno de los campos declarados, así que esa comprobación se aplica venga el cambio de la interfaz, de una importación o de otro módulo. Es la diferencia con validar dentro de `action_confirm`: eso solo protege ese botón.

Hay una variante que no pasa por Python: las restricciones de PostgreSQL. En Odoo 18 se declaran con `_sql_constraints` y su mensaje se muestra igual que una excepción:

```python
_sql_constraints = [
    ('name_uniq', 'unique(name)', "The order reference must be unique."),
]
```

Ventaja: la garantiza la base de datos, así que no hay forma de saltársela ni con SQL directo. Límite: el mensaje es fijo y no puede incluir el valor concreto que ha chocado.

## `RedirectWarning`: bloquear y ofrecer la salida

Es la excepción más útil y la menos usada. Sirve para cuando el problema tiene un sitio concreto donde arreglarse: en vez de decir "falta configurar el servidor de correo", te lleva a la pantalla del servidor de correo.

Su firma es `RedirectWarning(message, action, button_text, additional_context=None)`, donde `action` es el **ID de una acción de ventana**:

```python
from odoo.exceptions import RedirectWarning

def action_send_confirmation(self):
    self.ensure_one()
    if not self.partner_id.email:
        action = self.env.ref('base.action_partner_form')
        raise RedirectWarning(
            _("%s has no email address, so the confirmation cannot be sent.",
              self.partner_id.display_name),
            action.id,
            _("Open customer"),
            {'active_id': self.partner_id.id, 'res_id': self.partner_id.id},
        )
    ...
```

El diálogo muestra el mensaje y **dos** botones: "Close" y "Open customer". El segundo abre la ficha de Marina Costa, con el `additional_context` inyectado en el contexto de la acción, para que se pueda rellenar el correo que falta sin ir a buscarlo por los menús.

El `rollback` funciona igual que con `UserError`: se deshace todo. El valor añadido es puramente de experiencia de uso, y es enorme —convierte un callejón sin salida en un camino— así que merece la pena acordarse de ella cada vez que un mensaje de error contenga la frase "ve a configurar".

## Excepciones no controladas: cuándo sale "Server error"

Cualquier excepción que no sea de las que el cliente conoce se presenta como un error interno, con la traza completa visible:

```python
def action_apply_discount(self):
    self.amount_total = self.amount_total / self.discount_factor   # 💥 si es 0
```

Con `discount_factor` a cero, quien pulse el botón ve un diálogo con:

```
ZeroDivisionError: division by zero
Traceback (most recent call last):
  File "/opt/odoo/odoo/http.py", line ...
```

Esto no es "un error mal hecho": es información valiosa **para quien desarrolla** y basura para quien vende. La lectura correcta de un "Server error" en producción es que hay un caso no previsto en el código, y la solución no es envolverlo en un `try/except` para que salga bonito, sino comprobar la condición antes:

```python
if not self.discount_factor:
    raise UserError(_("Set a discount factor before applying a discount."))
```

Por eso conviene no capturar `Exception` a la ligera. Convertir un `KeyError` en un `UserError` con el texto "Ha ocurrido un error" hace desaparecer la traza que decía en qué línea y con qué dato falló, y con ella la única pista para arreglarlo.

## Avisos que **no** bloquean: el `warning` de `onchange`

Este es el otro mecanismo de la ficha, y es exactamente lo contrario: advierte sin abortar. Vive en un método `@api.onchange`, se dispara mientras se rellena el formulario —antes de guardar— y se declara devolviendo un diccionario:

```python
@api.onchange('partner_id')
def _onchange_partner_id_warn_blocked(self):
    """Warn when the selected customer has unpaid invoices, but let the user continue."""
    if self.partner_id and self.partner_id.has_overdue_invoices:
        return {'warning': {
            'title': _("Overdue invoices"),
            'message': _("%s has overdue invoices. Confirm the order at your own risk.",
                         self.partner_id.display_name),
        }}
```

Al elegir a Marina Costa como cliente aparece un aviso amarillo con ese texto. El formulario **sigue editable**, no se ha escrito nada en la base de datos —un `onchange` trabaja sobre un registro en memoria— y se puede guardar el pedido igualmente. Es el mecanismo correcto para "esto es raro pero es tu decisión".

Hay una variante que muchos no conocen: pedir que el aviso salga como **modal** en lugar de como aviso flotante, añadiendo `type: 'dialog'`.

```python
return {'warning': {
    'type': 'dialog',
    'title': _("Overdue invoices"),
    'message': _("%s has overdue invoices.", self.partner_id.display_name),
}}
```

Con `dialog` hay que cerrar la ventana para seguir: no bloquea la operación, pero garantiza que el aviso se ha leído. Sin él, sale un aviso flotante que se va solo (y acepta `sticky` para que no se vaya).

Dos límites del `onchange` que causan sorpresas:

- **Solo se dispara en un formulario.** Una importación, una llamada RPC o un `create` desde otro módulo no pasan por ahí. Un `onchange` **no es una validación**: si la regla tiene que cumplirse siempre, va además en `@api.constrains`.
- **Solo puede devolver un `warning` por llamada.** Si dos condiciones son ciertas, hay que juntar los dos textos en un mensaje.

## La tercera vía: el aviso dentro de la vista

Ni excepción ni aviso flotante: un bloque de color en el propio formulario, que aparece cuando se cumple una condición y no interrumpe nada.

```xml
<form>
    <div class="alert alert-warning" role="alert"
         invisible="not partner_id or partner_id_has_overdue">
        This customer has overdue invoices. Check with the finance team before shipping.
    </div>
    <group>
        <field name="partner_id"/>
        <field name="partner_id_has_overdue" invisible="1"/>
    </group>
</form>
```

El bloque solo se ve cuando el campo auxiliar `partner_id_has_overdue` es verdadero, y se queda ahí mientras la condición dure. Es la mejor opción para un aviso **permanente y contextual**: no se pierde como un aviso flotante, no molesta como un diálogo y está pegado a los datos que lo provocan. La contrapartida es que necesita un campo (normalmente calculado y no almacenado) que exprese la condición.

## Cómo elegir entre los cuatro

| Situación | Mecanismo |
|---|---|
| La operación no puede completarse | `UserError` |
| Los datos del registro violan una regla que debe cumplirse siempre | `ValidationError` en `@api.constrains` |
| La unicidad o la integridad la debe garantizar la base de datos | `_sql_constraints` |
| No puede completarse y hay una pantalla donde arreglarlo | `RedirectWarning` |
| Es dudoso pero se permite, y se decide al rellenar el formulario | `warning` de `@api.onchange` |
| Es dudoso y debe verse todo el rato mientras dure | `<div class="alert">` en la vista |
| Ha ido bien, con matices | [Aviso flotante](Notificaciones-de-Interfaz.md) |

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| El mensaje del chatter que escribiste antes del `raise` no aparece | El `rollback` lo deshizo. Usa `_logger` o no lances la excepción |
| Un proceso por lotes no hace nada cuando un registro falla | El `raise` deshace la transacción entera. Filtra antes, o usa `savepoint` |
| Sale "Server error" con traza en vez de tu mensaje | Se está lanzando una excepción que el cliente no conoce (`ValueError`, `KeyError`, `ZeroDivisionError`…) |
| `TypeError: __init__() takes 2 positional arguments but 3 were given` | `UserError` recibe **un** argumento: el mensaje ya formateado. Formatea con `_("... %s", valor)`, no con dos parámetros |
| El `@api.constrains` no se dispara | El campo modificado no está en la lista del decorador, o el cambio se hizo con SQL directo |
| El `onchange` no avisa al importar un fichero | Los `onchange` solo corren en un formulario. La validación va en `@api.constrains` |
| El aviso de `onchange` sale y el usuario no lo ve | Es un aviso flotante que se va en 4 s. Usa `type: 'dialog'` o `sticky` |
| El `RedirectWarning` da error al construirse | El segundo argumento es el **ID** de la acción (`action.id`), no el registro |
| El correo se envió aunque la operación falló | Se usó `force_send=True`: el correo salió antes del `rollback` |
| Se pierde la información de un error de un módulo externo | Un `except Exception` demasiado ancho. Captura lo que esperas |

## Ejemplo completo: proteger la máquina de estados de un pedido

Junta los mecanismos en la proporción en que aparecen en código real: una restricción para el invariante, una excepción para la transición, un `RedirectWarning` para lo que se arregla en otra pantalla y un aviso no bloqueante para lo dudoso.

```python
from odoo import _, api, fields, models
from odoo.exceptions import UserError, ValidationError, RedirectWarning

class ShopOrder(models.Model):
    _inherit = 'shop.order'

    # 1. Invariante de datos: se cumple siempre, venga el cambio de donde venga.
    @api.constrains('amount_total')
    def _check_amount_total(self):
        for order in self:
            if order.amount_total < 0:
                raise ValidationError(_("Order %s cannot have a negative total.", order.name))

    # 2. Aviso no bloqueante mientras se rellena el formulario.
    @api.onchange('partner_id')
    def _onchange_partner_id_warn_overdue(self):
        if self.partner_id and self.partner_id.has_overdue_invoices:
            return {'warning': {
                'title': _("Overdue invoices"),
                'message': _("%s has overdue invoices.", self.partner_id.display_name),
            }}

    def action_ship(self):
        self.ensure_one()

        # 3. Transición inválida: la operación no procede.
        if self.state != 'confirmed':
            raise UserError(_(
                "Only confirmed orders can be shipped. Order %(name)s is %(state)s.",
                name=self.name, state=self.state,
            ))

        # 4. Falta un dato que se arregla en otra pantalla: ofrécela.
        if not self.partner_id.street:
            action = self.env.ref('base.action_partner_form')
            raise RedirectWarning(
                _("%s has no delivery address.", self.partner_id.display_name),
                action.id,
                _("Complete the address"),
                {'active_id': self.partner_id.id, 'res_id': self.partner_id.id},
            )

        # 5. Todo en orden: escribe, deja constancia y confirma.
        self.state = 'shipped'
        self.message_post(body=_("Order shipped to %s.", self.partner_id.display_name))
        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {'type': 'success', 'message': _("Order %s shipped.", self.name)},
        }
```

El orden de los bloques dentro de `action_ship` no es casual: **todas las comprobaciones van antes de la primera escritura**. Así, cuando una falla, no hay nada que deshacer y el `rollback` es un no-operación; si se escribiera primero y se validara después, el resultado sería el mismo para la base de datos, pero cualquier efecto externo lanzado en medio (un correo, una llamada a una API) ya no tendría marcha atrás.

## Buenas prácticas avanzadas

- **Un mensaje de error debe decir qué registro, qué esperaba y qué hacer.** "Invalid state" obliga a quien lo recibe a abrir un ticket. "Only confirmed orders can be shipped. Order SO0042 is draft." se resuelve sin ayuda. La prueba es imaginar el mensaje pegado en un correo, sin captura y sin contexto: si así no se puede actuar, falta información. Y como el mensaje lleva datos variables, va con parámetros con nombre (`_("... %(name)s", name=...)`), que es lo que permite a un traductor reordenar la frase.
- **Valida en el sitio con el alcance correcto, no en el más cómodo.** Poner la comprobación dentro del método del botón protege ese botón; ponerla en `@api.constrains` protege también las importaciones, las llamadas RPC y el módulo que alguien instale el año que viene; ponerla en `_sql_constraints` la protege incluso de un `UPDATE` a mano en la base de datos. La pregunta no es dónde es más fácil escribirla, sino desde cuántos sitios se puede violar la regla.
- **Nunca cambies el estado y luego valides.** Aunque el `rollback` te cubra en la base de datos, mientras la transacción vive tu código trabaja con datos que no son válidos, y cualquier cosa que dispares en medio —un `message_post`, un cálculo de campo, un método heredado por otro módulo— los ve. Validar primero y escribir después evita toda una familia de errores difíciles de reproducir.
- **`_logger` es el único registro que sobrevive a un `raise`.** Cuando necesites que quede constancia de un fallo que aborta la operación, `_logger.warning("Cannot ship %s: %s", order.name, reason)` escribe en el log del servidor y no lo deshace nadie. Es la técnica que permite investigar después "¿cuántas veces ha pasado esto?", una pregunta que el chatter no puede contestar porque sus mensajes se deshicieron con la transacción.
- **Trata "Server error" como un aviso de que falta una comprobación, no como un problema de presentación.** La tentación es envolver el método en un `try/except Exception` que lo convierta en un `UserError` genérico. Eso oculta el fallo, no lo arregla, y borra la traza: la próxima vez que ocurra nadie sabrá dónde mirar. Cada "Server error" que aparece en producción es una condición que hay que comprobar explícitamente antes.
- **Si atrapas una excepción, atrápala estrecha y vuelve a lanzar lo que no esperabas.** `except (UserError, ValidationError)` en un bucle con `savepoint` es correcto; `except Exception: pass` convierte un error de programación en datos silenciosamente corruptos. Cuando de verdad necesitas capturar ancho —al hablar con un servicio externo, por ejemplo— registra la excepción original con `_logger.exception(...)` antes de convertirla, para no perder la traza.

## Documentación oficial

- [Código de `odoo/exceptions.py`](https://github.com/odoo/odoo/blob/18.0/odoo/exceptions.py) — la lista completa y comentada de excepciones que el cliente sabe interpretar, con la firma exacta de `RedirectWarning`. Son cien líneas y responden la mayoría de las dudas.
- [Odoo Developer Documentation — Constraints](https://www.odoo.com/documentation/18.0/developer/reference/backend/orm.html#constraints) — la sección del ORM sobre `@api.constrains` y las restricciones SQL: cuándo se disparan y qué campos vigilan.
- [Odoo Developer Documentation — Onchange](https://www.odoo.com/documentation/18.0/developer/reference/backend/orm.html#odoo.api.onchange) — el contrato de `@api.onchange`, incluido el diccionario de retorno con la clave `warning`.
- [PostgreSQL — Transacciones y SAVEPOINT](https://www.postgresql.org/docs/current/sql-savepoint.html) — la fuente normativa de lo que hace `self.env.cr.savepoint()` por debajo, útil cuando hay que razonar qué se deshace exactamente.

## Recursos didácticos

- [Módulo `test_exceptions` de Odoo](https://github.com/odoo/odoo/tree/18.0/odoo/addons/test_exceptions) — un módulo del propio Odoo cuyo único propósito es lanzar cada tipo de excepción desde un botón. Instálalo en una base de datos de pruebas y verás con tus ojos la diferencia entre un `UserError`, un `RedirectWarning` y un error no controlado.
- [HTTP Cats](https://http.cat/) — los códigos de estado HTTP ilustrados. Viene a cuento porque la distinción que hace Odoo entre `UserError` (culpa de quien pide) y error no controlado (culpa del servidor) es exactamente la de los códigos 4xx y 5xx, y con gatos se recuerda mejor.

---

*En resumen: en Odoo un error no es un mensaje, es un `rollback` con mensaje —así que lanza una excepción solo cuando de verdad quieras que no quede nada de lo hecho, y usa avisos no bloqueantes para todo lo demás.*
