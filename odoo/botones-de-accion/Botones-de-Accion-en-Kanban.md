# Botones de acción en Kanban

## ¿Qué es?

Un botón `type="object"` es un elemento de una vista de Odoo que, al pulsarlo, llama a un método Python del modelo sobre el registro concreto de esa tarjeta o fila. Es el mecanismo estándar para añadir una acción propia —marcar como favorito, archivar, iniciar un proceso— sin construir un formulario ni un asistente para algo que es, en realidad, una sola llamada.

## ¿Por qué existe?

Odoo genera solo, a partir del modelo, tres cosas: un formulario, un listado y una vista Kanban con los campos que definas. Eso cubre "ver y editar datos", pero no cubre "ejecutar una acción": archivar un pedido, confirmar una reserva, marcar una notificación como leída. Para eso hace falta un botón que dispare código, no que edite un campo.

La alternativa sería forzar todo a través de campos: por ejemplo, un `Selection` de estado que el usuario cambia a mano desde la propia tarjeta. Funciona para casos simples, pero no sirve en cuanto la acción tiene que comprobar algo antes de aplicarse (¿puede archivarse este pedido si tiene facturas pendientes?) o tiene que tocar más de un campo a la vez. El botón `type="object"` resuelve esto delegando la decisión al servidor: el clic llama a un método Python, y ese método decide qué pasa —incluyendo, si hace falta, negarse con un error.

> Si has trabajado con HTML y JavaScript, un botón `type="object"` es como un `<button>` con un `onClick` que hace una petición al servidor y espera su respuesta antes de decidir qué mostrar: la diferencia es que aquí la petición y la respuesta las gestiona el framework, y tú solo escribes el método que se ejecuta en el servidor.

## ¿Cuándo y para qué se usa?

Cualquier acción que aplique a un registro concreto y no encaje en "editar un campo": archivar, duplicar, confirmar, enviar, marcar como favorito, iniciar o detener un proceso. En una vista Kanban aparece casi siempre en dos sitios: en el pie de la tarjeta (junto a la prioridad o la fecha límite) o como un icono suelto en la esquina.

El ejemplo que recorre esta guía es un cronómetro de trabajo en las tarjetas de una lista de tareas: cada tarjeta tiene un botón de reproducir que, al pulsarlo, arranca un cronómetro para esa tarea concreta y se convierte en un botón de detener mientras el cronómetro sigue en marcha. Es un caso típico de botón `type="object"`: una sola llamada, con una condición (¿está ya en marcha?) que decide qué icono y qué texto mostrar.

---

## Extender una vista existente: `inherit_id` y `xpath`

Las vistas de Odoo casi nunca se escriben de cero: se **heredan**. Si quieres añadir el botón del cronómetro a la vista Kanban de tareas que ya trae Odoo, no la reescribes entera —eso rompería cualquier otro módulo que también la haya extendido—, sino que declaras una vista nueva que apunta a la original con `inherit_id` y describe **solo el cambio**.

```xml
<record id="view_task_kanban_inherit_timer" model="ir.ui.view">
    <field name="name">task.kanban.inherit.timer</field>
    <field name="model">project.task</field>
    <field name="inherit_id" ref="project.view_task_kanban"/>
    <field name="arch" type="xml">
        <xpath expr="//footer//field[@name='priority']" position="after">
            <!-- lo nuevo va aquí -->
        </xpath>
    </field>
</record>
```

`inherit_id` señala la vista base por su ID externo (`módulo.id_de_la_vista`). Dentro de `arch`, en vez de repetir toda la estructura, usas `xpath` para decir **dónde** insertar: `expr` es una ruta al elemento de referencia (aquí, el `<field name="priority">` que hay dentro del `<footer>` de la tarjeta) y `position` dice qué hacer respecto a él:

| `position` | Efecto |
|---|---|
| `after` | Inserta el contenido nuevo justo después del elemento de referencia |
| `before` | Lo inserta justo antes |
| `inside` | Lo mete dentro, al final de sus hijos |
| `replace` | Sustituye el elemento de referencia entero |
| `attributes` | No añade nodos: modifica atributos del elemento de referencia (con `<attribute name="...">`) |

`expr` acepta cualquier expresión XPath, pero en la práctica casi siempre son rutas cortas como `//field[@name='priority']` (busca ese campo en cualquier parte del árbol) o `//footer//field[@name='priority']` (lo mismo, pero solo dentro de un `<footer>`). Cuanto más específica la ruta, menos probable que choque con lo que añada otro módulo.

## Declarar campos para poder usarlos, aunque no se vean

Antes de poder mostrar u ocultar el botón según si el cronómetro está en marcha, la tarjeta necesita ese dato. Y aquí hay un matiz que sorprende la primera vez: **una vista Kanban solo recibe del servidor los campos que aparecen declarados en su `arch`**. Si el campo que necesitas no está ya ahí, tienes que añadirlo tú, aunque no se vaya a ver en pantalla.

Las vistas Kanban de Odoo declaran una lista de campos "a buscar" antes del bloque de plantilla, algo así:

