# Notificaciones de interfaz (*toasts*)

## ¿Qué es?

El rectángulo que aparece en la esquina superior derecha de Odoo, dice "Registro guardado" o "Nada que enviar" y desaparece a los cuatro segundos. En la jerga se llama *toast* o *notificación de interfaz*, y se puede disparar tanto desde Python (devolviendo una acción de cliente) como desde JavaScript (llamando a un servicio).

## ¿Por qué existe?

Porque hay una categoría de aviso que no merece interrumpir a nadie: la confirmación de que algo ha ido bien. Si cada "guardado correctamente" fuera un diálogo modal, trabajar en Odoo sería insoportable —habría que cerrar una ventana cada dos clics—. Y si no hubiera aviso ninguno, quien pulsa un botón que no cambia nada visible no sabría si funcionó.

El *toast* ocupa ese hueco: **se ve sin que haya que hacer nada y se va sin que haya que hacer nada**. El precio es que no deja rastro: si la persona miraba a otro lado, ese aviso no existió.

> Si has usado alguna librería de *toasts* del frontend (`react-toastify`, el `snackbar` de Material, `toastr`), es exactamente eso, con la diferencia de que en Odoo se puede lanzar **desde el servidor** sin escribir una línea de JavaScript.

## ¿Cuándo y para qué se usa?

Para *feedback* inmediato dirigido a quien acaba de actuar, cuando el aviso no tiene que sobrevivir a la sesión:

- Confirmar el resultado de una acción de servidor: "3 pedidos confirmados", "No se ha enviado nada: no había pedidos pendientes".
- Informar de que algo se ha lanzado en segundo plano: "Las facturas se están enviando".
- Avisar de un resultado parcial: "18 de 20 correos enviados; 2 sin dirección".
- Errores no bloqueantes en el navegador: "No se pudo copiar al portapapeles".

Y **no** se usa para lo que tiene que quedar registrado (eso es [chatter](Chatter-y-Seguidores.md)), para lo que debe impedir una operación (eso es una [excepción](Errores-y-Avisos.md)) ni para avisar a otra persona (eso es el bus (`bus.bus`) o el chatter).

Seguimos con el ejemplo de la colección: un modelo `shop.order` de una tienda online, y el pedido **SO0042** del cliente Marina Costa.

---

## El camino desde Python: `ir.actions.client`

Python no puede "mostrar" nada: solo puede **devolver** algo que el navegador sepa interpretar. Ese algo es un diccionario de acción con `type: 'ir.actions.client'` y la etiqueta `display_notification`:

```python
from odoo import _, models

class ShopOrder(models.Model):
    _inherit = 'shop.order'

    def action_confirm(self):
        for order in self.filtered(lambda o: o.state == 'draft'):
            order.state = 'confirmed'
        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'type': 'success',
                'title': _("Orders confirmed"),
                'message': _("%s order(s) moved to confirmed.", len(self)),
                'sticky': False,
            },
        }
```

Al pulsar el botón, el navegador recibe ese diccionario, reconoce la etiqueta y muestra un aviso verde con el título "Orders confirmed" y el texto "3 order(s) moved to confirmed.". Desaparece solo.

Lo importante de este mecanismo, y la fuente de casi todos los problemas con él: **es un valor de retorno**. Si tu método ya devuelve otra cosa —una acción de ventana para abrir un registro, por ejemplo—, no puedes devolver las dos. Ese límite y cómo saltárselo se ve más abajo, en la sección del parámetro `next`.

### Los parámetros de `params`

| Parámetro | Tipo | Qué hace |
|---|---|---|
| `message` | str | El cuerpo del aviso. **Obligatorio** en la práctica |
| `title` | str | Un título en negrita sobre el mensaje. Opcional |
| `type` | str | `success`, `warning`, `danger` o `info`. Por defecto `info` |
| `sticky` | bool | Si es `True`, no se cierra solo: hay que pulsar la X. Por defecto `False` |
| `className` | str | Clases CSS extra para el contenedor |
| `links` | list | Enlaces que se insertan en el mensaje (ver abajo) |
| `next` | dict | Otra acción a ejecutar después de mostrar el aviso |

### Los cuatro tipos y qué comunica cada uno

El `type` no es solo color: es la expectativa que creas en quien lo lee.

