# Dominios dinámicos

## ¿Qué es?

Un dominio dinámico es un filtro que no está fijado de antemano, sino que se calcula según el contexto: el valor de otro campo del formulario, el usuario actual, la fecha o una lógica en Python. En vez de "muestra siempre estos registros", dice "muestra los que correspondan ahora mismo".

## ¿Por qué existe?

Muchos filtros no se pueden escribir como una lista fija. El desplegable de "producto" de una línea de pedido debería ofrecer solo los productos del almacén elegido arriba; las opciones válidas cambian según lo que el usuario haya seleccionado. Un dominio estático no sabe reaccionar a eso. Los dominios dinámicos permiten que el filtro dependa de datos que solo se conocen en ejecución.

> Piensa en el clásico "país → provincia → ciudad": cada desplegable se recalcula en función del anterior. Eso es un dominio dinámico.

## ¿Cuándo y para qué se usa?

Aparece siempre que "las opciones válidas dependen de algo": el campo de ciudad que se acota por país, el responsable que solo puede ser alguien del equipo elegido, o poder filtrar un listado por un campo calculado (como "pedidos con retraso") que no existe como columna en la base de datos.

## Lo mínimo que necesitas saber

**1. `domain` en el campo, referenciando otro campo**

```xml
<field name="country_id"/>
<field name="city_id" domain="[('country_id', '=', country_id)]"/>
```

En cuanto cambias el país, las ciudades ofrecidas se recalculan solas.

**2. `domain` en la definición del campo (Python)**

```python
warehouse_id = fields.Many2one('stock.warehouse')
product_id = fields.Many2one(
    'product.product',
    domain="[('warehouse_id', '=', warehouse_id)]")
```

**3. Filtrar por un campo calculado no almacenado: método `search`**

Un campo calculado sin `store=True` no se puede filtrar... salvo que le des un método `search`.

```python
is_late = fields.Boolean(compute='_compute_is_late', search='_search_is_late')

def _search_is_late(self, operator, value):
    today = fields.Date.today()
    return [('date_deadline', '<', today)]
```

Ahora `[('is_late', '=', True)]` funciona en filtros y en el search panel.

**4. Filtros dependientes del usuario con el `context`**

```python
# 'uid' es el usuario actual; se resuelve en el momento de buscar
[('user_id', '=', uid)]
```

**5. Un método que devuelve el dominio**

```python
def _get_partner_domain(self):
    return [('company_id', 'in', self.env.companies.ids)]
```

## Lo que NO hace

- **No ejecuta SQL arbitrario** — sigue siendo un dominio; la lógica compleja va en el método que lo construye, no en el filtro.
- **El `domain` de un campo no valida** — restringe lo que el usuario ve en el desplegable, pero por API se puede grabar un valor fuera de ese dominio; si importa, valida aparte.
- **Un campo calculado no almacenado no es buscable por sí solo** — sin método `search`, filtrar por él da error.

## Buenas prácticas avanzadas

- **Prefiere el `domain` declarativo al `onchange` que devuelve dominio** — devolver `{'domain': {...}}` desde un `onchange` está desaconsejado en versiones modernas de Odoo; poner el `domain` en el campo (XML o definición) referenciando otros campos es más limpio y sobrevive a los cambios del cliente web.
- **El método `search` debe traducir operador y valor** — recibe el `operator` y el `value` reales (`=`, `!=`, `True`, `False`...); si solo contemplas el caso `('=', True)` y alguien filtra con `!=`, devolverás resultados incorrectos. Cúbrelos o lanza un error explícito.
- **Vigila el coste del `search` calculado** — como no hay columna indexable, tu método debe traducir la condición a un dominio sobre campos que sí estén almacenados; nunca cargues todos los registros en memoria para filtrarlos en Python.
- **`uid`, `company_id` y `context_today()` se evalúan tarde** — al ser variables del contexto, el mismo dominio se adapta a cada usuario y momento. Aprovéchalo en vez de calcular valores fijos al definir la vista.

## Recursos didácticos

- La documentación de referencia del ORM de Odoo, en el apartado de campos (`Field`), detalla cómo implementar el parámetro `search` y cuándo un campo calculado es buscable.

---

*En resumen: los dominios dinámicos hacen que un filtro dependa del contexto —otro campo, el usuario, la fecha o una lógica en Python— para ofrecer siempre las opciones que tienen sentido en cada momento.*
