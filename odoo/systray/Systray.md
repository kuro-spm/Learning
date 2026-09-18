# Systray

## ¿Qué es?

El *systray* es la franja de iconos en la esquina superior derecha del backend de Odoo, junto al selector de apps, el buscador y el avatar del usuario. Cada icono que ves ahí —la campana de actividades, el reloj de fichaje, el globo de mensajes— es un pequeño componente independiente que un módulo ha "enchufado" a esa franja, y que se ve **desde cualquier vista**, sin importar qué pantalla tengas abierta.

## ¿Por qué existe?

Hay información que no pertenece a ningún registro ni a ninguna vista concreta: pertenece al **usuario que ha iniciado sesión**, ahora mismo, esté mirando lo que esté mirando. "¿Tengo actividades pendientes?", "¿estoy fichado?", "¿tengo algo corriendo en segundo plano?" son preguntas sobre la persona, no sobre el pedido o la tarea que tiene abiertos en ese momento.

Si esa información viviera dentro de cada vista, habría que repetirla en todas partes y desaparecería en cuanto cambiaras de pantalla. El systray resuelve eso poniéndola en un sitio que **se monta una vez, al arrancar el cliente web, y sobrevive a toda la navegación posterior**: cambies de app, abras un formulario o vuelvas al listado, el icono sigue ahí con su propio estado.

> Si conoces la bandeja del sistema de Windows o macOS —wifi, batería, notificaciones, siempre visibles sin importar qué programa tengas abierto—, el systray de Odoo es exactamente esa idea, aplicada a una aplicación web.

## ¿Cuándo y para qué se usa?

Para cualquier control o indicador ligado al usuario actual que deba estar disponible en todo momento, no solo mientras mira un registro concreto:

- Un contador de notificaciones o actividades pendientes.
- Un reloj de fichaje (entrada/salida) que se pueda accionar sin ir a ninguna pantalla concreta.
- Un indicador de "tengo algo corriendo": una exportación en marcha, una sincronización, un cronómetro de una app de seguimiento de tiempo.
- Un selector rápido de idioma o de tema visual.

Y **no** se usa para nada ligado a un registro concreto que se esté viendo —eso va en la vista o el formulario de ese registro— ni para una acción puntual de un flujo de trabajo —eso es un botón normal.

A lo largo de la guía usamos un ejemplo recurrente: una app de gestión de tareas donde cualquiera puede arrancar y parar un cronómetro sobre una tarea, y queremos un icono en el systray que muestre los cronómetros que la persona tiene corriendo ahora mismo, con un botón para pararlos.

---

## El registro `systray`: enchufar un icono a la barra

Odoo organiza buena parte de su frontend con **registries**: listas con nombre donde cualquier módulo puede añadir una entrada, sin tener que modificar el código de quien las lee. El systray es una de esas listas. Añadir un icono es registrar un componente en ella:

```javascript
/* @odoo-module */

import { registry } from "@web/core/registry";
import { TaskTimersMenu } from "./task_timers_menu";

registry
    .category("systray")
    .add("task_manager.task_timers_menu", { Component: TaskTimersMenu }, { sequence: 20 });
```

En cuanto este archivo se carga en el navegador, el icono aparece en la barra sin que haya que tocar ningún otro fichero: la barra de navegación simplemente recorre la categoría `"systray"` y pinta un componente por cada entrada.

`sequence` decide el orden **relativo** frente a los iconos de otros módulos: la barra los ordena de menor a mayor y luego los coloca de derecha a izquierda, así que un `sequence` alto queda más a la izquierda dentro del grupo de iconos, y uno bajo, más cerca del avatar del usuario. No hace falta acertar un número exacto: basta con elegir uno que te deje razonablemente cerca de los iconos junto a los que tiene sentido aparecer.

## Anatomía de un componente systray

Un componente systray es una clase de **Owl** (el framework de componentes de Odoo) como cualquier otra que puedas usar en un formulario o un widget de lista; lo único que lo distingue es que se registra en `"systray"` en lugar de en un campo o una vista. La versión mínima que funciona:

```javascript
/* @odoo-module */

import { Component } from "@odoo/owl";

export class TaskTimersMenu extends Component {
    static template = "task_manager.task_timers_menu";
    static props = [];
}
```

```xml
<templates xml:space="preserve">
    <t t-name="task_manager.task_timers_menu">
        <div class="o_task_timers_menu">
            <i class="fa fa-clock-o" role="img" aria-label="Cronómetros"/>
        </div>
    </t>
</templates>
```

