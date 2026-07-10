# Search Panel

## ¿Qué es?

El *search panel* (`<searchpanel>`) es un panel lateral de filtrado que Odoo muestra a la izquierda de un listado o un kanban. Presenta los valores de uno o varios campos como categorías o casillas en las que hacer clic para filtrar, sin tener que escribir nada en la barra de búsqueda.

## ¿Por qué existe?

La barra de búsqueda es potente pero poco visual: hay que saber qué escribir. Cuando un listado se organiza de forma natural por una dimensión —la categoría del producto, el estado del pedido, el proyecto de una tarea— resulta mucho más cómodo ver todas las opciones de un vistazo y pinchar. El search panel convierte ese campo en una navegación lateral tipo "carpetas".

> Si has usado el filtro lateral de una tienda online (marca, talla, color), el search panel es exactamente eso, pero dentro de Odoo: un filtro por facetas.

## ¿Cuándo y para qué se usa?

Se usa en vistas de varios registros (lista y kanban) cuando hay una o dos dimensiones claras por las que la gente filtra constantemente: un catálogo que se recorre por categoría, un tablero de tareas que se filtra por proyecto y por estado, o una bandeja de documentos organizada por tipo. No aparece en formularios, porque un formulario muestra un solo registro.

## Lo mínimo que necesitas saber

**1. Se declara dentro de la vista de búsqueda**

```xml
<search>
  <searchpanel>
    <field name="categ_id" icon="fa-th-list" enable_counters="1"/>
  </searchpanel>
</search>
```

**2. Categoría (`select="one"`) vs. filtro (`select="multi"`)**

```xml
<searchpanel>
  <!-- categoría: una sola selección, como navegar por carpetas -->
  <field name="categ_id" select="one"/>
  <!-- filtro: varias casillas marcables a la vez -->
  <field name="state" select="multi" icon="fa-filter"/>
</searchpanel>
```

Por defecto un campo se comporta como categoría (`one`); `select="multi"` lo convierte en casillas combinables.

**3. Contadores de registros**

```xml
<field name="stage_id" enable_counters="1"/>
```

`enable_counters="1"` muestra junto a cada valor cuántos registros le corresponden.

**4. Acotar los valores y mostrar los vacíos**

```xml
<field name="categ_id"
       domain="[('active', '=', True)]"
       expand="1"/>
```

`domain` limita qué valores se ofrecen; `expand="1"` muestra también los valores que no tienen registros.

**5. Agrupar las opciones del panel**

```xml
<field name="tag_id" select="multi" groupby="category_id"/>
```

`groupby` agrupa las casillas por otro campo (por ejemplo, etiquetas agrupadas por su categoría).

## Lo que NO hace

- **No funciona en formularios** — solo en vistas de varios registros (lista, kanban).
- **No sustituye a la barra de búsqueda** — convive con ella; sus filtros se combinan (AND) con lo que escribas.
- **No admite muchos campos** — está pensado para una o dos dimensiones; abusar de él llena la pantalla y confunde.
- **No filtra por campos calculados no almacenados**, salvo que definan un método `search` (ver [Dominios dinámicos](Dominios-Dinamicos.md)).

## Buenas prácticas avanzadas

- **`enable_counters` cuesta rendimiento** — cada contador es una consulta de agrupación adicional. En modelos con millones de filas puede ralentizar la carga; actívalo solo donde de verdad aporte.
- **Limita las categorías con muchos valores** — un panel de tipo categoría con cientos de opciones es inmanejable. Odoo desactiva el panel cuando un campo supera cierto límite de valores (unos cientos); usa un `domain` que reduzca las opciones o replantéate si ese campo es buena faceta.
- **`groupby` es para `select="multi"`** — agrupar las opciones tiene sentido en las casillas de filtro, no en la lista de categorías de selección única.
- **Elige campos de baja cardinalidad** — estado, categoría, fase, tipo. Un campo con un valor casi único por registro (como el nombre o la referencia) no agrupa nada útil y solo estorba.

## Recursos didácticos

- En modo desarrollador, "Editar vista de búsqueda" desde el menú de depuración te deja ver el XML real de cualquier search panel ya existente en Odoo y usarlo como plantilla.

---

*En resumen: el search panel es el filtro por facetas de Odoo — convierte los valores de un campo en una navegación lateral de un clic, ideal para las dimensiones por las que siempre filtras.*