```python
# La operación ha terminado y el resultado es el esperado.
{'type': 'success', 'message': _("Order %s confirmed.", self.name)}

# Ha terminado, pero hay algo que la persona debería mirar.
{'type': 'warning', 'message': _("Order confirmed, but the customer has no email address.")}

# No ha terminado. Algo ha fallado y no se ha hecho lo que se pedía.
{'type': 'danger', 'message': _("The carrier API is unreachable. Try again later.")}

# Información neutra: ni éxito ni problema.
{'type': 'info', 'message': _("Invoices are being sent in the background.")}
```

Un detalle que despista al depurar: si **no** pasas `type`, la acción de Python usa `info`, pero el componente de JavaScript que dibuja el aviso tiene `warning` como valor por defecto de la propiedad. Es decir, un *toast* lanzado desde Python sin `type` sale azul, y uno lanzado desde JavaScript sin `type` sale amarillo. No es un error: son dos capas con defectos distintos. Pasa siempre `type` explícitamente y no tendrás que acordarte de esto.

### `sticky`: cuándo quitar el temporizador

Por defecto el aviso se va en cuatro segundos. Con `sticky: True` se queda hasta que la persona lo cierre:

```python
return {
    'type': 'ir.actions.client',
    'tag': 'display_notification',
    'params': {
        'type': 'warning',
        'title': _("Partial import"),
        'message': _("42 of 50 orders imported. Check the log for the 8 failures."),
        'sticky': True,
    },
}
```

La regla práctica: **si el mensaje contiene información que la persona necesita leer con calma o copiar, `sticky`**. Un "42 de 50" con instrucciones no se puede leer en cuatro segundos, y no hay forma de recuperar un *toast* que ya se fue. Para todo lo demás, sin `sticky`: un aviso pegajoso de "Guardado" obliga a un clic extra y molesta.

### `links`: meter un enlace dentro del mensaje

Los avisos no aceptan HTML arbitrario en el mensaje —el texto se escapa por seguridad—, pero sí un mecanismo controlado de enlaces. Se ponen marcadores `%s` en el mensaje y se pasa una lista de `{'label': ..., 'url': ...}` en el mismo orden:

```python
return {
    'type': 'ir.actions.client',
    'tag': 'display_notification',
    'params': {
        'type': 'info',
        'message': _("Order created. See it here: %s"),
        'links': [{
            'label': self.name,
            'url': f'/odoo/shop.order/{self.id}',
        }],
    },
}
```

El aviso muestra "Order created. See it here: SO0042", donde SO0042 es un enlace que abre el pedido en una pestaña nueva. Por dentro, el cliente escapa la etiqueta y la URL, construye la etiqueta `<a>` y la sustituye en el `%s`. Fíjate en que el `%s` va **dentro** de la cadena traducible y sin interpolar: el `_()` recibe el texto con el marcador, y la sustitución la hace el navegador.

### `next`: hacer algo más después del aviso

Este parámetro resuelve el límite de "solo puedo devolver una cosa". Lo que pongas en `next` es otra acción, que el cliente ejecuta después de mostrar el aviso. El uso más común, con diferencia, es cerrar el asistente desde el que se pulsó el botón:

```python
def action_send_all(self):
    self._do_send()
    return {
        'type': 'ir.actions.client',
        'tag': 'display_notification',
        'params': {
            'type': 'success',
            'message': _("Orders sent."),
            'next': {'type': 'ir.actions.act_window_close'},
        },
    }
```

Sin el `next`, el asistente se quedaría abierto detrás del aviso, y quien lo usa tendría que cerrarlo a mano. Con él, la ventana se cierra y el aviso se queda flotando encima de la vista de origen.

`next` acepta cualquier acción, así que también sirve para recargar la vista o abrir un registro:

```python
# Avisar y recargar el listado, para que se vean los estados nuevos
'next': {'type': 'ir.actions.act_window', 'res_model': 'shop.order', 'views': [[False, 'list']]},

# Avisar y abrir el pedido recién creado
'next': {
    'type': 'ir.actions.act_window',
    'res_model': 'shop.order',
    'res_id': order.id,
    'views': [[False, 'form']],
},
```

## El *rainbow man*: la celebración

Odoo tiene un segundo mecanismo efímero, reservado para hitos: el arcoíris con un mensaje que aparece centrado en la pantalla. No es un `display_notification`, sino una clave `effect` en la acción devuelta:

```python
def action_mark_won(self):
    self.state = 'confirmed'
    return {
        'effect': {
            'type': 'rainbow_man',
            'message': _("Great! Order %s is confirmed.", self.name),
            'fadeout': 'slow',
            'img_url': '/web/static/img/smile.svg',
        }
    }
```