Tres detalles de esta clase que no son opcionales:

- **`static template`** apunta al nombre `t-name` de la plantilla XML, no a una ruta de fichero. Owl busca ese nombre entre todas las plantillas cargadas, así que si los dos textos no coinciden letra a letra, el componente revienta al montarse con un error de "template not found".
- **`static props = []`** declara que este componente no espera recibir ninguna propiedad. Owl, en modo desarrollo, valida las props recibidas contra esta lista; un componente systray no recibe nada del padre (la barra de navegación solo lo instancia), así que la lista vacía es lo normal.
- El icono es simplemente una etiqueta `<i>` de Font Awesome, la misma librería de iconos que usa el resto del backend de Odoo.

Esto ya se registra y se ve, pero no hace nada: para que muestre información real hace falta estado, y para tener estado hace falta un `setup()`.

## De dónde salen los datos: el servicio `orm` o un controlador propio

Un componente systray necesita traer del servidor la información del usuario actual apenas se monta. Hay dos caminos, y cuál usar depende de qué tan simple sea la consulta.

**Camino simple: el servicio `orm`.** Si lo único que hace falta es leer o escribir registros con las operaciones normales del ORM (`search`, `read`, `write`, llamar a un método), no hace falta escribir ni una línea de Python nueva:

```javascript
import { Component, useState } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";

export class TaskTimersMenu extends Component {
    static template = "task_manager.task_timers_menu";
    static props = [];

    setup() {
        this.orm = useService("orm");
        this.state = useState({ timers: [] });
        this.loadTimers();
    }

    async loadTimers() {
        this.state.timers = await this.orm.searchRead(
            "task.timer",
            [["user_id", "=", this.env.services.user.userId], ["running", "=", true]],
            ["task_id", "task_name"]
        );
    }
}
```

`useService("orm")` es el mismo patrón que usarías en cualquier otro componente del backend: te da acceso a los métodos del ORM desde JavaScript, con permisos y reglas de acceso aplicados igual que si la llamada viniera de una vista.

**Camino con controlador propio:** cuando la operación no es un simple `read` o `write`, sino una regla de negocio con varios pasos —por ejemplo, "para el cronómetro de esta tarea y recalcula el resto de cosas que dependen de él"—, conviene dejar esa lógica en el servidor detrás de una ruta propia, en lugar de repetirla en JavaScript:

```python
from odoo import http
from odoo.http import request


class TaskManagerController(http.Controller):
    @http.route("/task_manager/running_timers", type="json", auth="user", readonly=True)
    def running_timers(self):
        timers = request.env["task.timer"].search([
            ("user_id", "=", request.env.uid),
            ("running", "=", True),
        ])
        return [{"task_id": t.task_id.id, "task_name": t.task_id.name} for t in timers]

    @http.route("/task_manager/stop_timer", type="json", auth="user")
    def stop_timer(self, task_id):
        timer = request.env["task.timer"].search([
            ("user_id", "=", request.env.uid),
            ("task_id", "=", task_id),
            ("running", "=", True),
        ], limit=1)
        timer.action_stop()  # aquí vive la lógica de negocio real
        return True
```

```javascript
import { rpc } from "@web/core/network/rpc";

async loadTimers() {
    this.state.timers = await rpc("/task_manager/running_timers");
}
```

`type="json"` marca la ruta como pensada para ser llamada desde JavaScript (recibe y devuelve JSON en lugar de renderizar una página), y `auth="user"` exige una sesión iniciada — exactamente lo que necesita un icono que solo tiene sentido dentro del backend. `readonly=True` en la primera ruta le dice a Odoo que esta llamada no escribe nada, lo que le permite servirla desde una réplica de lectura si el despliegue tiene una configurada.

La regla práctica: `useService("orm")` para leer y escribir registros tal cual; un controlador propio cuando hay una operación de negocio que agrupa varios pasos y no quieres duplicarla en el cliente.

## Icono dinámico y contador: plantar el estado en la plantilla

El objeto que pasas a `useState` es reactivo: en cuanto una propiedad cambia, Owl vuelve a renderizar la parte de la plantilla que la usa, sin que tengas que pedirlo explícitamente. Aprovecharlo para que el icono cambie de color y muestre un contador es solo cuestión de leer ese estado en el XML:

```xml
<t t-name="task_manager.task_timers_menu">
    <div class="o_task_timers_menu">
        <i class="fa fa-clock-o"
           t-attf-class="text-{{ state.timers.length ? 'success' : 'muted' }}"
           role="img" aria-label="Cronómetros"/>
        <span t-if="state.timers.length"
              class="badge rounded-pill text-bg-success"
              t-esc="state.timers.length"/>
    </div>
</t>
```

`t-attf-class` construye la clase interpolando una expresión (aquí, verde si hay cronómetros en marcha, gris si no) y `t-esc` imprime el número de cronómetros como texto. En cuanto `loadTimers()` reasigna `this.state.timers`, este fragmento se repinta solo — no hace falta un `render()` manual ni un evento de cambio.

Para que el icono despliegue una lista al pulsarlo, Odoo trae un componente `Dropdown` ya hecho, que se usa como cualquier otro componente hijo:

```javascript
import { Dropdown } from "@web/core/dropdown/dropdown";

export class TaskTimersMenu extends Component {
    static template = "task_manager.task_timers_menu";
    static components = { Dropdown };
    static props = [];
    // ...setup() igual que antes
}
```

```xml
<t t-name="task_manager.task_timers_menu">
    <Dropdown position="'bottom-end'" beforeOpen.bind="loadTimers">
        <i class="fa fa-clock-o" t-attf-class="text-{{ state.timers.length ? 'success' : 'muted' }}"/>
        <t t-set-slot="content">
            <div t-foreach="state.timers" t-as="timer" t-key="timer.task_id" t-esc="timer.task_name"/>
        </t>
    </Dropdown>
</t>
```

`beforeOpen.bind="loadTimers"` es el detalle que evita el error más típico de este patrón: sin él, el desplegable mostraría los datos que había en el momento de montar el componente —al principio de la sesión—, no los datos actuales. Volviendo a cargar justo antes de abrirse, cada apertura refleja el estado real.

## Interactuar desde el desplegable: acciones y clics dobles

Un icono que solo informa es útil, pero la mayoría también deja actuar desde ahí mismo. Dos servicios cubren los dos casos habituales.

**Abrir un registro con el servicio `action`:**

```javascript
setup() {
    this.action = useService("action");
    // ...
}

openTask(taskId) {
    this.action.doAction({
        type: "ir.actions.act_window",
        res_model: "task.task",
        res_id: taskId,
        views: [[false, "form"]],
    });
}
```

`doAction` acepta la misma clase de diccionario de acción que usarías desde un botón de vista o desde Python; aquí simplemente lo disparas desde código en vez de recibirlo como valor de retorno.

**Evitar el doble clic con `useDebounced`:** un botón dentro de un dropdown que dispara una llamada al servidor es candidato perfecto a un doble clic accidental —el desplegable tarda un instante en reaccionar y es fácil pulsar dos veces—. `useDebounced` envuelve la función y absorbe esas repeticiones:

```javascript
import { useDebounced } from "@web/core/utils/timing";

setup() {
    // ...
    this.onClickStop = useDebounced(this.stopTimer, 200);
}

async stopTimer(taskId) {
    await rpc("/task_manager/stop_timer", { task_id: taskId });
    await this.loadTimers();
}
```

```xml
<button t-on-click="() => this.onClickStop(taskId)">Parar</button>
```

Con el retardo por defecto (sin más opciones), `useDebounced` deja pasar la llamada solo cuando han transcurrido los milisegundos indicados desde el último clic — el patrón estándar de "espera a que la persona deje de pulsar". El segundo parámetro admite además un objeto de opciones (`{ immediate: true }` para ejecutar en el primer clic y descartar los siguientes durante el retardo, en vez de al revés); usa siempre ese objeto explícito si necesitas ese matiz, no un valor suelto — una llamada como `useDebounced(fn, 200, true)` no lanza ningún error, pero tampoco activa nada: un booleano no tiene la propiedad `immediate` que la función busca, así que se queda con el comportamiento por defecto sin avisar de que lo has pedido mal.

## Registrar los ficheros en el manifest

Nada de esto se carga solo: el módulo tiene que declarar el JavaScript y el XML en el *bundle* de assets del backend, dentro de `__manifest__.py`:

```python
{
    "name": "Task Manager",
    "depends": ["web"],
    "data": [
        # vistas, seguridad...
    ],
    "assets": {
        "web.assets_backend": [
            "task_manager/static/src/components/task_timers_menu/task_timers_menu.js",
            "task_manager/static/src/components/task_timers_menu/task_timers_menu.xml",
        ],
    },
}
```