```xml
<kanban>
    <field name="stage_id"/>
    <field name="priority"/>
    <templates>
        <t t-name="kanban-box">
            <!-- diseño de la tarjeta -->
        </t>
    </templates>
</kanban>
```

Esos `<field>` sueltos no dibujan nada: solo le dicen al cliente "trae también este valor para cada tarjeta". Para que el campo booleano que indica si el cronómetro está en marcha (llamémosle `is_timer_running`, un campo calculado que no se guarda en la base de datos) llegue al navegador, tu vista heredada tiene que añadirlo a esa misma lista:

```xml
<field name="stage_id" position="after">
    <field name="is_timer_running"/>
</field>
```

Sin esta línea, cualquier expresión que use `is_timer_running` en el botón —lo que viene en la siguiente sección— se evaluaría siempre igual (normalmente como si el campo no existiese) porque el dato nunca llegó al cliente. Es un error silencioso: la vista carga sin fallos, el botón simplemente no reacciona.

## El botón `type="object"`: anatomía

Con el campo ya disponible, el botón en sí:

```xml
<xpath expr="//footer//field[@name='priority']" position="after">
    <button name="action_toggle_timer"
            type="object"
            title="Start Timer"
            class="btn btn-sm btn-link p-0"
            invisible="is_timer_running">
        <i class="fa fa-play"/>
    </button>
    <button name="action_toggle_timer"
            type="object"
            title="Stop Timer"
            class="btn btn-sm btn-link p-0 text-danger"
            invisible="not is_timer_running">
        <i class="fa fa-stop"/>
    </button>
</xpath>
```

Los atributos que importan:

| Atributo | Para qué sirve |
|---|---|
| `name` | Qué llamar: el nombre del método (si `type="object"`) o el external ID de una acción (si `type="action"`) |
| `type` | Cómo interpretar `name` — ver la tabla siguiente |
| `string` / contenido | El texto o icono del botón. Aquí se usa un `<i class="fa ...">` en vez de texto |
| `class` | Clases CSS normales de Bootstrap/Odoo (`btn-link`, `btn-sm`, `text-danger`...) |
| `invisible` | Expresión que oculta el botón cuando es verdadera (ver la sección siguiente) |
| `confirm` | Si se pone, muestra un diálogo de confirmación con ese texto antes de ejecutar |
| `groups` | Restringe el botón a uno o varios grupos de seguridad |
| `context` | Contexto adicional que se añade a la llamada |

Nota algo importante en el ejemplo: son **dos botones**, no uno que cambia de icono. Cada uno tiene su propia condición `invisible` opuesta a la del otro, así que en cualquier momento se ve exactamente uno de los dos. Es el patrón habitual en Odoo para "un control con dos estados visuales": más simple de leer en el XML que un único botón con un icono calculado dinámicamente.

### `object`, `action` y sus parientes

`type="object"` no es la única forma de que un botón haga algo. Convive con otras dos, y confundirlas es un error de principiante frecuente:

| `type` | Qué espera en `name` | Qué ocurre al pulsar |
|---|---|---|
| `object` | Un método Python del modelo | Llama al método sobre el registro; puede devolver una acción (ver más abajo) |
| `action` | El external ID (o el ID numérico) de una `ir.actions.*` | Ejecuta esa acción directamente — abre otra vista, lanza un servidor, etc. — sin pasar por un método intermedio |
| `edit` | (no aplica) | Abre el registro en modo edición, sin llamar a nada |

Y hay un tercer patrón que se confunde con estos porque también es "un botón sobre un registro" pero vive en el **formulario**, no en la tarjeta: el botón inteligente (`<button class="oe_stat_button" type="object" icon="fa-tasks">`), que técnicamente es un botón `type="object"` como cualquier otro, solo que con una clase CSS que lo dibuja como una cajita con un número dentro. Mismo mecanismo, presentación distinta.

## Mostrar y ocultar: el atributo `invisible`

`invisible="is_timer_running"` no es una plantilla de texto ni una llamada a Python: es una expresión que el propio cliente evalúa, en el navegador, contra los datos de esa tarjeta. Puede ser tan simple como el nombre de un campo booleano (verdadero = oculto) o una expresión más elaborada:

```xml
<button ... invisible="is_timer_running or not can_track_time"/>
<button ... invisible="stage_id.name == 'Done'"/>
```

Estas expresiones solo pueden usar campos que la vista ya tenga disponibles —de ahí la sección anterior— y un subconjunto de operadores Python (`and`, `or`, `not`, comparaciones). No hay acceso a métodos arbitrarios: si la condición necesita lógica de negocio real, la forma correcta es calcularla en un campo `compute` en Python y poner ahí el nombre del campo, no intentar meter la lógica dentro del propio `invisible`.

## Qué recibe el método, y qué puede devolver

Cuando se pulsa el botón de una tarjeta, el cliente llama al método pasándole **el ID de esa tarjeta y solo ese**, nunca una selección de varias. Por eso el patrón habitual es:

```python
def action_toggle_timer(self):
    self.ensure_one()
    # ... arrancar o parar el cronómetro de self ...
```

`self.ensure_one()` no lo pone Odoo por ti de forma automática: lo escribes tú, como comprobación explícita. Pero en un botón de tarjeta Kanban es casi una formalidad, porque el propio mecanismo de clic ya garantiza que `self` trae un único registro. Su verdadero valor aparece si ese mismo método se reutiliza desde otro sitio —una llamada RPC externa, un cron, un botón de un list view con selección múltiple—: ahí si `self` llega con más de un registro, `ensure_one()` falla con un mensaje claro en el momento exacto donde la suposición se rompe, en vez de fallar más adelante con un error confuso.

El valor de retorno también importa:

- **No devolver nada** (o devolver `None`/`False`): el cliente lo interpreta como "no hay ninguna acción que ejecutar además de la llamada". Es el caso normal para un simple cambio de estado como arrancar o parar un cronómetro.
- **Devolver un diccionario de acción**: el cliente lo ejecuta como si fuera el resultado de cualquier otra acción. Es lo que se usa para abrir un formulario relacionado, o para mostrar un aviso flotante de confirmación:

  ```python
  def action_archive_and_notify(self):
      self.ensure_one()
      self.active = False
      return {
          'type': 'ir.actions.client',
          'tag': 'display_notification',
          'params': {'type': 'success', 'message': "Task archived."},
      }
  ```

En cualquiera de los dos casos, tras la llamada la vista **recarga el registro de esa tarjeta** de forma automática — es lo que hace que el botón de reproducir se convierta en botón de detener sin que tengas que escribir nada para refrescar la tarjeta. Esa recarga automática solo ocurre porque el cambio llegó *a través del propio botón*; si el mismo cambio de estado se dispara desde otro sitio (otro componente de la interfaz, otro proceso), esa tarjeta no se entera sola y hace falta provocar el refresco por otra vía — normalmente, el bus de eventos del cliente.

---

## Buenas prácticas avanzadas

- **Declara siempre los campos que usan tus condiciones `invisible`, aunque parezca que "ya deberían estar".** Es el error más habitual con este patrón: el botón no aparece o no desaparece nunca, no porque la lógica esté mal, sino porque el campo nunca llegó al cliente. Antes de depurar la condición, comprueba con las herramientas de desarrollador que el campo viene en los datos de la tarjeta.
- **Prefiere dos botones con condiciones opuestas a un solo botón con estado calculado en el XML.** Es más código, pero cada botón tiene su propio icono, su propio texto y su propia clase CSS sin necesitar expresiones condicionales dentro de esos atributos. Se lee mejor y se depura mejor.
- **Usa `xpath` con la ruta más específica que puedas, no la más corta.** `//field[@name='priority']` funciona, pero si dos módulos distintos anclan ahí su propio contenido con `position="after"`, el orden final depende del orden de carga de los módulos —no de lo que tú decidiste—. Anclar sobre un elemento único dentro de una sección conocida (`//footer//field[@name='priority']`) reduce ese riesgo, aunque no lo elimina del todo.
- **No metas lógica de negocio en el método del botón sin protegerla también en otro sitio.** Un método llamado `action_toggle_timer` solo se ejecuta cuando alguien pulsa ese botón concreto; si la misma transición de estado tiene que ser válida siempre —también desde una importación o desde otro módulo—, la validación real va en el modelo (por ejemplo, en una restricción), y el método del botón se limita a ser una de las varias puertas de entrada a esa lógica.
- **Cuando el método pueda tardar o fallar, decide explícitamente qué ve la persona.** Si el método puede lanzar una excepción, ese mensaje aparece tal cual en un diálogo; si quieres confirmar el éxito, la vía correcta es devolver un `display_notification`, no dar el cambio por "obviamente visible" solo porque la tarjeta se recargue.

## Documentación oficial

- [Odoo Developer Documentation — Kanban Views](https://www.odoo.com/documentation/18.0/developer/reference/user_interface/view_architectures.html#kanban) — la referencia completa de la arquitectura Kanban: qué elementos acepta, cómo se declaran los campos y las plantillas.
- [Odoo Developer Documentation — View Inheritance](https://www.odoo.com/documentation/18.0/developer/reference/user_interface/view_architectures.html#inheritance) — la sintaxis completa de `xpath`, sus posiciones y las alternativas (`position="attributes"`, herencia por prioridad).

## Recursos didácticos

- [Modo desarrollador de Odoo, panel "Edit View: Kanban"](https://www.odoo.com/documentation/18.0/applications/general/developer_mode.html) — con el modo desarrollador activo, el menú de depuración de cualquier vista Kanban muestra el XML ya combinado con todas las herencias aplicadas. Es la forma más rápida de ver qué `xpath` heredado ha caído dónde, sin tener que reconstruirlo a mano.

---

*En resumen: un botón `type="object"` es una llamada a un método de Python disfrazada de icono en una tarjeta — y para que reaccione a los datos del registro, esos datos tienen que estar ya declarados en la vista, se vean o no.*
