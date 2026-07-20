# Referencias por ID externo (XML ID)

## ¿Qué es?

Un **ID externo** (o *XML ID*) es un nombre estable y legible que Odoo asocia a un registro concreto de la base de datos —por ejemplo `base.group_user` para el grupo "Usuario interno"—. Sirve para **referirse a ese registro sin usar su ID numérico**. Por debajo, esas parejas "nombre → registro" se guardan en el modelo `ir.model.data`.

## ¿Por qué existe?

El ID numérico de un registro (el `id` de la fila en PostgreSQL) **no es fiable entre bases de datos distintas**: la etapa "Hecho" puede ser el `id` 322 en producción y el `id` 47 en una instalación nueva. Si el código dice `if stage_id == 322`, funcionará en un sitio y fallará silenciosamente en otro. El ID externo resuelve esto: es un identificador que **se mantiene igual** allá donde esté instalado el módulo, y Odoo se encarga de traducirlo al número que corresponda en cada base de datos.

> Es la diferencia entre citar a alguien por su DNI (opaco y distinto en cada país) o por un nombre acordado que todos reconocen. `project.stage_done` significa lo mismo en cualquier instalación; `322`, no.

## ¿Cuándo y para qué se usa?

Siempre que el código o una vista necesiten **apuntar a un registro concreto que se comporta como configuración**: un grupo de seguridad, una etapa de proyecto, un producto especial, un diario contable, una plantilla de correo. En cuanto veas un número mágico comparándose con un `id`, ahí falta un ID externo.

## Lo mínimo que necesitas saber

**1. Resolver un ID externo desde Python**

```python
grupo = self.env.ref("base.group_user")
etapa_hecho = self.env.ref("mi_modulo.stage_done")

if tarea.stage_id == etapa_hecho:
    ...
```

`env.ref("modulo.nombre")` devuelve el registro correspondiente en *esta* base de datos.

**2. Darle un ID externo a un registro que creas en XML**

El `id` que pones en un `<record>` de un fichero de datos **es** su ID externo:

```xml
<record id="stage_done" model="project.task.type">
    <field name="name">Hecho</field>
</record>
```

Desde otro módulo se referenciaría como `mi_modulo.stage_done`.

**3. Referenciar un registro dentro de otro XML**

Con `ref` en los campos relacionales:

```xml
<record id="regla_solo_lectura" model="ir.rule">
    <field name="name">Solo lectura para usuarios</field>
    <field name="groups" eval="[(4, ref('base.group_user'))]"/>
</record>
```

**4. Encontrar el ID externo de un registro existente**

Con el modo desarrollador activo, en el formulario del registro: menú de desarrollador → *Ver metadatos* (*View Metadata*), donde aparece su *External ID*.

## Lo que NO hace

- **No crea el registro** — `env.ref` solo lo *busca*. El registro debe existir ya (venir en un fichero de datos o haberse creado antes).
- **No falla en silencio por defecto** — si el ID externo no existe, `env.ref` lanza un error. Para tolerarlo se usa `self.env.ref("modulo.nombre", raise_if_not_found=False)`, que devuelve `False` en su lugar.
- **No es un valor de texto configurable** — para un plazo, una URL o un flag numérico, el sitio son los [parámetros del sistema](Parametros-del-Sistema.md); el ID externo es para **apuntar a un registro**, no para guardar un dato suelto.

## Buenas prácticas avanzadas

- **Nunca compares contra IDs numéricos clavados en el código** — un `STAGE_DONE_ID = 322` esparcido por los módulos es una bomba de relojería: el día que se reinstala en otra base de datos, apunta a otra cosa o a nada. Sustitúyelo por `self.env.ref("mi_modulo.stage_done")` y el problema desaparece.
- **`env.ref` frente a `search` por nombre** — buscar un registro por su nombre (`search([("name", "=", "Hecho")])`) es frágil: basta con que alguien lo renombre o lo traduzca para romperlo. El ID externo es inmune a cambios de nombre, así que es la referencia preferida para registros de configuración.
- **`noupdate="1"` para datos que el cliente puede tocar** — si defines un registro por XML pero quieres que las actualizaciones del módulo no machaquen lo que el cliente haya cambiado a mano, márcalo con `noupdate="1"` (o mételo en una sección `<data noupdate="1">`). Sin eso, cada actualización revierte sus ajustes.
- **Prefija con el módulo y usa nombres con significado** — `mi_modulo.stage_done` se entiende de un vistazo; un ID externo genérico o sin prefijo invita a colisiones y a confusión cuando otro módulo referencia lo que no es.

## Recursos didácticos

- Documentación de ficheros de datos y IDs externos de Odoo: <https://www.odoo.com/documentation/18.0/developer/reference/backend/data.html>

---

*En resumen: el ID externo es el nombre estable de un registro —usa `env.ref("modulo.nombre")` en lugar de IDs numéricos clavados, y tu código dejará de romperse al cambiar de base de datos.*
