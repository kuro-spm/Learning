# El bus de eventos de OWL (`env.bus`)

## ¿Qué es?

`env.bus` es un canal de eventos compartido por todos los componentes del cliente web de Odoo: cualquiera de ellos puede disparar un evento con `bus.trigger(nombre, datos)` y cualquier otro puede escucharlo, sin que uno conozca al otro ni exista una relación padre-hijo entre ambos.

## ¿Por qué existe?

En OWL (el framework de componentes que usa el cliente web de Odoo desde la versión 15), cada vista mantiene su propio estado en memoria: una vista *kanban* ya abierta ha leído sus registros una vez y los conserva en el navegador hasta que algo le dice que los vuelva a leer. Mientras el cambio se produce **dentro** de esa misma vista —por ejemplo, un botón de la propia tarjeta que llama a un método del modelo—, Odoo ya se ocupa de recargar el registro después de la llamada: es el comportamiento estándar de un botón de tipo `object` en una vista.

El problema aparece cuando el cambio llega **desde fuera**: un widget de la barra de navegación, un diálogo independiente, otra parte de la interfaz que no forma parte del árbol de componentes de esa vista. Ese widget puede escribir perfectamente en la base de datos, pero no tiene ninguna referencia a la vista abierta para decirle "tus datos ya no valen, vuelve a leerlos". Sin esa referencia, el registro se queda obsoleto en pantalla hasta que algo fuerza una recarga completa (cambiar de pestaña del *breadcrumb*, refrescar el navegador).

`env.bus` resuelve exactamente esa comunicación entre desconocidos. En vez de que el widget busque una referencia a la vista (que en general no puede tener), dispara un evento genérico al aire; la vista, si le interesa, se apunta a escucharlo. Ninguno de los dos necesita saber que el otro existe.

> Si has usado el patrón *publish/subscribe* en algún otro sitio —un `EventEmitter` de Node, el `DOM` con `dispatchEvent`/`addEventListener`, o un *message bus* interno de una aplicación de escritorio—, `env.bus` es exactamente eso: un punto de encuentro neutral entre quien avisa y quien escucha.

## ¿Cuándo y para qué se usa?

El caso de uso es siempre el mismo patrón: **una acción que ocurre en un sitio de la interfaz debe reflejarse en otro sitio con el que no hay relación de componentes**. Algunos ejemplos habituales en un cliente web de Odoo:

- Un icono de la barra de navegación (un *systray*) permite completar una acción rápida sobre un registro —marcarlo como favorito, archivarlo, cerrarlo— y una lista o un *kanban* de ese mismo modelo, si está abierta en esa pestaña, debe dejar de mostrar el estado antiguo.
- Un diálogo modal cambia una configuración global y varios widgets de la pantalla de fondo necesitan enterarse al cerrarse el diálogo.
- Un servicio en segundo plano (por ejemplo, uno que sincroniza algo cada pocos segundos) necesita avisar a quien esté interesado sin acoplarse a qué componente es.

Lo que **no** hace falta resolver con `env.bus` es el caso contrario: un botón que vive dentro de la propia vista y llama a un método del modelo con `type="object"`. Ahí Odoo ya recarga el registro automáticamente al terminar la llamada, porque el propio framework de vistas sabe qué registro se ha tocado. `env.bus` entra en juego precisamente cuando esa relación directa no existe.

Tampoco resuelve la comunicación entre pestañas distintas del navegador ni entre usuarios distintos — de eso se habla al final de esta guía.

## `env.bus`: disparar y escuchar eventos

`env.bus` es una instancia de `EventBus`, una clase de OWL que extiende el `EventTarget` estándar del navegador (el mismo que usan `document` o cualquier elemento del DOM) y le añade un método de conveniencia:

```js
class EventBus extends EventTarget {
    trigger(name, payload) {
        this.dispatchEvent(new CustomEvent(name, { detail: payload }));
    }
}
```

