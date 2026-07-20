# Configuración por compañía

## ¿Qué es?

Es la forma de tener un mismo parámetro con un **valor distinto en cada compañía** dentro de una misma base de datos de Odoo. Se consigue de dos maneras que se complementan: campos en el modelo `res.company` (una empresa) y campos marcados con `company_dependent=True` en cualquier otro modelo (el valor se guarda por compañía de forma automática).

## ¿Por qué existe?

Odoo puede gestionar varias empresas a la vez sobre la misma instalación (multi-compañía). En ese escenario, muchos ajustes no pueden ser globales: cada empresa tiene su propio plazo de pago por defecto, su cuenta contable, su formato de factura. Un [parámetro del sistema](Parametros-del-Sistema.md) no sirve, porque es único para toda la base de datos. Hace falta un mecanismo que guarde "un valor por empresa".

> Si conoces la multi-tenancy, esto es su versión de grano fino: no separas los datos en bases distintas, sino que un mismo campo lleva un valor diferente según la compañía activa.

## ¿Cuándo y para qué se usa?

Siempre que la instalación tenga **más de una empresa** y un ajuste deba variar entre ellas: el diario contable por defecto, el plazo de pago, la cuenta de ingresos de un producto, un texto legal al pie de los documentos. Aunque haya una sola empresa hoy, usar el mecanismo correcto evita reescribirlo el día que se añada una segunda.

## Lo mínimo que necesitas saber

**1. Un campo propio de la empresa (`res.company`)**

```python
class ResCompany(models.Model):
    _inherit = "res.company"

    plazo_pago_id = fields.Many2one("account.payment.term", string="Plazo de pago por defecto")
```

Ese valor se edita normalmente desde [Ajustes](Ajustes.md), con un campo `related="company_id.plazo_pago_id"`.

**2. Un campo `company_dependent` en otro modelo**

Marcar un campo con `company_dependent=True` hace que Odoo guarde un valor por compañía sin que tengas que montar nada más:

```python
class ProductTemplate(models.Model):
    _inherit = "product.template"

    margen_objetivo = fields.Float(company_dependent=True)
```

El mismo producto mostrará `margen_objetivo = 20` cuando estés en una empresa y `15` en otra, según la compañía activa.

**3. Leer el valor de la compañía actual**

```python
plazo = self.env.company.plazo_pago_id
```

`self.env.company` es siempre la compañía activa del usuario en ese momento.

## Lo que NO hace

- **No es global** — es justo lo contrario de un parámetro del sistema: cada empresa ve lo suyo. Si buscas un valor único para toda la BD, esto no es lo que quieres.
- **No es por usuario** — la separación es por compañía, no por persona. Para valores por usuario, mira [Valores por defecto](Valores-por-Defecto.md).
- **No guarda el histórico** — al cambiar el valor de una compañía, machaca el anterior; no lleva un registro de versiones.

## Buenas prácticas avanzadas

- **Un campo `company_dependent` se comporta distinto al buscar y agrupar** — por dentro no vive en la columna normal de la tabla, sino como una propiedad por compañía (`ir.property`). Eso significa que filtrar o agrupar por él tiene limitaciones y sorpresas; no asumas que se comporta como un campo corriente en informes y dominios.
- **Expón siempre estos valores por Ajustes, no por el menú técnico** — la configuración por compañía es de negocio; el usuario debe cambiarla desde una pantalla clara con la empresa bien visible, no editando propiedades a mano.
- **Cuidado con qué compañía está activa al leer** — si un código automático (un cron, una acción de servidor) lee `self.env.company`, asegúrate de que corre con la compañía correcta; en multi-compañía es un origen habitual de "a veces sale un valor y a veces otro".

---

*En resumen: cuando el mismo parámetro debe valer distinto en cada empresa, se usa un campo en `res.company` o un `company_dependent=True` —el mecanismo multi-compañía de Odoo, lo opuesto a un parámetro global.*
