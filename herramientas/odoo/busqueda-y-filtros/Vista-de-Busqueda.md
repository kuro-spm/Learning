# Vista de búsqueda

## ¿Qué es?

La vista de búsqueda (`<search>`) es la definición en XML de la barra de búsqueda que aparece encima de cualquier listado en Odoo: el campo donde escribes, los filtros predefinidos y las agrupaciones disponibles.

## ¿Por qué existe?

Un listado sin forma de acotarlo es inservible en cuanto hay miles de registros. Odoo separa "qué datos hay" (el modelo) de "cómo los busco" (la vista de búsqueda), de modo que puedes definir de forma declarativa los filtros que tienen sentido para ese modelo sin escribir lógica: le dices qué campos son buscables, qué filtros rápidos ofreces y por qué campos se puede agrupar.

## ¿Cuándo y para qué se usa?

Cada modelo que se muestra en una lista o kanban tiene (o hereda) una vista de búsqueda. Es donde defines el filtro "Solo activos" de un catálogo de productos, la búsqueda por nombre de cliente en una lista de pedidos, o la agrupación "por estado" en un tablero de tareas.

## Lo mínimo que necesitas saber

**1. Campos buscables**

```xml
<search>
  <field name="name"/>
  <field name="partner_id"/>
</search>
```

**2. Filtros predefinidos con dominio**

```xml
<filter name="solo_activos" string="Activos"
        domain="[('active', '=', True)]"/>
<filter name="este_mes" string="Este mes"
        domain="[('date_order', '>=', context_today().replace(day=1))]"/>
```

**3. Agrupaciones (`group_by`)**

La agrupación viaja en el `context`, no en el `domain`.

```xml
<separator/>
<filter name="agrupar_estado" string="Estado"
        context="{'group_by': 'state'}"/>
```

**4. Activar un filtro por defecto desde la acción**

El prefijo `search_default_` enciende un filtro al abrir la vista.

```xml
<field name="context">{'search_default_solo_activos': 1}</field>
```

**5. Filtros contiguos = OR; separados = AND**

Dos `<filter>` seguidos (sin `<separator/>` en medio) se combinan con OR; separados por `<separator/>`, con AND.

## Lo que NO hace

- **No define las columnas** del listado — eso es la vista de lista (`<tree>`), no la de búsqueda.
- **No guarda los filtros** del usuario — los "favoritos" los guarda cada usuario en su sesión, no la vista.
- **No aplica seguridad** — ofrece u oculta filtros, pero no restringe qué puede ver cada usuario (eso son las reglas de acceso).

## Buenas prácticas avanzadas

- **Pon siempre `name` a los filtros** — sin `name` no puedes activarlos con `search_default_` ni heredarlos limpiamente desde otro módulo. El `string` es la etiqueta visible; el `name` es la identidad estable.
- **Agrupar es un `context`, no un `domain`** — un despiste habitual es intentar agrupar con un dominio. La agrupación va en `context={'group_by': ...}`; el dominio solo filtra.
- **Usa fechas relativas, no hardcodeadas** — funciones como `context_today()` se evalúan en el momento de la búsqueda y respetan la zona horaria del usuario, así que el filtro "Este mes" sigue siendo correcto mañana.
- **Hereda, no dupliques** — para añadir un filtro a un modelo existente, extiende su vista de búsqueda con `xpath` en lugar de redefinirla entera; así no pisas los filtros que otros módulos hayan añadido.

---

*En resumen: la vista de búsqueda es la capa declarativa donde defines, sin código, cómo se busca, se filtra y se agrupa un listado.*