`fadeout` acepta `fast`, `medium`, `slow` y `no` (con `no` se queda hasta que se pulsa fuera). Y tiene una propiedad que conviene conocer: **si la persona ha desactivado los efectos** en sus preferencias, el mismo mensaje se muestra como un *toast* normal. No hay que programar la alternativa: ya está.

Úsalo con cuentagotas. Está pensado para "has cerrado la venta", no para "se ha guardado el formulario"; un arcoíris cada vez que se pulsa un botón deja de significar nada en dos días.

## Desde JavaScript: el servicio `notification`

Cuando el aviso nace en el navegador —una validación de formulario, un error de una llamada al servidor, una copia al portapapeles— no hay que dar el viaje al servidor: se llama al servicio `notification`.

En un componente Owl se obtiene con `useService` y se usa con `add`:

```javascript
import { Component } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";
import { _t } from "@web/core/l10n/translation";

export class TrackingButton extends Component {
    setup() {
        this.notification = useService("notification");
    }

    onCopyTracking() {
        navigator.clipboard.writeText(this.props.record.data.carrier_tracking_ref);
        this.notification.add(_t("Tracking reference copied."), {
            type: "success",
        });
    }
}
```

Al pulsar el botón se copia la referencia y sale un aviso verde. Nótese que en JavaScript la función de traducción es `_t`, no `_`.

La firma es `add(message, options)` y las opciones son casi las mismas que en Python, con dos añadidos que Python no tiene:

| Opción | Qué hace |
|---|---|
| `title` | Título en negrita |
| `type` | `success`, `warning`, `danger`, `info` (por defecto `warning`) |
| `sticky` | No cerrar automáticamente |
| `autocloseDelay` | Milisegundos antes de cerrarse. Por defecto 4000 |
| `className` | Clases CSS extra |
| `onClose` | Función que se ejecuta al cerrarse |
| `buttons` | **Botones dentro del aviso** |

### Botones dentro del aviso

Es la capacidad que no existe desde Python y la razón para bajar a JavaScript en muchos casos. Cada botón lleva su texto y su función:

```javascript
this.notification.add(_t("Order archived."), {
    type: "info",
    sticky: true,
    buttons: [{
        name: _t("Undo"),
        primary: true,
        onClick: () => this.restoreOrder(),
    }],
});
```

El aviso se queda en pantalla —`sticky: true` es obligatorio aquí, si no el botón se va antes de que nadie pueda pulsarlo— y ofrece un "Undo" que llama a `restoreOrder()`. Este patrón, avisar con posibilidad de deshacer, es mucho más amable que un diálogo de "¿Estás seguro?" antes de cada acción.

### Cerrar un aviso desde el código

`add` devuelve la función que lo cierra. Sirve para avisos de progreso que se sustituyen a sí mismos:

```javascript
const closeProgress = this.notification.add(_t("Sending orders…"), {
    type: "info",
    sticky: true,
});
try {
    await this.orm.call("shop.order", "action_send_all", [this.orderIds]);
    closeProgress();
    this.notification.add(_t("Orders sent."), { type: "success" });
} catch {
    closeProgress();
    this.notification.add(_t("Sending failed."), { type: "danger" });
}
```

Mientras la llamada está en marcha hay un aviso pegajoso de "Sending orders…"; cuando termina, se cierra y se sustituye por el resultado. Sin ese `closeProgress()`, el aviso de progreso se quedaría en pantalla para siempre.

## Cuando el *toast* no es suficiente: el servicio `dialog`

Si el aviso tiene que interrumpir de verdad —porque hace falta una decisión antes de seguir— el mecanismo es un diálogo, no un *toast*:

```javascript
import { ConfirmationDialog } from "@web/core/confirmation_dialog/confirmation_dialog";

this.dialog = useService("dialog");

this.dialog.add(ConfirmationDialog, {
    title: _t("Cancel order"),
    body: _t("This will cancel order %s. Continue?", this.props.record.data.name),
    confirm: () => this.cancelOrder(),
    cancel: () => {},
});
```

La diferencia de fondo con un *toast* no es visual: un diálogo **espera una respuesta** y el *toast* no espera nada. Si tu aviso hace una pregunta, no es un *toast*.

## Los límites: donde esto no funciona

Es lo que se aprende por las malas, así que va explícito:

