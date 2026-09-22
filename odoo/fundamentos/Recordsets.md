# Recordsets

## ¿Qué es?

Un *recordset* es la estructura de datos central del ORM de Odoo: una colección ordenada de
registros de un mismo modelo. No hay un tipo separado para "un registro" y otro para "una lista
de registros" — incluso un único resultado, o ningún resultado, siguen siendo un recordset (de
tamaño 1 o de tamaño 0). Todo lo que el ORM te devuelve —una búsqueda, un campo relacional, el
resultado de filtrar otro recordset— es siempre este mismo tipo.

## ¿Por qué existe?

En SQL a pelo, o en un ORM más clásico, normalmente hay una distinción explícita entre "un
objeto" y "una lista de objetos", y hay que comprobar aparte si ese objeto existe o es `null`.
Odoo colapsa esas tres situaciones —ninguno, uno, varios— en un único tipo. Eso evita la rama de
código "¿y si no hay ninguno?" por separado, y permite encadenar operaciones de conjunto
(filtrar, transformar, combinar) directamente sobre el resultado de cualquier consulta, sin
distinguir si detrás hay 0, 1 o 500 registros.

> Si conoces LINQ o los DataFrames de pandas, un recordset se parece a eso: un conjunto sobre el
> que encadenas operaciones (`filtered`, `mapped`, `sorted`) en vez de recorrerlo a mano con un
> bucle y comprobar antes si está vacío.

## ¿Cuándo y para qué se usa?

Cada vez que se interactúa con el ORM: una búsqueda (`search`), acceder a un registro por su id
(`browse`), o leer un campo relacional (`Many2one`, `One2many`, `Many2many`) de un registro ya
cargado. El ejemplo que sigue esta guía es una aplicación de gestión de tareas: el modelo `Task`
tiene un campo `stage_id` (`Many2one` a `Stage`, la etapa en la que está la tarea) y un campo
`tag_ids` (`Many2many` a `Tag`).

## De dónde salen los recordsets y sus operaciones básicas

Las dos formas más habituales de obtener uno:

```python
tasks = env["task"].search([("stage_id.name", "=", "In Progress")])  # 0, 1 o varios registros
task = env["task"].browse(42)                                        # apunta al id 42
```

`browse(42)` **no comprueba que el registro exista** ni consulta la base de datos todavía — solo
construye un recordset que *apunta* a ese id. Si el id 42 no existe, `task` sigue siendo un
recordset válido de tamaño 1, y solo falla cuando intentas leer un campo suyo. Para comprobarlo
sin leer nada, está `task.exists()`, que devuelve el subconjunto de `task` que sí está en la base
de datos (vacío si no existía).

Un recordset se comporta como un conjunto, con operaciones que **siempre devuelven otro
recordset**, nunca una lista de Python:

```python
urgentes = tasks.filtered(lambda t: t.priority == "high")   # subconjunto que cumple la condición
nombres = tasks.mapped("name")                               # aquí sí: una lista de valores, no un recordset
ordenadas = tasks.sorted(key=lambda t: t.create_date)

todas = tasks_a | tasks_b      # unión
solo_a = tasks_a - tasks_b     # diferencia
comunes = tasks_a & tasks_b    # intersección
```

Y como colección que puede estar vacía, su valor de verdad será `False` cuando no contiene
ningún registro — por eso `if not tasks:` es el modismo habitual para "no hay resultados", igual
de válido tanto si `tasks` viene de un `search()` sin coincidencias como si es el recordset vacío
por defecto de un campo `Many2one` sin rellenar.

## Cómo se comparan dos recordsets

Un recordset **no se compara por identidad de objeto Python**, sino por **modelo + conjunto de
ids**. Dos recordsets del mismo modelo, con los mismos ids, son iguales aunque sean dos objetos
Python distintos en memoria:

```python
task_a = env["task"].browse(42)
task_b = env["task"].browse(42)

task_a == task_b        # True: mismo modelo, mismo id, aunque sean dos objetos distintos
task_a is task_b        # False casi siempre — no es esta la comparación que quieres
```

Esto es exactamente lo que hace natural comparar un campo `Many2one` contra otro recordset del
mismo modelo:

```python
if task.stage_id == other_task.stage_id:
    ...   # ambas tareas están en la misma etapa
```

## La trampa: comparar un recordset con un valor plano

Aquí está el error más fácil de cometer con recordsets, y el más silencioso. `env.ref(xml_id)`
—la función que resuelve un [ID externo](../configuracion-parametros/Referencias-por-ID-Externo.md)
a su registro— **devuelve un recordset**, no un número:

```python
etapa_done = env.ref("mi_modulo.stage_done")
type(etapa_done)   # task.stage(3,) — un recordset, no un int
```

Si se guarda tal cual en una estructura pensada para ids numéricos, la comparación deja de
funcionar sin avisar:

