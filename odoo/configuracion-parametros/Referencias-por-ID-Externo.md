# Referencias por ID externo (XML ID)

## ¿Qué es?

Un **ID externo** (o *XML ID*) es un nombre estable y legible que Odoo asocia a un registro concreto de la base de datos —por ejemplo `base.group_user` para el grupo "Usuario interno", o `project.project_stage_1` para una etapa de proyecto—. Sirve para **crear ese registro desde un módulo y para referirse a él después sin usar su ID numérico**. Por debajo, cada pareja "nombre → registro" se guarda como una fila del modelo `ir.model.data`, con el `id` numérico real del registro y el módulo que lo declaró.

## ¿Por qué existe?

El ID numérico de un registro (el `id` de la fila en PostgreSQL) **no es fiable entre bases de datos distintas**: la etapa "Hecho" puede ser el `id` 322 en una instalación y el `id` 47 en otra. Si el código dice `if stage_id == 322`, funcionará en un sitio y fallará en silencio en otro —comparará contra un registro que no existe, o contra uno que significa otra cosa—. El ID externo resuelve esto: es un identificador que **se mantiene igual** allá donde esté instalado el módulo, y Odoo se encarga de traducirlo al número que corresponda en cada base de datos.

> Es la diferencia entre citar a alguien por su DNI (opaco y distinto en cada país) o por un nombre acordado que todos reconocen. `mi_modulo.stage_done` significa lo mismo en cualquier instalación; `322`, no.

## ¿Cuándo y para qué se usa?

Siempre que el código o una vista necesiten **apuntar a un registro concreto que se comporta como configuración**: un grupo de seguridad, una etapa de un tablero Kanban, un producto especial, un diario contable, una plantilla de correo. En cuanto veas un número mágico comparándose con un `id`, ahí falta un ID externo.

No lo confundas con un valor configurable suelto (un plazo en días, una URL, un interruptor): eso vive en los [parámetros del sistema](Parametros-del-Sistema.md). El ID externo es para **apuntar a un registro**, no para guardar un dato.

## Resolver un ID externo desde Python

La función que vas a usar casi siempre es `env.ref`, que busca la pareja `(módulo, nombre)` en `ir.model.data` y devuelve el registro real de esta base de datos:

```python
grupo = self.env.ref("base.group_user")
etapa_hecho = self.env.ref("mi_modulo.stage_done")

if tarea.stage_id == etapa_hecho:
    tarea.message_post(body="Tarea completada")
```

Dos matices que conviene saber antes de usarla en serio:

- **No crea el registro, solo lo busca.** Si `mi_modulo.stage_done` no existe todavía en esta base de datos, `env.ref` no lo inventa: lanza `ValueError`. El registro tiene que venir ya declarado en un fichero de datos del módulo (siguiente sección) o haberse creado antes por otra vía.
- **Para tolerar que no exista**, pásale `raise_if_not_found=False`: devuelve `False` en lugar de lanzar el error, útil cuando el registro es opcional o viene de un módulo que podría no estar instalado.

```python
etapa_opcional = self.env.ref("otro_modulo.stage_revision", raise_if_not_found=False)
if etapa_opcional:
    ...
```

## Definir y enlazar datos de módulo en XML

Un ID externo no aparece solo: alguien lo declara. La forma habitual es un **fichero de datos** del módulo, listado en la clave `data` del `__manifest__.py`, con un `<record>` por registro. El `id` que le pones **es** su ID externo:

```xml
<!-- data/project_task_type_data.xml -->
<odoo>
    <data noupdate="1">
        <record id="stage_done" model="project.task.type">
            <field name="name">Hecho</field>
            <field name="sequence">40</field>
        </record>
    </data>
</odoo>
```

Desde otro módulo, o desde Python, ese registro se referencia como `mi_modulo.stage_done` (siempre `<módulo>.<id>`, salvo que se declare explícitamente con un prefijo distinto). Para enlazarlo dentro de **otro** XML se usa el atributo `ref`, o la función `ref()` dentro de un `eval`:

```xml
<record id="regla_solo_lectura" model="ir.rule">
    <field name="name">Solo lectura para usuarios</field>
    <field name="groups" eval="[(4, ref('base.group_user'))]"/>
</record>
```

El atributo `noupdate="1"` (a nivel de `<data>` o de un `<record>` suelto) le dice a Odoo: *aplica estos valores cuando el módulo se instala, pero no los toques en actualizaciones posteriores*. Sin él, cada `-u` de tu módulo revertiría cualquier cambio que un usuario hubiera hecho a mano sobre ese registro (renombrar la etapa, cambiar su color, reordenarla). Se usa casi siempre para datos que nacen como plantilla pero luego son del negocio.