- **No funciona desde una tarea programada (cron).** Un cron no tiene navegador al que devolver una acción: nadie ha pulsado nada. Lo que se devuelva se descarta en silencio. Para avisar desde un cron: [chatter](Chatter-y-Seguidores.md), [actividad](Actividades.md), [correo](Plantillas-de-Correo.md) o bus (`bus.bus`).
- **No funciona desde un método llamado por otro método** que ignore su valor de retorno. Si `action_confirm` devuelve un aviso pero quien lo llama es un `_confirm_all()` que no propaga el retorno, el aviso desaparece. El diccionario tiene que llegar hasta el `return` que ve el cliente.
- **No sirve para avisar a otra persona.** La acción se devuelve por el mismo canal HTTP que trajo la petición: llega a quien pulsó el botón y a nadie más. Para avisar a otro usuario en el mismo instante, el bus (`bus.bus`).
- **No se puede mostrar dos avisos desde Python en una llamada.** Solo hay un valor de retorno. Si tienes dos cosas que decir, júntalas en un mensaje o encadena con `next`.
- **No sobrevive a una excepción.** Si después de construir el aviso lanzas un `UserError`, el cliente recibe el error, no el aviso.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| El botón funciona pero no sale ningún aviso | El método no **devuelve** el diccionario (falta el `return`, o quien lo llama descarta el retorno) |
| El aviso sale pero el asistente se queda abierto | Falta `'next': {'type': 'ir.actions.act_window_close'}` |
| El aviso sale amarillo y esperabas azul (o al revés) | No pasaste `type`: el defecto de Python es `info` y el del componente JS es `warning` |
| El HTML del mensaje se ve como texto literal | El mensaje se escapa a propósito. Para enlaces, usa `links` con `%s` |
| `%s` aparece tal cual en el aviso | Hay marcadores en el mensaje pero falta la lista `links`, o tiene menos elementos que marcadores |
| El botón `Undo` del aviso nunca llega a pulsarse | Falta `sticky: true`: el aviso se cierra a los 4 s |
| Nada aparece al lanzarlo desde un cron | Un cron no tiene cliente web. Cambia de mecanismo |
| El aviso de progreso se queda para siempre | No se llamó a la función de cierre que devolvió `add` |
| `TypeError: this.notification is undefined` | Falta `this.notification = useService("notification")` en el `setup()` |
| El mensaje sale en inglés a un usuario en catalán | El texto no pasó por `_()` en Python o `_t()` en JavaScript |

## Ejemplo completo: una acción de lote que informa del resultado

Junta casi todo lo anterior. Una acción sobre varios pedidos que procesa lo que puede, no aborta por los casos raros y resume al final:

```python
from odoo import _, models

class ShopOrder(models.Model):
    _inherit = 'shop.order'

    def action_ship_selected(self):
        """Ship every selected order that is ready, and report what happened."""
        shippable = self.filtered(lambda o: o.state == 'confirmed' and o.carrier_tracking_ref)
        blocked = self - shippable

        shippable.write({'state': 'shipped'})
        for order in shippable:
            order.message_post(body=_("Shipped with reference %s.", order.carrier_tracking_ref))

        if not shippable:
            params = {
                'type': 'warning',
                'title': _("Nothing shipped"),
                'message': _("None of the %s selected order(s) is ready to ship.", len(self)),
                'sticky': False,
            }
        elif blocked:
            params = {
                'type': 'warning',
                'title': _("Partially shipped"),
                'message': _("%(ok)s order(s) shipped, %(ko)s skipped for missing tracking reference.",
                             ok=len(shippable), ko=len(blocked)),
                'sticky': True,
            }
        else:
            params = {
                'type': 'success',
                'title': _("Shipped"),
                'message': _("%s order(s) shipped.", len(shippable)),
                'sticky': False,
            }

        return {'type': 'ir.actions.client', 'tag': 'display_notification', 'params': params}
```

Tres cosas que este ejemplo hace bien y que son el motivo de mostrarlo:

- **No lanza una excepción por los pedidos que no puede procesar.** Si lo hiciera, perdería también los que sí estaban listos, porque el `raise` deshace la transacción entera.
- **Distingue tres resultados**: nada, algo y todo. Un "Operación completada" cuando en realidad no se ha hecho nada es peor que no avisar.
- **Pone `sticky` solo en el caso parcial**, que es el único cuyo mensaje contiene información que hay que leer y actuar en consecuencia.
- **Deja el detalle donde persiste** (`message_post` en cada pedido) y el resumen donde es efímero (el *toast*).

