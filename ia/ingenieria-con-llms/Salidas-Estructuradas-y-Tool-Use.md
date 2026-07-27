# Salidas estructuradas y tool use

## ¿Qué es?

Son los dos mecanismos que convierten un LLM en una pieza integrable de software. Las **salidas estructuradas** obligan al modelo a responder con un JSON que cumple un esquema que tú defines. El ***tool use*** (o *function calling*) le permite pedirle a tu código que ejecute algo —consultar la base de datos, llamar a una API— y seguir razonando con el resultado.

Van juntos en una misma guía porque por debajo son el mismo truco: describir con un esquema JSON la forma exacta de los datos que quieres, y dejar que el modelo la rellene.

## ¿Por qué existe?

Un modelo devuelve texto, y el texto es un formato terrible para hablar con código. Si le pides «dame los datos en JSON», lo normal es recibir algo así:

```text
Claro, aquí tienes los datos extraídos:

```json
{"nombre": "Teclado mecánico", "precio": 89.90}
```

He asumido que el precio incluye IVA. ¿Necesitas algo más?
```

Ese `json.loads()` falla. Y falla de forma intermitente: unas veces hay prosa alrededor, otras vienen las tres comillas del bloque Markdown, otras sale limpio. Puedes escribir un extractor con expresiones regulares para recortar el JSON del medio, y muchos proyectos lo hacen, pero es una capa frágil que hay que mantener para siempre.

El segundo problema es distinto: hay preguntas que el modelo **no puede** responder por mucho que le pidas. «¿Cuántas unidades del SKU-1042 hay en stock?» no está en sus pesos, está en tu base de datos. Sin herramientas, el modelo hace lo único que sabe hacer: producir el texto más plausible, es decir, inventarse un número.

> Si has trabajado con contratos de API, las salidas estructuradas son un esquema OpenAPI aplicado a la respuesta del modelo, y el *tool use* es el modelo actuando como cliente de tus endpoints: él dice qué quiere llamar y con qué parámetros, tú ejecutas y le devuelves el resultado.

## ¿Cuándo y para qué se usa?

- **Salidas estructuradas**: siempre que la respuesta vaya a código en lugar de a una persona. Clasificar, extraer datos de documentos, puntuar, enrutar, rellenar un formulario.
- ***Tool use***: siempre que la respuesta dependa de datos que el modelo no puede conocer (stock, precios, estado de un pedido), de cálculo exacto, o cuando el modelo deba *hacer* algo (crear un ticket, enviar un email).

---

# Parte 1 — Salidas estructuradas

## El esquema como contrato

En vez de pedir el formato en el *prompt*, se declara:

```python
ESQUEMA_PRODUCTO = {
    "type": "object",
    "properties": {
        "nombre":    {"type": "string"},
        "precio":    {"type": "number"},
        "categoria": {"type": "string", "enum": ["periferico", "monitor", "portatil", "otro"]},
        "en_stock":  {"type": "boolean"},
    },
    "required": ["nombre", "precio", "categoria", "en_stock"],
    "additionalProperties": False,
}

respuesta = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    output_config={"format": {"type": "json_schema", "schema": ESQUEMA_PRODUCTO}},
    messages=[{"role": "user", "content":
        "Extrae los datos: 'Teclado mecánico RGB, 89,90 €, disponible'"}],
)

import json
datos = json.loads(next(b.text for b in respuesta.content if b.type == "text"))
print(datos)
```

Salida:

```
{'nombre': 'Teclado mecánico RGB', 'precio': 89.9, 'categoria': 'periferico', 'en_stock': True}
```

Sin prosa, sin bloque Markdown, sin campos inventados, y `categoria` es obligatoriamente uno de los cuatro valores del `enum`. El `additionalProperties: False` es lo que impide que añada campos de su cosecha, y es obligatorio en cada objeto del esquema.

## Con clases en lugar de diccionarios

En Python, definir el esquema con Pydantic evita mantener el JSON a mano y —más importante— devuelve un objeto ya validado y con tipos:

```python
from pydantic import BaseModel, Field
from typing import Literal

class ProductoExtraido(BaseModel):
    nombre: str = Field(min_length=1)
    precio: float = Field(gt=0)
    categoria: Literal["periferico", "monitor", "portatil", "otro"]
    en_stock: bool

respuesta = client.messages.parse(
    model="claude-sonnet-5",
    max_tokens=1024,
    output_format=ProductoExtraido,
    messages=[{"role": "user", "content":
        "Extrae los datos: 'Teclado mecánico RGB, 89,90 €, disponible'"}],
)

producto = respuesta.parsed_output      # instancia de ProductoExtraido, ya validada
print(producto.precio + 10)             # es un float de verdad, no un string
```

`messages.parse()` genera el esquema a partir de la clase, valida la respuesta y te da el objeto. El editor te autocompleta los campos y el *type checker* te avisa si escribes `producto.precio_total`.

## Lo que el esquema sí garantiza y lo que no

Aquí está el error conceptual que hay que evitar: **el esquema garantiza la forma, nunca el contenido.**

```python
# Válido según el esquema. Y probablemente falso.
{'nombre': 'Teclado', 'precio': 0.01, 'categoria': 'otro', 'en_stock': True}
```

Así que la validación de dominio sigue siendo tuya:

```python
try:
    producto = respuesta.parsed_output
except ValidationError as e:
    registrar_fallo_extraccion(e)
    return None

# El esquema dice que precio es un número > 0. El negocio dice más:
if not (1 <= producto.precio <= 10_000):
    marcar_para_revision_manual(producto)
    return None
```

Además, el esquema JSON admitido es un subconjunto del estándar. Lo que **no** se puede expresar:

- **Esquemas recursivos** (un árbol de categorías con hijos del mismo tipo).
- **Restricciones numéricas** (`minimum`, `maximum`, `multipleOf`).
- **Restricciones de longitud de cadena** (`minLength`, `maxLength`).

Sí se admiten `enum`, `const`, `anyOf`, `$ref` y los formatos de cadena habituales (`date`, `email`, `uri`). Los SDK de Python y TypeScript retiran las restricciones no soportadas del esquema que envían y las validan en el cliente, así que un `Field(gt=0)` de Pydantic sigue funcionando —pero como validación local, no como garantía del modelo.

Dos detalles operativos:

- **La primera petición con un esquema nuevo es más lenta**: hay una compilación que se cachea 24 horas. No te alarmes si el primer *benchmark* sale raro.
- **No es compatible con citas** ni con el prellenado del turno del asistente.

---

# Parte 2 — Tool use

## Definir una herramienta

Una herramienta son tres campos: nombre, descripción y esquema de entrada.

```python
HERRAMIENTAS = [
    {
        "name": "consultar_stock",
        "description": (
            "Consulta las unidades disponibles de un producto en el almacén. "
            "Úsala siempre que la persona pregunte por disponibilidad, stock o "
            "si un producto está agotado. No estimes el stock por tu cuenta."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "sku": {
                    "type": "string",
                    "description": "Código del producto, con el formato SKU-1234",
                },
            },
            "required": ["sku"],
        },
    },
]
```

**La descripción es la parte que más influye en que la herramienta se use bien, y la que más se descuida.** Fíjate en que no dice solo *qué hace*: dice *cuándo llamarla* y qué no hacer en su lugar. Esa forma prescriptiva es la que marca la diferencia con los modelos actuales, que tienden a ser conservadores al decidir si usan una herramienta o responden directamente.

## El ciclo completo

El modelo no ejecuta nada: pide que ejecutes tú.

```python
mensajes = [{"role": "user", "content": "¿Tenéis stock del SKU-1042?"}]

respuesta = client.messages.create(
    model="claude-sonnet-5", max_tokens=1024,
    tools=HERRAMIENTAS, messages=mensajes,
)

print(respuesta.stop_reason)   # "tool_use"
```