Para encontrar el ID externo de un registro que ya tienes delante en la interfaz, con el modo desarrollador activo abre su formulario y ve a *menú de desarrollador → Ver metadatos* (*View Metadata*): ahí aparece su *External ID*, o el aviso de que no tiene ninguno.

## Adoptar bajo un módulo un registro que ya existía

Aquí aparece el caso incómodo: una etapa de tarea que alguien creó **a mano desde la interfaz**, mucho antes de que existiera tu módulo. Ese registro no tiene ninguna fila en `ir.model.data` — no lo declaró ningún XML, así que Odoo no sabe que corresponde a ningún ID externo.

Si simplemente añades el `<record id="stage_done" model="project.task.type">` de la sección anterior a tu módulo y lo instalas, Odoo **no reconoce** que ya existe una etapa "Hecho": busca `(mi_modulo, stage_done)` en `ir.model.data`, no encuentra nada, y crea una etapa **nueva**. El resultado es una etapa duplicada, con las tareas antiguas apuntando todavía a la de siempre y las nuevas cayendo en la recién creada.

La solución es **crear tú mismo, de antemano, la fila de `ir.model.data`** que le falta al registro existente, apuntando al `id` numérico que ya tiene. En cuanto esa fila existe, Odoo trata el registro como si ya fuera "suyo": cuando llegue el `<record id="stage_done">`, en vez de crear uno nuevo **actualiza el que ya hay**.

```python
def _adopt_existing_stage(env, module, xml_id, stage_name):
    """Vincula una etapa creada a mano a un ID externo del módulo, sin duplicarla."""
    IrModelData = env["ir.model.data"]

    ya_adoptada = IrModelData.search_count([
        ("module", "=", module),
        ("name", "=", xml_id),
    ])
    if ya_adoptada:
        return  # una ejecución anterior ya hizo este trabajo

    etapa = env["project.task.type"].search([("name", "=", stage_name)], limit=1)
    if etapa:
        IrModelData.create({
            "module": module,
            "name": xml_id,
            "model": "project.task.type",
            "res_id": etapa.id,
            "noupdate": True,
        })
```

Buscar por nombre (`stage_name`) es exactamente lo que se desaconseja como forma habitual de referenciar un registro (ver más abajo) — pero aquí es distinto: este código no se ejecuta en cada petición, sino **una sola vez**, en un momento controlado, con el único propósito de tender el puente hacia el ID externo. A partir de ahí, todo el resto del módulo usa `env.ref("mi_modulo.stage_done")` con normalidad.

### Un antipatrón fácil de escribir por error

La lógica anterior se confunde a menudo con esta otra, que parece razonable y no lo es:

```python
def _adopt_existing_stage_MAL(env, module, xml_id, stage_name):
    etapa = env.ref(f"{module}.{xml_id}", raise_if_not_found=False)
    if not etapa:
        env["project.task.type"].create({"name": stage_name})
```

La pregunta que hace este código es "¿ya existe un registro con este ID externo?", y si la
respuesta es no, **crea uno nuevo**. El problema es que, en el caso que nos ocupa, esa pregunta
siempre responde que no —el ID externo nunca ha existido, es precisamente lo que falta crear—,
así que esta función crea una etapa "Hecho" nueva cada vez que se ejecuta, sin tocar nunca la
etapa real que las tareas ya llevan usando. Y como tampoco registra el ID externo del registro
que acaba de crear, ni siquiera es idempotente: una segunda ejecución crea una tercera etapa
"Hecho".

La pregunta correcta no es "¿existe ya el puente?" sino "¿existe ya el registro que quiero
adoptar?" — por eso la versión de arriba busca la etapa **por nombre**, no por su (inexistente)
ID externo, y es ella la que decide si hay que crear la fila de `ir.model.data` o no.

## Hooks de instalación y migraciones de actualización

Falta decidir **cuándo** se ejecuta ese código de adopción, y la respuesta depende de si el módulo es nuevo o ya estaba instalado:

| Mecanismo | Cuándo se ejecuta | Para qué sirve |
|---|---|---|
| `pre_init_hook` | Antes de cargar los datos del módulo, solo en su **instalación** | Preparar el terreno antes de que se procesen los ficheros XML —el sitio para la adopción de registros existentes— |
| `post_init_hook` | Después de cargar todos los datos, solo en su **instalación** | Trabajo que necesita que los datos del módulo ya existan (crear registros derivados, lanzar un cálculo inicial) |
| `uninstall_hook` | Al **desinstalar** el módulo | Limpiar lo que el módulo dejó fuera de su propio control de datos (parámetros de sistema, ficheros externos) |
| `migrations/<versión>/pre-migrate.py`, `post-migrate.py`, `end-migrate.py` | En una **actualización** (`-u`) de un módulo que ya estaba instalado, cuando el número de versión del `__manifest__.py` sube | El equivalente de los tres hooks anteriores pero para módulos que no se instalan de cero |