```python
# ❌ mal: falta el .id
def etapas_por_clave(env):
    return {
        "done": env.ref("mi_modulo.stage_done"),      # recordset, no id
        "todo": env.ref("mi_modulo.stage_todo"),
    }

stage_ids = etapas_por_clave(env)

if task.stage_id.id == stage_ids["done"]:   # int == recordset
    task.message_post(body="Tarea completada")
```

Ese `if` **nunca se cumple**. No lanza ninguna excepción — Odoo compara un recordset contra un
valor de otro tipo, decide que no son comparables, registra un aviso en el log parecido a
*"Comparing apples and oranges"* y la expresión evalúa a `False`. El código se ejecuta sin
errores, simplemente no hace nunca lo que debería: la tarea nunca se marca como completada, y
nadie se entera hasta que alguien nota que el aviso falta siempre.

La corrección es añadir el `.id` al construir la estructura, no al usarla:

```python
# ✅ bien: se guarda el id, no el recordset
def etapas_por_clave(env):
    return {
        "done": env.ref("mi_modulo.stage_done").id,
        "todo": env.ref("mi_modulo.stage_todo").id,
    }

stage_ids = etapas_por_clave(env)

if task.stage_id.id == stage_ids["done"]:   # int == int
    task.message_post(body="Tarea completada")
```

La regla práctica: si una variable va a compararse contra `algo.id` o va a viajar dentro de un
dominio (`[("campo", "in", [...])]`), termina de resolverla con `.id` en el mismo sitio donde la
construyes. Mezclar "a veces guardo el recordset, a veces el id" en la misma función es la
receta para este bug.

## Los dominios esperan ids, no recordsets

El mismo desliz aparece al construir un dominio de búsqueda. `search()` espera que el lado
derecho de una condición `"in"` sea una lista de valores planos (ids), no de recordsets:

```python
etapas_visibles = [env.ref("mi_modulo.stage_todo"), env.ref("mi_modulo.stage_done")]  # ❌
env["task"].search([("stage_id", "in", etapas_visibles)])
```

Aquí, a diferencia de la comparación con `==`, el fallo suele ser ruidoso: el adaptador de
PostgreSQL no sabe convertir un recordset en un parámetro de consulta, y la búsqueda lanza una
excepción en vez de devolver un resultado silenciosamente vacío. Sigue siendo el mismo olvido de
fondo —un recordset donde tocaba un id—, solo que aquí el error se nota enseguida en vez de
esconderse.

```python
etapas_visibles = [env.ref("mi_modulo.stage_todo").id, env.ref("mi_modulo.stage_done").id]  # ✅
env["task"].search([("stage_id", "in", etapas_visibles)])
```

## Buenas prácticas avanzadas

- **Las funciones que resuelven ids externos deberían devolver ids, nunca recordsets.** Si una
  función se llama `get_stage_ids` o similar, que su tipo de retorno sea siempre `int` o una
  lista/diccionario de `int`. Mezclar recordsets y sus ids bajo el mismo nombre es lo que produce
  el bug de esta guía.
- **`bool(recordset)` es tamaño, no verdad de negocio.** `if task.stage_id:` no pregunta "¿está
  la tarea en una etapa válida?", pregunta "¿el campo tiene algún registro?". Para un
  `Many2one` sin rellenar, ambas preguntas coinciden; para un `One2many`/`Many2many`, un
  recordset vacío es tan "falso" como uno con un registro inactivo que ya no debería contar —
  hay que distinguir ambos casos a propósito si importa.
- **`ensure_one()` antes de leer un campo cuando el método asume un único registro.** Leer
  `task.name` sobre un recordset de más de un elemento lanza un error de Odoo poco claro sobre
  el propio campo; llamar primero a `self.ensure_one()` falla en el sitio correcto, con un
  mensaje que dice exactamente cuántos registros había.
- **Un recordset vacío del modelo correcto no es lo mismo que `False` a secas.** `self.browse()`
  (sin argumentos) crea un recordset vacío del modelo de `self`, útil como valor inicial en un
  bucle que va acumulando con `|=`. Usar `False` en su lugar funciona para las comprobaciones de
  verdad, pero rompe en cuanto alguien intenta encadenar `.filtered()` o `.mapped()` sobre ese
  valor inicial antes de que se le sume nada.

## Documentación oficial

- [Recordsets — ORM API de Odoo](https://www.odoo.com/documentation/18.0/developer/reference/backend/orm.html#recordsets) — la referencia formal de cómo se construyen, comparan y combinan.

---

*En resumen: un recordset es siempre un conjunto —nunca un objeto ni un `null`—, se compara por
modelo e ids, y el bug más traicionero es guardar el recordset donde hacía falta su `.id`: no
falla, simplemente deja de ser cierto para siempre.*