`web.assets_backend` es el *bundle* que Odoo carga en todas las pantallas del backend (a diferencia de `web.assets_frontend`, para el sitio público, o `web.assets_tests`, para los tests). Cualquier fichero que quede fuera de esta lista, sencillamente no llega al navegador, por correcto que esté su código.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| El icono no aparece nunca | El JS y el XML no están en la lista de `web.assets_backend` del manifest |
| El icono aparecía y, tras un cambio de código, sigue igual | El navegador sirvió el *bundle* de assets desde caché; hace falta recargar sin caché o reiniciar el servidor tras un cambio de manifest |
| `Error: Cannot find the definition of template "..."` | El valor de `static template` no coincide exactamente con el `t-name` del XML (incluida mayúscula/minúscula) |
| Warning de Owl sobre una prop desconocida | Falta `static props = []` (o no incluye una prop que sí se está pasando) |
| El desplegable muestra datos viejos la segunda vez que se abre | Falta `beforeOpen.bind` apuntando a la función que recarga los datos |
| Nada ocurre al pulsar el botón de acción, sin error visible | La ruta del controlador no coincide, o le falta `auth="user"` y la sesión no tiene permiso |

## Buenas prácticas avanzadas

- **No esperes a los datos para montar el componente.** Llama a `loadTimers()` en `setup()` sin `await`: si el `setup()` completo esperara la respuesta del servidor, todo el arranque del cliente web se retrasaría por un solo icono. El patrón correcto es "lanzar la carga y dejar que el estado reactivo actualice la plantilla cuando llegue", no bloquear el montaje.
- **Usa `isDisplayed` para ocultar el icono entero, no una condición dentro de la plantilla que deje un hueco vacío.** El objeto que registras en la categoría admite una función `isDisplayed(env)`; si el icono no aplica a este usuario (por ejemplo, no tiene ningún cronómetro nunca), devuélvela en `false` en vez de renderizar un `<div>` vacío que ocupa espacio y complica el CSS del resto de iconos.
- **El texto siempre en inglés en el código, y traducido con `_t`, no con una cadena literal.** Cualquier texto visible en la plantilla o el JavaScript (`_t("Stop")`, no `"Detener"` a pelo) sigue el mismo mecanismo de traducción que el resto del backend; escribirlo ya en el idioma del usuario rompe ese mecanismo para todos los demás idiomas.
- **Un `sequence` "bonito" (números redondos, huecos de 10 en 10) facilita que otro módulo se intercale sin tener que renumerar nada.** Los sequences de los iconos estándar de Odoo dejan huecos deliberados por esta misma razón.
- **La llamada que dispara la acción del botón conviene que sea idempotente o esté protegida contra doble ejecución** (con `useDebounced`, o comprobando el estado antes de actuar): a diferencia de un botón dentro de un formulario, aquí no hay una fila con estado de carga que impida pulsar dos veces mientras la primera llamada sigue en vuelo.

## Documentación oficial

- [Odoo Developer Documentation — JavaScript services](https://www.odoo.com/documentation/18.0/developer/reference/frontend/services.html) — cómo funcionan los servicios del cliente (`orm`, `action`, `notification`...) y cómo se obtienen con `useService`, la base de todo lo que hace un componente systray.
- [Código de `navbar.js`](https://github.com/odoo/odoo/blob/18.0/addons/web/static/src/webclient/navbar/navbar.js) — cómo la barra de navegación lee la categoría `"systray"`, aplica `isDisplayed` y ordena los iconos por `sequence`; la fuente exacta si el comportamiento no cuadra con lo esperado.
- [Código de `attendance_menu.js` (hr_attendance)](https://github.com/odoo/odoo/blob/18.0/addons/hr_attendance/static/src/components/attendance_menu/attendance_menu.js) — un systray real y completo de un módulo estándar de Odoo: carga de datos, `Dropdown`, `useDebounced` y una acción con geolocalización incluida.

## Recursos didácticos

- [Owl Playground](https://odoo.github.io/owl/playground) — el entorno interactivo del framework de componentes de Odoo. No trae los servicios propios de Odoo (`useService`, `rpc`...), pero es donde se entiende sin instalar nada qué es un componente, un `setup()`, un *hook* y el estado reactivo de `useState` — justo lo que hace falta para seguir esta guía.

---

*En resumen: el systray es la bandeja de iconos que vive por encima de cualquier vista de Odoo — un componente Owl más, solo que enchufado a un registro global en vez de a una pantalla concreta, con estado reactivo, sus propios datos y sus propias acciones.*