## Buenas prácticas avanzadas

- **El caso "no se ha hecho nada" merece su propio aviso, y en `warning`.** Es el fallo de diseño más común en acciones de lote: devolver `success` sin comprobar si el filtro dejó el conjunto vacío. Quien pulsa "Enviar" y ve un aviso verde asume que se envió; volverá dentro de una semana a preguntar por qué el cliente no recibió nada. Contar los registros afectados y ramificar el mensaje cuesta cuatro líneas y ahorra esa conversación.
- **Un *toast* nunca es el único registro de una operación que importa.** Si mañana alguien puede preguntar "¿esto se envió?", el *toast* no responde. Escribe el detalle en el chatter o en `_logger` y usa el aviso solo como acuse de recibo visual. La combinación correcta es casi siempre "detalle persistente + resumen efímero", no una de las dos.
- **`sticky: True` en los `danger`, nunca en los `success`.** Un error que se autocierra en cuatro segundos es un error que nadie leerá, y el mensaje suele ser justo el que necesitaba copiar quien va a reportarlo. Al contrario, un "Guardado" pegajoso obliga a un clic de más cada vez. Este par de reglas se puede aplicar mecánicamente y mejora la interfaz de golpe.
- **Cuenta y nombra: el número de registros afectados va en el mensaje.** "3 order(s) shipped" es verificable de un vistazo; "Orders shipped" no. Y con un solo registro, el nombre concreto (`SO0042`) es mejor que el número: permite comprobar que se ha actuado sobre lo que se creía.
- **Si necesitas botones o cerrar el aviso desde el código, el aviso tiene que nacer en JavaScript.** `display_notification` no acepta `buttons` —los botones llevan una función `onClick`, y una función no viaja en un diccionario JSON—. Antes de intentar simularlo con `links`, valora si el patrón correcto no es directamente un asistente: si la acción ofrecida es importante, un enlace en un aviso que se va en cuatro segundos es un sitio pésimo para ponerla.
- **No metas datos sensibles en un aviso.** Es texto que puede quedarse en pantalla en un puesto compartido y que aparece en las capturas que la gente pega en los tickets. El importe del pedido está bien; el correo del cliente, el token de una API o el motivo médico de una baja, no.

## Documentación oficial

- [Odoo Developer Documentation — Actions](https://www.odoo.com/documentation/18.0/developer/reference/backend/actions.html) — la sección de `ir.actions.client` explica el mecanismo de acciones de cliente, del que `display_notification` es un caso concreto.
- [Odoo Developer Documentation — JavaScript services](https://www.odoo.com/documentation/18.0/developer/reference/frontend/services.html) — cómo funcionan los servicios del cliente web y cómo se obtienen con `useService`. Es la puerta de entrada a `notification`, `dialog` y `effect`.
- [Código de `notification_service.js`](https://github.com/odoo/odoo/blob/18.0/addons/web/static/src/core/notifications/notification_service.js) — noventa líneas que contienen la lista real de opciones aceptadas y el valor exacto del retardo de autocierre. Es la fuente a consultar cuando la documentación no menciona una opción.
- [Código de `client_actions.js`](https://github.com/odoo/odoo/blob/18.0/addons/web/static/src/webclient/actions/client_actions.js) — la implementación de `displayNotificationAction`: veinte líneas que muestran exactamente qué hace el cliente con cada clave de `params`, incluido el escapado de `links`.

## Recursos didácticos

- [Owl Playground](https://odoo.github.io/owl/playground) — el entorno interactivo del framework de componentes de Odoo. No trae los servicios de Odoo, pero es donde se entiende sin instalar nada qué es un componente, un `setup()` y un *hook*, que es lo que hace falta para leer los ejemplos de JavaScript de esta ficha.
- [Buscar `display_notification` en el código de Odoo](https://github.com/search?q=repo%3Aodoo%2Fodoo+%22display_notification%22+language%3APython&type=code) — decenas de usos reales. Mirar cómo lo resuelve un módulo estándar es más rápido que razonarlo desde cero, y aparecen variantes con `next` y `links` que no están en la documentación.

---

*En resumen: el aviso flotante es el acuse de recibo visual de Odoo —gratis desde Python con `display_notification`, con botones solo desde JavaScript— y su regla de oro es que nunca debe ser el único sitio donde consta lo que ha pasado.*