En `content` llega un bloque con la petición:

```python
{
    "type": "tool_use",
    "id": "toolu_01A2b3C4d5E6f7",
    "name": "consultar_stock",
    "input": {"sku": "SKU-1042"},
}
```

Y tú cierras el ciclo en tres pasos:

```python
# 1. Añadir la respuesta del modelo al historial, ENTERA
mensajes.append({"role": "assistant", "content": respuesta.content})

# 2. Ejecutar cada herramienta pedida y recoger los resultados
resultados = []
for bloque in respuesta.content:
    if bloque.type != "tool_use":
        continue
    try:
        unidades = repositorio.stock_por_sku(bloque.input["sku"])
        contenido, es_error = f"{unidades} unidades disponibles", False
    except ProductoNoEncontrado:
        contenido, es_error = f"No existe el SKU {bloque.input['sku']}", True

    resultados.append({
        "type": "tool_result",
        "tool_use_id": bloque.id,        # debe coincidir con el id del tool_use
        "content": contenido,
        "is_error": es_error,
    })

# 3. Devolver TODOS los resultados en un ÚNICO mensaje de usuario
mensajes.append({"role": "user", "content": resultados})

final = client.messages.create(
    model="claude-sonnet-5", max_tokens=1024,
    tools=HERRAMIENTAS, messages=mensajes,
)
print(next(b.text for b in final.content if b.type == "text"))
```

Salida:

```
Sí, quedan 12 unidades del SKU-1042 en almacén.
```

Cuatro reglas que se rompen a menudo:

1. **Añade `respuesta.content` completo al historial**, no solo el texto. Si pierdes los bloques `tool_use`, la API rechaza el mensaje siguiente porque hay un `tool_result` sin su `tool_use` correspondiente.
2. **Cada `tool_use` necesita su `tool_result`.** Si el modelo pide tres herramientas y solo respondes a dos, la petición devuelve un 400.
3. **Todos los `tool_result` van en un solo mensaje de usuario.** Repartirlos en varios mensajes «funciona», pero le enseña al modelo a dejar de pedir herramientas en paralelo, y pierdes la concurrencia.
4. **Los errores se devuelven con `is_error: True`, no se omiten.** El modelo puede entonces reintentar con otro parámetro o explicarle el problema al usuario. Omitir el resultado lo deja esperando algo que nunca llega.

Y el bucle no termina en una vuelta: el modelo puede pedir otra herramienta a partir del resultado de la primera. La estructura general —el `while` que sigue hasta que deje de pedir— es el bucle agéntico que se trata en [Agentes con LLMs](Agentes-LLM.md).

## `tool_choice`: quién decide

| Valor | Comportamiento |
|---|---|
| `{"type": "auto"}` | el modelo decide (por defecto) |
| `{"type": "any"}` | obligado a usar alguna herramienta |
| `{"type": "tool", "name": "..."}` | obligado a usar esa herramienta concreta |
| `{"type": "none"}` | no puede usar ninguna |

Forzar una herramienta concreta es un truco útil para extracción: defines una herramienta cuyo esquema es la estructura que quieres y obligas a llamarla. Hoy, con salidas estructuradas disponibles, para *extraer datos* es preferible `output_config.format`; `tool_choice` forzado se reserva para cuando de verdad hay que ejecutar algo.

## `strict: true`: garantizar el esquema de los parámetros

Por defecto, el modelo *intenta* cumplir el esquema de entrada de la herramienta. Con `strict` se garantiza:

```python
{
    "name": "crear_pedido",
    "description": "Crea un pedido para un cliente existente.",
    "strict": True,                       # va en la herramienta, no en tool_choice
    "input_schema": {
        "type": "object",
        "properties": {
            "cliente_id": {"type": "integer"},
            "sku":        {"type": "string"},
            "cantidad":   {"type": "integer", "enum": [1, 2, 3, 4, 5]},
        },
        "required": ["cliente_id", "sku", "cantidad"],
        "additionalProperties": False,     # obligatorio con strict
    },
}
```

