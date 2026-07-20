# Valores por defecto

## ¿Qué es?

Son los valores que Odoo **precarga en un campo al crear un registro nuevo**, antes de que el usuario escriba nada. Odoo ofrece varios niveles para definirlos: desde el código del módulo (`default=`), desde la configuración sin programar (`ir.default`, los *valores por defecto de usuario*) y de forma puntual al lanzar una acción (a través del contexto).

## ¿Por qué existe?

Rellenar siempre los mismos campos a mano es tedioso y propenso a olvidos. Si el 90 % de las facturas usan el mismo diario, o toda tarea nace en la etapa "Por hacer", tiene sentido que ese valor ya venga puesto. Los valores por defecto ahorran clics y, sobre todo, evitan que se creen registros incompletos o incoherentes.

> Es lo mismo que el valor por defecto de un parámetro en una función o de una columna en SQL (`DEFAULT`): si no se indica nada, se usa un valor razonable predefinido.

## ¿Cuándo y para qué se usa?

Siempre que un campo tenga un valor "casi siempre igual" al crear: la fecha de hoy en un campo de fecha, la compañía actual, un estado inicial, un responsable habitual. Según de dónde deba salir ese valor —fijo en el código, elegido por el cliente, o dependiente de desde dónde se crea el registro— se usa un mecanismo u otro.

## Lo mínimo que necesitas saber

**1. `default=` en la definición del campo (nivel código)**

Para un valor fijo o calculado por una función:

```python
class Pedido(models.Model):
    _inherit = "sale.order"

    fecha_alta = fields.Date(default=fields.Date.context_today)
    responsable_id = fields.Many2one("res.users", default=lambda self: self.env.user)
```

Un valor constante se pone tal cual (`default=10`); uno dinámico, con una función o un `lambda self:`.

**2. `ir.default`: valores por defecto sin tocar código**

Permite a un administrador fijar el valor por defecto de un campo desde la interfaz (*Ajustes → Técnico → Valores por defecto de usuario*), opcionalmente solo para un usuario o una compañía:

```python
self.env["ir.default"].set(
    "sale.order", "note", "Gracias por su compra",
    user_id=False, company_id=False,
)
```

Es la vía para cambiar un valor por defecto **sin desplegar código nuevo**.

**3. El contexto: `default_<campo>` (puntual)**

Una acción o un botón puede precargar un campo pasándolo en el contexto, útil cuando el valor depende de desde dónde se abre el formulario:

```python
return {
    "type": "ir.actions.act_window",
    "res_model": "project.task",
    "view_mode": "form",
    "context": {"default_project_id": self.id},
}
```

Al abrir el formulario de tarea, `project_id` ya vendrá relleno con ese proyecto.

## Lo que NO hace

- **No es configuración global de negocio** — un valor por defecto solo actúa al **crear**; no es un parámetro que la lógica consulte luego. Para eso están los [parámetros del sistema](Parametros-del-Sistema.md).
- **No obliga a nada** — es una sugerencia inicial; el usuario puede cambiar el valor antes de guardar.
- **`default=` no se cambia sin redeploy** — al vivir en el código, modificarlo exige actualizar el módulo. Si necesitas que el cliente lo cambie por su cuenta, usa `ir.default`.

## Buenas prácticas avanzadas

- **Usa `lambda self:` para los valores dinámicos** — un `default=fields.Date.today()` evaluado directamente se calcularía una sola vez al cargar el modelo, no en cada creación. Con `default=lambda self: fields.Date.today()` (o `context_today`) el valor se recalcula cada vez, que casi siempre es lo que quieres.
- **Elige el nivel según quién manda sobre el valor** — si lo decide el desarrollador, `default=`; si lo decide el cliente y puede cambiar, `ir.default`; si depende del origen del registro (desde qué proyecto, qué pedido…), el contexto `default_`. Mezclarlos mal lleva a valores que "no hay forma de cambiar" o que "cambian cuando no deberían".
- **El contexto `default_` es la forma limpia de enlazar acciones** — cuando un botón crea un registro relacionado, precargar los campos con `default_` en el contexto es más robusto y legible que sobrescribir `default_get()` a mano.

---

*En resumen: los valores por defecto precargan un campo al crear un registro —fijos en el código con `default=`, configurables sin programar con `ir.default`, o puntuales según el origen con `default_` en el contexto—; elige el nivel según quién debe poder cambiarlos.*
