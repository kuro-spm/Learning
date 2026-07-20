# Ajustes (res.config.settings)

## ¿Qué es?

`res.config.settings` es el modelo que hay detrás de la pantalla de **Ajustes** de Odoo (*Settings*). Es un modelo especial —**transitorio**, no guarda registros permanentes— cuya única misión es ofrecer una interfaz bonita y agrupada para que un usuario configure parámetros, y traducir lo que rellena en la pantalla a donde esos valores se guardan de verdad.

## ¿Por qué existe?

Un [parámetro del sistema](Parametros-del-Sistema.md) es potente, pero su lista técnica de clave-valor no es sitio para un usuario de negocio: es árida, no valida nada y está escondida en el menú técnico. Hace falta una capa por encima con casillas, campos con etiqueta, secciones y textos de ayuda. Eso es `res.config.settings`: la vitrina amigable sobre los parámetros que viven por debajo.

> Si ya conoces la separación entre almacenamiento y presentación, piensa en `ir.config_parameter` como la tabla donde se guarda el dato y en `res.config.settings` como el formulario que lo muestra y lo edita.

## ¿Cuándo y para qué se usa?

Cuando un módulo tiene parámetros que **el cliente debe poder cambiar por su cuenta**, sin llamar a un desarrollador: activar una funcionalidad opcional, fijar un plazo, elegir un producto por defecto, meter la clave de una integración. Todo eso se añade como campos en la pantalla de Ajustes, en la sección del módulo correspondiente.

## Lo mínimo que necesitas saber

**1. Heredar el modelo y añadir campos**

```python
from odoo import fields, models

class ResConfigSettings(models.TransientModel):
    _inherit = "res.config.settings"

    dias_de_gracia = fields.Integer(
        string="Días de gracia",
        config_parameter="mi_modulo.dias_de_gracia",
    )
```

**2. La magia está en `config_parameter`**

Ese atributo conecta el campo con un [parámetro del sistema](Parametros-del-Sistema.md): al abrir Ajustes, Odoo **lee** el parámetro y rellena el campo; al pulsar *Guardar*, lo **escribe** de vuelta. No hay que programar el guardado.

**3. Enlazar con un campo de la compañía**

Si el valor debe ser distinto por empresa, en lugar de `config_parameter` se usa un campo `related` a `res.company` (ver [Configuración por compañía](Configuracion-por-Compania.md)):

```python
plazo_pago_id = fields.Many2one(
    "account.payment.term",
    related="company_id.plazo_pago_id",
    readonly=False,   # imprescindible para poder editarlo desde Ajustes
)
```

**4. Colocar el campo en la vista de Ajustes**

Se añade a la vista heredando `res_config_settings_view_form`, dentro del bloque de la aplicación correspondiente, para que aparezca en su sitio de la pantalla.

## Lo que NO hace

- **No almacena nada por sí mismo** — al ser transitorio, sus registros se borran solos. El dato real vive en `ir.config_parameter`, en `res.company` o en otro modelo; Ajustes solo es el intermediario.
- **No sustituye a los parámetros del sistema** — se apoya en ellos (o en campos de compañía); es la capa de presentación, no el almacén.
- **No es el sitio para configuración de infraestructura** — puertos, rutas o credenciales de BD siguen en [odoo.conf](El-Fichero-odoo-conf.md).

## Buenas prácticas avanzadas

- **`config_parameter` para lo global, `related` a la compañía para lo multi-empresa** — elegir mal el respaldo es el error clásico: si el valor debería variar por empresa y lo enganchas a un parámetro del sistema, todas las compañías compartirán el mismo, y al revés desperdicias complejidad para un valor que siempre es único.
- **No olvides `readonly=False` en los campos `related`** — por defecto un campo `related` es de solo lectura; sin ese atributo, el usuario ve el valor en Ajustes pero no puede cambiarlo, un despiste muy habitual.
- **Aprovecha `implied_group` para activar funcionalidades por permisos** — una casilla de Ajustes puede conceder o quitar un grupo de seguridad a los usuarios (`group='...'`, `implied_group='...'`), que es como Odoo enciende y apaga funcionalidades enteras sin tocar código.
- **Agrupa y explica** — la pantalla de Ajustes es lo primero que ve el cliente; usa los bloques, títulos y textos de ayuda de la vista para que cada opción se entienda sin manual.

---

*En resumen: `res.config.settings` es el escaparate amable de la configuración —no guarda el dato, lo presenta y lo enruta hacia el parámetro del sistema o el campo de compañía que hay debajo.*