Con esto te ahorras la comprobación de que `cantidad` no llega como `"dos"`. Requiere `additionalProperties: False` y que todos los campos estén en `required`. Nota: `strict` no es compatible con la llamada programática a herramientas ni con `tool_choice` forzado.

## Herramientas de servidor: las que no ejecutas tú

Algunas herramientas las ejecuta el proveedor. Las declaras y no hay ciclo que cerrar:

```python
respuesta = client.messages.create(
    model="claude-opus-5",
    max_tokens=8192,
    tools=[
        {"type": "web_search_20260209", "name": "web_search"},
        {"type": "code_execution_20260120", "name": "code_execution"},
    ],
    messages=[{"role": "user", "content":
        "Calcula la desviación típica de estos valores y busca la media del sector."}],
)
```

La búsqueda web y la ejecución de código se resuelven en el servidor y sus resultados llegan como bloques dentro de la misma respuesta. La ejecución de código es especialmente relevante porque **resuelve el problema del cálculo**: con ella, un total o una desviación típica se calculan de verdad en lugar de predecirse.

Los errores de estas herramientas **no lanzan excepción**: llegan como un 200 con un bloque de resultado que contiene un objeto de error (por ejemplo `{"error_code": "max_uses_exceeded"}`). Hay que comprobarlo antes de leer el contenido.

## Dejar el bucle al SDK

Escribir el bucle a mano tiene sentido para entenderlo, y a veces para tener control total. Para el resto, el SDK lo hace:

```python
from anthropic import beta_tool

@beta_tool
def consultar_stock(sku: str) -> str:
    """Consulta las unidades disponibles de un producto en el almacén.

    Args:
        sku: Código del producto, con el formato SKU-1234.
    """
    return f"{repositorio.stock_por_sku(sku)} unidades disponibles"

runner = client.beta.messages.tool_runner(
    model="claude-sonnet-5",
    max_tokens=1024,
    tools=[consultar_stock],
    messages=[{"role": "user", "content": "¿Tenéis stock del SKU-1042?"}],
)

for mensaje in runner:
    print(mensaje)
```

El decorador genera el esquema a partir de la firma y del *docstring*, y el *runner* recorre el ciclo hasta que el modelo deja de pedir herramientas. Y sigue siendo interceptable: cada iteración te entrega el mensaje **antes** de ejecutar, así que una puerta de aprobación humana no obliga a volver al bucle manual —basta con no dejar que la herramienta se ejecute sin confirmación.

## Seguridad: las herramientas son la superficie de ataque

Una herramienta es código tuyo ejecutándose con parámetros que ha decidido un modelo, que a su vez ha leído texto que puede venir de un usuario. Eso es entrada no confiable, con todas las consecuencias:

```python
# Mal: SQL construido con lo que decida el modelo
def buscar_productos(filtro: str) -> str:
    return db.query(f"SELECT * FROM Productos WHERE Nombre LIKE '%{filtro}%'")

# Bien: parámetros, validación y un límite
def buscar_productos(filtro: str) -> str:
    if len(filtro) > 100:
        return "El filtro es demasiado largo."
    filas = db.query(
        "SELECT Sku, Nombre, Precio FROM Productos WHERE Nombre LIKE @f LIMIT 20",
        {"f": f"%{filtro}%"},
    )
    return json.dumps(filas)
```

Tres reglas mínimas:

- **Valida los parámetros como si vinieran de un formulario público.** Rangos, longitudes, formatos, listas de valores permitidos.
- **Pon aprobación humana en lo irreversible.** Enviar un email, cobrar, borrar. Una herramienta de solo lectura puede ejecutarse sola; una que escribe fuera del sistema, no.
- **Aplica el privilegio mínimo.** La herramienta debe poder hacer exactamente lo que necesita y nada más. Si consulta stock, su conexión no debería poder escribir.

