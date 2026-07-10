# Dominios

## ¿Qué es?

Un dominio es la forma que tiene Odoo de expresar un filtro sobre registros: una lista de condiciones que decide qué filas de una tabla se seleccionan. Es el equivalente al `WHERE` de SQL, pero escrito como una estructura de datos de Python.

## ¿Por qué existe?

En Odoo casi todo lo que filtra registros —una vista de lista, un botón de filtro, un campo relacional que solo debe ofrecer ciertas opciones— necesita decir "quédate solo con los registros que cumplan esto". En lugar de escribir SQL a mano en cada sitio, Odoo define un lenguaje único y portable: el dominio. El mismo dominio sirve en XML, en Python y en las llamadas del cliente web.

> Si vienes de SQL, piensa en un dominio como la cláusula `WHERE` convertida en lista: `WHERE state = 'done' AND amount > 100` se vuelve `[('state', '=', 'done'), ('amount', '>', 100)]`.

## ¿Cuándo y para qué se usa?

Aparece en cualquier punto donde haya que acotar registros: la lista de pedidos que muestra solo los del mes, el desplegable de "cliente" que solo enseña empresas y no personas, o el filtro guardado "facturas pendientes". Siempre que pienses "aquí quiero ver solo algunos", debajo hay un dominio.

## Lo mínimo que necesitas saber

**1. Una condición es una tupla de tres elementos**

```python
# (campo, operador, valor)
[('state', '=', 'done')]
```

**2. Varias condiciones se combinan con AND por defecto**

```python
# state = 'done' AND amount > 100
[('state', '=', 'done'), ('amount', '>', 100)]
```

**3. OR y NOT en notación prefija (polaca)**

El operador va delante de los operandos a los que afecta.

```python
# state = 'done' OR state = 'sent'
['|', ('state', '=', 'done'), ('state', '=', 'sent')]

# NOT (state = 'cancel')
['!', ('state', '=', 'cancel')]
```

**4. Operadores más habituales**

```python
[('name', 'ilike', 'madrid')]              # contiene, sin distinguir mayúsculas
[('state', 'in', ['done', 'sent'])]        # uno de la lista
[('partner_id', '!=', False)]              # tiene cliente asignado
[('categ_id', 'child_of', parent_id)]      # jerarquías (nodo y descendientes)
```

**5. Se usa igual desde Python**

```python
pedidos = self.env['sale.order'].search([('state', '=', 'done')])
```

## Lo que NO hace

- **No ordena ni agrupa** — un dominio solo filtra; el orden va aparte (`order=`) y la agrupación por otro lado (`group_by` / `read_group`).
- **No es SQL** — no admite subconsultas arbitrarias; el ORM traduce el dominio a SQL por ti.
- **No aplica permisos** — filtrar no sustituye a las reglas de acceso; un registro fuera de tu alcance no aparece aunque el dominio lo incluya.

## Buenas prácticas avanzadas

- **Cuidado al mezclar OR y AND en notación prefija** — `['|', A, B, C]` no significa "A o B o C": el `|` solo afecta a los dos términos siguientes, así que equivale a `(A o B) y C`. Para "A o B o C" necesitas dos operadores: `['|', '|', A, B, C]`. Es el error de dominios más frecuente.
- **`child_of` y `parent_of` recorren jerarquías** — en categorías o cuentas contables anidadas, `('id', 'child_of', x)` trae el nodo y toda su descendencia sin que tengas que recorrer el árbol a mano.
- **`in` con lista vacía nunca coincide** — `('id', 'in', [])` devuelve cero registros. Si construyes el dominio dinámicamente a partir de una lista, controla ese caso o filtrarás sin querer todo el listado.
- **El `False` es tu `IS NULL`** — `('date_end', '=', False)` selecciona los registros sin fecha. No uses `None` ni `''`: el operando correcto para "vacío" en Odoo es `False`.

## Recursos didácticos

- En modo desarrollador, cualquier filtro que compongas con la interfaz se puede inspeccionar con la opción "Editar filtro" / "Añadir filtro personalizado", que muestra el dominio real generado: una forma cómoda de aprender la sintaxis probando.

---

*En resumen: el dominio es el `WHERE` de Odoo escrito como lista — el idioma común con el que todo el sistema expresa "quédate solo con estos registros".*