El matiz importante: `pre_init_hook` y `post_init_hook` **solo se disparan la primera vez que el módulo se instala**. Si el módulo lleva meses en producción y publicas una nueva versión con el código de adopción, esa instalación no vuelve a ocurrir — lo que corre es una actualización, y ahí lo que Odoo ejecuta son los scripts de `migrations/`. Se registran por carpeta con el número de versión al que pertenecen y una función `migrate(cr, version)`:

```python
# migrations/18.0.1.1.0/pre-migrate.py
from odoo import api, SUPERUSER_ID


def migrate(cr, version):
    env = api.Environment(cr, SUPERUSER_ID, {})
    _adopt_existing_stage(env, "mi_modulo", "stage_done", "Hecho")
```

Para una instalación nueva, el mismo código se registra como hook normal en el manifest:

```python
{
    "name": "Mi módulo",
    ...
    "pre_init_hook": "_adopt_existing_stage_on_install",
}
```

con la función expuesta desde el `__init__.py` del módulo. Desde Odoo 17 la firma de los hooks recibe directamente un entorno (`def post_init_hook(env):`); en versiones anteriores recibía `(cr, registry)` y había que construirse el entorno a mano con `api.Environment(cr, SUPERUSER_ID, {})`, como en el ejemplo de migración de arriba.

## Buenas prácticas avanzadas

- **Nunca compares contra IDs numéricos clavados en el código.** Un `STAGE_DONE_ID = 322` esparcido por varios ficheros es una bomba de relojería: el día que el módulo se instala en otra base de datos, ese número apunta a otra cosa o a nada. Sustitúyelo por `self.env.ref("mi_modulo.stage_done")` y el problema desaparece de raíz.
- **`env.ref` frente a `search` por nombre, para el uso cotidiano del código.** Buscar un registro por su nombre (`search([("name", "=", "Hecho")])`) es frágil: basta con que alguien lo renombre o lo traduzca para romperlo. El ID externo es inmune a eso, así que es la referencia preferida para configuración — la búsqueda por nombre solo se justifica como paso puntual de migración, nunca como mecanismo habitual.
- **Adoptar un registro le ata su ciclo de vida al módulo.** Una vez que una etapa creada a mano tiene una fila en `ir.model.data` de tu módulo, desinstalarlo hará que Odoo intente borrarla junto con el resto de sus datos —aunque esa etapa existiera mucho antes que el módulo—. Si otro módulo o muchas tareas siguen apuntando a ella, la desinstalación puede fallar o arrastrar más de lo que esperabas. Revisa las dependencias antes de desinstalar un módulo que adoptó datos preexistentes.
- **`noupdate="1"` (o `noupdate=True` al crear la fila a mano) protege del pisoteo, no del borrado.** Evita que una actualización del módulo sobrescriba los valores que el negocio cambió a mano, pero no impide que una desinstalación borre el registro, ni que el propio módulo lo modifique explícitamente por código en tiempo de ejecución.
- **Prefija con el módulo y usa nombres con significado.** `mi_modulo.stage_done` se entiende de un vistazo; un ID externo genérico o sin prefijo invita a colisiones con el de otro módulo y a confusión sobre quién es el dueño del dato.

## Documentación oficial

- [Ficheros de datos e IDs externos](https://www.odoo.com/documentation/18.0/developer/reference/backend/data.html) — la referencia de `<record>`, `noupdate`, `ref` y `eval` dentro de un fichero de datos.
- [Manifiesto del módulo](https://www.odoo.com/documentation/18.0/developer/reference/backend/module.html) — dónde se documentan las claves `pre_init_hook`, `post_init_hook` y `uninstall_hook` del `__manifest__.py`.

## Recursos didácticos

- [openupgradelib](https://github.com/OCA/openupgradelib) — la librería que usa la comunidad OCA para migrar bases de datos entre versiones de Odoo trae, entre sus utilidades, funciones como `add_xmlid` que hacen exactamente la adopción de esta ficha (crear la fila de `ir.model.data` que le falta a un registro existente). Ver su código es la forma más rápida de comprobar que el patrón manual de arriba es el mismo que usa la comunidad.

---

*En resumen: el ID externo es el nombre estable de un registro —defínelo con `<record id="...">`, resuélvelo con `env.ref("modulo.nombre")` en vez de IDs numéricos clavados, y si el registro ya existía a mano, tiéndele el puente creando su fila de `ir.model.data` antes de que el módulo intente declararlo, para que lo actualice en vez de duplicarlo.*