`trigger(nombre, payload)` no es más que azúcar sintáctico sobre `dispatchEvent`: crea un [`CustomEvent`](https://developer.mozilla.org/es/docs/Web/API/CustomEvent) con ese nombre y mete el `payload` en su propiedad `detail`. Cualquier componente con acceso a `this.env` puede dispararlo:

```js
// Desde un widget de la barra de navegación, tras guardar el cambio en el servidor
this.env.bus.trigger("task:starred_changed", { taskId: 42, starred: true });
```

Y cualquier otro puede escucharlo con la API estándar del DOM:

```js
this.env.bus.addEventListener("task:starred_changed", (ev) => {
    const { taskId, starred } = ev.detail;
    console.log(`La tarea ${taskId} ahora está ${starred ? "marcada" : "desmarcada"}`);
});
```

Dos detalles que conviene tener claros desde el principio:

- **El nombre del evento es una cadena libre, no una lista cerrada.** Odoo no reserva un catálogo de eventos de `env.bus`; cada funcionalidad inventa el nombre que necesita. Elige uno específico (`"task:starred_changed"`, no `"changed"`) para no chocar por casualidad con el de otra parte del sistema.
- **`env.bus` vive mientras vive la sesión del cliente web**, no la vista concreta. Es el mismo objeto para toda la aplicación en esa pestaña, así que un evento disparado desde cualquier rincón de la interfaz llega a cualquier otro rincón que esté escuchando en ese momento — y a nadie más: no sale de esa pestaña del navegador.

## El hook `useBus`: suscribirse sin fugas de memoria

Escuchar con `addEventListener` a pelo tiene una trampa: si el componente que escucha se destruye (el usuario navega a otra pantalla) y nadie quita ese listener, sigue vivo, sigue reaccionando a eventos y el componente entero no puede liberarse de memoria. Es una fuga clásica.

Para evitarla, el núcleo del cliente web ofrece el hook `useBus`, importado de `@web/core/utils/hooks`:

```js
import { useBus } from "@web/core/utils/hooks";

// dentro del setup() de un componente OWL
useBus(this.env.bus, "task:starred_changed", (ev) => {
    const { taskId, starred } = ev.detail;
    this.refreshIfNeeded(taskId, starred);
});
```

`useBus` hace justo dos cosas: engancha el listener cuando el componente se monta, y lo quita automáticamente cuando se destruye. El componente que escucha nunca tiene que acordarse de limpiar nada — por eso es la forma recomendada de suscribirse desde un componente, y `addEventListener` a pelo queda para el código que no vive dentro del ciclo de vida de un componente (por ejemplo, un servicio).

## Engancharse a una vista que no es tuya: `patch()`

Hasta aquí, disparar y escuchar un evento es sencillo si controlas los dos componentes. El problema práctico es distinto: normalmente quieres que **una vista que ya existe en Odoo** (una vista *kanban*, un formulario) reaccione a tu evento, y esa vista no tiene ningún hueco pensado para que añadas código dentro de su `setup()`.

La solución del cliente web es `patch()`, de `@web/core/utils/patch`: modifica en caliente el prototipo de una clase ya definida, añadiendo o sustituyendo métodos, y deja intacto el resto.

```js
import { patch } from "@web/core/utils/patch";
import { useBus } from "@web/core/utils/hooks";
import { KanbanController } from "@web/views/kanban/kanban_controller";

patch(KanbanController.prototype, {
    setup() {
        super.setup(); // imprescindible: aquí es donde el controller monta su modelo
        useBus(this.env.bus, "task:starred_changed", (ev) => {
            // lo que se hace con el evento se ve en el ejemplo guiado de abajo
        });
    },
});
```

Tres cosas a tener en cuenta:

- **`patch()` se aplica al `prototype` de la clase**, no a una instancia. Se ejecuta una vez, normalmente al cargar el módulo, y a partir de ahí **toda** instancia futura de esa clase —cada vista *kanban* que se abra en la aplicación, sea del modelo que sea— pasa por tu código añadido.
- **Llamar a `super.setup()` es obligatorio si quieres conservar lo que la clase original hacía.** `patch()` reescribe el método, no lo añade al lado; sin la llamada a `super`, el `KanbanController` dejaría de montar su propio modelo y la vista no funcionaría en absoluto.
- Como el `patch()` afecta a **todas** las vistas *kanban* de la aplicación, casi siempre hace falta filtrar dentro del método para actuar solo cuando interesa (ver el ejemplo guiado).

`patch()` devuelve una función para deshacer el parche, pensada sobre todo para tests: en código de producción normalmente se aplica una vez al cargar el módulo y no se deshace nunca.

## Ejemplo guiado completo: sincronizar dos componentes independientes

Con las tres piezas ya vistas —`trigger`, `useBus` y `patch()`— se puede montar el caso completo: un widget de la barra de navegación marca una tarea como favorita, y una vista *kanban* de tareas que ya está abierta actualiza esa tarjeta sin que nadie la recargue a mano.

**Paso 1 — el widget dispara el evento después de guardar.** El widget de favoritos, tras confirmar el cambio contra el servidor, avisa con el `id` de la tarea afectada:

```js
/* widget de la barra de navegación */
import { Component } from "@odoo/owl";
import { rpc } from "@web/core/network/rpc";

export class FavoriteTasksMenu extends Component {
    async toggleStar(taskId, starred) {
        await rpc("/tasks/toggle_star", { task_id: taskId, starred });
        this.env.bus.trigger("task:starred_changed", { taskId, starred });
    }
}
```

**Paso 2 — la vista *kanban* de tareas se suscribe, filtrando por modelo.** El `patch()` de antes cobra sentido ahora: se aplica a *todos* los `KanbanController`, así que hay que comprobar el modelo antes de actuar, o una vista *kanban* de clientes reaccionaría también a un evento que no es asunto suyo.

```js
/* extensión del kanban, en un fichero aparte */
import { patch } from "@web/core/utils/patch";
import { useBus } from "@web/core/utils/hooks";
import { KanbanController } from "@web/views/kanban/kanban_controller";

patch(KanbanController.prototype, {
    setup() {
        super.setup();
        if (this.props.resModel === "project.task") {
            useBus(this.env.bus, "task:starred_changed", (ev) => {
                const { taskId } = ev.detail;
                const record = this.model.root.records.find((r) => r.resId === taskId);
                if (record) {
                    record.load();
                }
            });
        }
    },
});
```

**Paso 3 — recargar solo el registro que cambió, no toda la vista.** La línea que hace el trabajo real es `record.load()`. Cada fila visible de una vista basada en el modelo relacional de Odoo es un objeto `Record` con ese método: vuelve a leer ese registro concreto del servidor y actualiza su estado, y como ese estado es reactivo (OWL vuelve a pintar lo que depende de él), la tarjeta se actualiza sola en cuanto `load()` termina. No hace falta recargar la lista entera ni pedirle nada más al componente: `this.model.root.records` da la lista de registros cargados ahora mismo en la vista (agrupada o no), y basta con encontrar el que coincide por `resId`.

El resultado: el widget de la barra de navegación no sabe que existe una vista *kanban* abierta, y la vista *kanban* no sabe que existe un widget en la barra de navegación. Los dos solo conocen el nombre del evento.

## Cuándo esto no basta: notificar a otra pestaña o a otra persona

Todo lo anterior ocurre **dentro de una sola pestaña del navegador, en la sesión de una sola persona**. `env.bus` no manda nada por red: es un objeto en memoria del propio JavaScript, así que una segunda pestaña con la misma vista abierta, o la pantalla de otra persona conectada al mismo sistema, no reciben el evento y no tienen forma de recibirlo.

Cuando de verdad hace falta empujar un aviso a otra sesión —otra pestaña, otro usuario, incluso otra persona en otra ubicación—, Odoo tiene un mecanismo distinto y bastante más pesado: un bus real basado en WebSocket, con un canal de servidor y una suscripción explícita, donde es el propio backend quien decide a quién avisar. Son dos herramientas para dos problemas distintos: si la pregunta es "¿cómo entero a algo que ya está en esta misma pantalla?", la respuesta es `env.bus`; si es "¿cómo aviso a alguien que ni siquiera está mirando esta pantalla ahora mismo?", hace falta ese otro mecanismo, y montarlo para el primer caso sería usar una grúa para levantar un lápiz.

## Buenas prácticas avanzadas

- **Nombra los eventos con un espacio de nombres, como si fueran claves de configuración.** `"changed"` o `"update"` parecen inocentes hasta que dos funcionalidades distintas del mismo cliente web eligen el mismo nombre por casualidad y una empieza a reaccionar a eventos que no le incumben. Un prefijo estable (`"task:starred_changed"`, `"pos_order:paid"`) evita la colisión y además documenta de un vistazo quién es el dueño del evento.
- **Filtra siempre dentro del `patch()`, no confíes en que el evento "no va a llegar".** Un `patch()` sobre una clase base como `KanbanController` se aplica a cada instancia que exista en la aplicación, presente o futura. Si el filtro por `resModel` (o el criterio que corresponda) se te olvida, el código se ejecuta también para vistas de otros modelos, normalmente sin error visible — simplemente no encuentra nada que coincida, y el fallo se descubre mucho más tarde, cuando alguien reutiliza sin querer el mismo `resId` en otro contexto.
- **No dispares el evento antes de que el servidor confirme el cambio.** Es tentador disparar `trigger` justo antes de la llamada al servidor para que la interfaz se sienta instantánea, pero si esa llamada falla, cualquier vista que ya haya reaccionado queda mostrando un estado que nunca llegó a guardarse. Disparar después de que la respuesta llega es más lento en apariencia y correcto en la práctica.
- **No uses `env.bus` para pasar datos voluminosos.** El `payload` de `trigger` debería ser lo mínimo para que quien escucha sepa *qué* volver a pedir (un `id`, un par de claves), no *el* dato entero ya calculado. Si el evento lleva el objeto completo, cualquier componente que lo escuche empieza a depender de una forma de datos que solo conoce quien lo disparó, y ese acoplamiento oculto es más difícil de romper que el que `env.bus` pretendía evitar.
- **Comprueba que el registro sigue existiendo antes de tocarlo.** `this.model.root.records.find(...)` puede no encontrar nada — la tarjeta puede haberse filtrado fuera de la vista, o el grupo donde vivía puede estar plegado. El `if (record)` del ejemplo no es defensivo por exceso de cuidado: es la condición normal, no la excepcional.

## Documentación oficial

- [Odoo Developer Documentation — JavaScript services](https://www.odoo.com/documentation/18.0/developer/reference/frontend/services.html) — la puerta de entrada a `useService`, los *hooks* del cliente web y cómo un componente obtiene acceso a `env`.
- [Código de `hooks.js`](https://github.com/odoo/odoo/blob/18.0/addons/web/static/src/core/utils/hooks.js) — la implementación real de `useBus`, un puñado de líneas que muestran exactamente cuándo engancha y desengancha el listener.
- [Código de `patch.js`](https://github.com/odoo/odoo/blob/18.0/addons/web/static/src/core/utils/patch.js) — la implementación completa de `patch()`, incluida la función de deshacer que no se usa casi nunca en producción pero que explica cómo funciona `super` tras un parche.
- [Repositorio de OWL](https://github.com/odoo/owl) — el framework de componentes en el que vive la clase `EventBus`; el código fuente es corto y responde cualquier duda que esta guía no cubra sobre el ciclo de vida de un componente.

## Recursos didácticos

- [Owl Playground](https://odoo.github.io/owl/playground) — entorno interactivo del framework de componentes de Odoo, sin instalar nada. No incluye los servicios propios de Odoo (`useService`, `env.bus` tal cual se usa en el cliente web), pero es el sitio más rápido para entender qué es un componente, un `setup()` y un *hook* antes de leer código real del cliente web.

---

*En resumen: `env.bus` es el punto de encuentro para que dos partes de la interfaz que no se conocen entre sí se pongan de acuerdo sin acoplarse — uno avisa con `trigger`, el otro escucha con `useBus`, `patch()` presta el hueco en una vista ajena, y `record.load()` aplica el cambio justo donde hace falta.*