El desarrollo completo está en [Seguridad en aplicaciones con LLMs](Seguridad-en-Aplicaciones-LLM.md).

---

## Buenas prácticas avanzadas

- **Escribe descripciones de herramienta prescriptivas: cuándo llamarla, no solo qué hace.** «Consulta el stock» produce una herramienta que el modelo usa a veces; «úsala siempre que pregunten por disponibilidad, stock o si algo está agotado, y no estimes el stock por tu cuenta» produce una que se usa cuando debe. Los modelos recientes son conservadores al decidir si llaman a una herramienta, así que la condición de activación dentro de la propia descripción es la palanca de mayor efecto —más que cualquier ajuste en el `system`.
- **Mantén el catálogo de herramientas corto y sin solapamientos.** Cada herramienta ocupa contexto en cada petición y añade una decisión que el modelo puede equivocar; dos herramientas que hacen casi lo mismo garantizan que a veces elija la que no toca. Antes de añadir la número quince, pregúntate si tres de las existentes no deberían ser una con un parámetro.
- **Valida el dominio aunque uses salidas estructuradas, y registra los rechazos.** El esquema te da la forma; `{"precio": 0.01}` es válido y falso. La tasa de fallos de validación de dominio es además la mejor alarma temprana que vas a tener: cuando sube sin que hayas tocado nada, algo se ha degradado —el formato de los documentos de entrada, el modelo, tu *prompt*.
- **Devuelve los errores de herramienta con `is_error: True` en lugar de omitirlos o de lanzar.** Un modelo que recibe «no existe el SKU-9999» reintenta con otro parámetro o se lo explica al usuario; un modelo que no recibe nada se queda esperando, y tu excepción sin capturar rompe el bucle entero por un error que era recuperable.
- **No reparta nunca los `tool_result` de un mismo turno en varios mensajes.** Es el error más silencioso de todos: la petición funciona, así que nadie lo detecta, pero el modelo aprende del formato del historial y deja de pedir herramientas en paralelo. Pierdes la concurrencia sin ningún síntoma más que una latencia que sube y nadie sabe por qué.
- **Marca cada herramienta como de lectura o de escritura, y trátalas distinto por diseño.** Las de lectura pueden ejecutarse sin fricción y en paralelo; las que escriben fuera del sistema necesitan confirmación, idempotencia y registro. Esa distinción explícita en el código —no en la cabeza de quien lo escribió— es lo que evita que un día una herramienta de «actualizar» se ejecute sin supervisión porque estaba en la misma lista que las de consulta.
- **Usa ejecución de código en lugar de confiar en el cálculo del modelo.** Cualquier número que importe —totales, medias, conversiones, fechas— debe salir de código ejecutado, no de predicción de tokens. Es el fallo silencioso más caro que existe con LLMs: el resultado sale con formato perfecto y valor equivocado, y nadie lo revisa porque parece bien.

## Recursos didácticos

- **[Tool use overview (Anthropic)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)** — la referencia completa del ciclo, con los formatos exactos de cada bloque. Imprescindible cuando la API devuelva un 400 que no entiendas.
- **[Structured outputs (Anthropic)](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)** — qué parte de JSON Schema se admite y qué no, con la lista de limitaciones. Ahorra la hora de descubrir a base de errores que `minimum` no está soportado.
- **[JSON Schema — Learn](https://json-schema.org/learn/getting-started-step-by-step)** — el tutorial oficial del estándar. Como los esquemas son la base tanto de las salidas estructuradas como de las herramientas, media hora aquí se amortiza en las dos.
- **[Anthropic Cookbook — tool use](https://github.com/anthropics/anthropic-cookbook/tree/main/tool_use)** — notebooks con el ciclo completo funcionando, incluidas varias herramientas en paralelo y el manejo de errores.

---

*En resumen: un esquema JSON es el contrato que convierte texto en datos y capacidades en herramientas — pero garantiza la forma, no la verdad, así que valida siempre el contenido.*
