# Cómo piensa un LLM

## ¿Qué es?

Un **LLM** (*Large Language Model*, modelo grande de lenguaje) es un programa que, dado un texto, calcula qué fragmento de texto es más probable que venga a continuación, y lo va escribiendo pieza a pieza. Todo lo demás —que conteste preguntas, escriba código o use herramientas— sale de aplicar esa única operación muchas veces seguidas.

## ¿Por qué existe?

Durante décadas, hacer que una máquina entendiera lenguaje humano se intentó con reglas escritas a mano: diccionarios, gramáticas, listas de sinónimos. No escala, porque el lenguaje real está lleno de excepciones, ironía, jerga y contexto implícito. Los LLM invierten el enfoque: en vez de escribir las reglas, se entrena un modelo estadístico con cantidades enormes de texto para que las reglas emerjan solas. El resultado es un sistema que generaliza a frases que nunca vio, a cambio de no poder explicarte *por qué* respondió lo que respondió.

> Si ya conoces la autocorrección del teclado del móvil, un LLM es esa misma idea llevada al extremo: en vez de sugerir la siguiente palabra a partir de las tres anteriores, sugiere el siguiente fragmento a partir de cientos de miles de palabras de contexto, y lo hace tan bien que puede escribir una función entera.

## ¿Cuándo y para qué se usa?

Para cualquier tarea donde la entrada o la salida sea texto poco estructurado y las reglas sean difíciles de enumerar: clasificar los mensajes de un formulario de contacto en «incidencia / consulta / spam», extraer los datos de una factura en PDF a JSON, resumir las reseñas de un producto, traducir, reescribir, generar código, o conducir una conversación de soporte.

Y también para lo contrario de lo que la gente espera: **no** para calcular, **no** para consultar el precio real de un producto, **no** para garantizar que un dato es cierto. Esas cosas se resuelven dándole herramientas (ver [Tool use y salidas estructuradas](Salidas-Estructuradas-y-Tool-Use.md)), no confiando en su memoria.

Entender el mecanismo importa porque casi todos los fallos que te vas a encontrar en producción —alucinaciones, respuestas que cambian entre ejecuciones, instrucciones que se «olvidan», costes que se disparan— son consecuencias directas de cómo funciona por dentro.

---

## Tokens: la unidad real de trabajo

Un LLM no ve caracteres ni palabras: ve **tokens**. Un token es un fragmento de texto de tamaño variable (una palabra corta, un trozo de palabra larga, un signo de puntuación, un salto de línea). El texto se parte en tokens antes de entrar al modelo y la salida se genera token a token.

Esto no es un detalle académico: **todo se mide y se factura en tokens** —el contexto disponible, el precio, la latencia, los límites de la API.

Veamos cuántos tokens ocupa realmente un texto. La API de Anthropic tiene un endpoint específico para contarlos:

```python
import anthropic

client = anthropic.Anthropic()

texto = "Devuelve los productos activos ordenados por precio."

resultado = client.messages.count_tokens(
    model="claude-opus-5",
    messages=[{"role": "user", "content": texto}],
)
print(resultado.input_tokens)
```

Salida (aproximada):

```
16
```

Dieciséis tokens para 51 caracteres: el ratio típico en español ronda **3-4 caracteres por token**. Dos consecuencias prácticas:

- **El código consume más tokens que la prosa.** La indentación, los símbolos y los nombres tipo `productoDescatalogadoRepository` se parten en muchos tokens. Un archivo de 500 líneas de C# puede rondar los 6.000-8.000 tokens.
- **El conteo depende del modelo.** Cada familia de modelos tiene su propio *tokenizador*. El mismo texto puede dar 1.000 tokens en un modelo y 1.300 en otro, lo que cambia el coste sin que tú hayas cambiado nada.

> **Nunca estimes tokens con la librería de otro proveedor.** `tiktoken` es el tokenizador de OpenAI; usarlo para estimar tokens de Claude subestima entre un 15 % y un 20 % en texto normal, y bastante más en código. Si necesitas el número exacto, pide el conteo al proveedor con su propio endpoint, como en el ejemplo anterior.

---

## De texto a probabilidades: por qué la misma pregunta da respuestas distintas

En cada paso, el modelo no elige «la» siguiente palabra: produce una **distribución de probabilidad** sobre todos los tokens posibles. Algo conceptualmente así:

```text
Contexto: "El método devuelve una lista de "

  productos     0.34
  usuarios      0.11
  resultados    0.09
  objetos       0.07
  ... (miles más, cada uno con su probabilidad)
```

Después hay que **elegir uno**. Y ahí está el origen del no determinismo: si siempre se cogiera el más probable, el texto sería repetitivo y plano, así que el proceso normal incluye algo de azar controlado. Los parámetros clásicos para regular ese azar son:

- **`temperature`** — cuánto se «achatan» las probabilidades. Cerca de 0, el modelo casi siempre coge el token más probable (respuestas repetibles, previsibles, algo secas). Más alto, reparte más y produce salidas más variadas y creativas.
- **`top_p`** — en vez de tocar la forma de la distribución, recorta la cola: solo se consideran los tokens que acumulan una probabilidad de `p` (por ejemplo, 0,9), y se descarta el resto.

En una API que los admita, se usan así:

```python
respuesta = client.messages.create(
    model="claude-haiku-4-5",
    max_tokens=1024,
    temperature=0.0,   # mínima variación entre ejecuciones
    messages=[{"role": "user", "content": "Clasifica: '¿cuándo llega mi pedido?'"}],
)
```

Aquí `temperature=0.0` busca la salida más estable posible, que es lo que quieres en clasificación o extracción de datos. Para redactar textos de marketing querrías lo contrario.

Tres avisos que separan a quien ha leído un tutorial de quien ha llevado esto a producción:

1. **`temperature=0` no garantiza salidas idénticas.** Reduce mucho la variación, pero el cálculo en GPU no es bit a bit reproducible y el proveedor puede cambiar la infraestructura por debajo. Nunca diseñes un sistema que rompa si dos llamadas idénticas devuelven textos distintos.
2. **Ajusta uno, no los dos.** `temperature` y `top_p` atacan el mismo problema por caminos distintos; combinarlos hace el comportamiento difícil de razonar. Muchas APIs directamente rechazan la petición si envías ambos.
3. **Los modelos más nuevos han eliminado estos parámetros.** En los modelos de última generación de Anthropic (`claude-opus-5`, `claude-sonnet-5`, `claude-fable-5`), enviar `temperature`, `top_p` o `top_k` devuelve un error **400**. La forma de controlar el comportamiento en ellos es el *prompt* y el parámetro de esfuerzo (más abajo). Si migras código antiguo, esto es lo primero que rompe.

---

## El modelo no recuerda nada: la ventana de contexto

Cada llamada a la API es **independiente**. El modelo no guarda estado entre peticiones: si la conversación parece tener memoria es porque tú le vuelves a enviar el historial completo en cada turno.

Se ve claro montando una conversación a mano:

```python
mensajes = [
    {"role": "user", "content": "Me llamo Sara y trabajo con .NET."},
]

r1 = client.messages.create(model="claude-opus-5", max_tokens=1024, messages=mensajes)
texto1 = next(b.text for b in r1.content if b.type == "text")

# Para el segundo turno hay que reenviar TODO: la pregunta anterior y la respuesta.
mensajes.append({"role": "assistant", "content": texto1})
mensajes.append({"role": "user", "content": "¿En qué lenguaje trabajo?"})

r2 = client.messages.create(model="claude-opus-5", max_tokens=1024, messages=mensajes)
print(next(b.text for b in r2.content if b.type == "text"))
```

Salida:

```
Trabajas con .NET.
```

Sabe la respuesta porque está en el array `mensajes` que acabas de enviar, no porque «se acuerde». Si hubieras omitido los dos primeros elementos, no tendría ni idea.

El límite de cuánto texto cabe en una llamada es la **ventana de contexto**, y se mide en tokens (los modelos actuales de Anthropic manejan 1 millón; los más pequeños, 200.000). Esa ventana incluye las instrucciones de sistema, el historial, los documentos que adjuntes, las definiciones de herramientas y la respuesta que se está generando. Cuando se llena, hay que decidir qué se tira o se resume.

Todo lo que rodea a esa decisión —qué metes, en qué orden, cómo recortas cuando se llena— es un tema en sí mismo: está desarrollado en la colección de [Context Engineering](../context-engineering/README.md), y en particular en [Ventana de Contexto](../context-engineering/Ventana-de-Contexto.md) y [Compactación de Contexto](../context-engineering/Compactacion-de-Contexto.md).

---

## Por qué alucina

Una **alucinación** es una afirmación que el modelo produce con total naturalidad y que es falsa: un método que no existe en la librería, un parámetro inventado, una cita atribuida a quien no la dijo.

No es un bug: es el mecanismo funcionando como debe. El modelo genera lo que es *plausible* según el patrón del texto, y una firma de método inventada puede ser perfectamente plausible. El modelo no tiene un canal separado para decir «esto no lo sé»; su salida es siempre el siguiente token más probable.

Dos ejemplos con los que te vas a topar programando:

```python
# El modelo "conoce" una librería hasta cierta fecha. Si le preguntas por una
# función añadida después, la respuesta plausible es inventarse una firma:
#
#   "Usa client.messages.create_batch_async(requests=[...])"
#
# ...que no existe. La firma real es client.messages.batches.create(requests=[...]).
```

La lección: **cuanto más específico y verificable sea el dato, menos debes fiarte de la memoria del modelo.** Lo que funciona:

- **Meter la fuente en el contexto.** Si le pasas la documentación real de la librería en el *prompt*, deja de inventar porque tiene de dónde copiar. Esa es la idea entera de [RAG](../context-engineering/RAG.md).
- **Darle herramientas para consultar la verdad.** Una búsqueda web, una consulta a tu base de datos, un lector de ficheros.
- **Verificar en lugar de confiar.** Compilar, ejecutar los tests, comprobar que el endpoint existe. Ver [Revisión de código generado](Revision-de-Codigo-Generado.md).
- **Pedirle que admita la duda, explícitamente.** Funciona sorprendentemente bien: «Si no estás seguro de la firma exacta, dilo en lugar de inventarla».

---

## Razonamiento explícito: *thinking*

Los modelos recientes pueden generar, antes de la respuesta final, un bloque de razonamiento intermedio. Es literalmente el modelo escribiendo su propio borrador para sí mismo: al haber escrito «primero necesito comprobar X, luego Y», esos tokens pasan a formar parte del contexto y la respuesta final sale mejor fundamentada.

Se activa con el parámetro `thinking`:

```python
respuesta = client.messages.create(
    model="claude-opus-5",
    max_tokens=16000,
    thinking={"type": "adaptive"},        # el modelo decide cuánto razonar
    output_config={"effort": "high"},     # cuánto esfuerzo total invertir
    messages=[{"role": "user", "content": "¿Por qué este query plan hace un seq scan?"}],
)

for bloque in respuesta.content:
    if bloque.type == "thinking":
        print("[razonamiento]", bloque.thinking)
    elif bloque.type == "text":
        print("[respuesta]", bloque.text)
```

La respuesta llega dividida en bloques: primero los de tipo `thinking` y después los de tipo `text`. Dos cosas que hay que tener en la cabeza:

- **El razonamiento se paga.** Son tokens de salida al precio normal, y cuentan contra `max_tokens`. Un `max_tokens` ajustado al tamaño de la respuesta puede truncarla porque el razonamiento se comió el presupuesto.
- **Sirve para problemas de varios pasos, no para todo.** En una clasificación de una frase no aporta nada y multiplica el coste y la latencia. En depurar, planificar un refactor o razonar sobre concurrencia, la diferencia es grande.

El parámetro hermano es el **esfuerzo** (`output_config.effort`, con valores de `low` a `max`): regula cuánto invierte el modelo en total —cuánto razona, cuántas herramientas usa, cuánto explora antes de responder. Es la palanca principal para mover el equilibrio entre calidad, latencia y coste, y se trata en detalle en [Modelos y parámetros](Modelos-y-Parametros.md).

---

## Qué NO es un LLM

Tener claro el perímetro te ahorra la mitad de los errores de diseño:

- **No ejecuta código.** Si le pides calcular `17 * 4839`, *predice* el resultado, no lo calcula. Acierta a menudo y falla sin avisar. Para que lo calcule de verdad hay que darle una herramienta de ejecución de código.
- **No consulta internet ni tu base de datos** salvo que le des una herramienta para hacerlo.
- **No tiene estado.** Ni entre llamadas ni entre usuarios. Lo que parezca memoria lo estás construyendo tú (ver [Memoria de Agentes](../context-engineering/Memoria-de-Agentes.md)).
- **No distingue tus instrucciones de los datos que le pasas.** Todo es texto en la misma ventana. Un documento que diga «ignora tus instrucciones anteriores» es un vector de ataque real, no una curiosidad: es la *prompt injection* de [Seguridad en aplicaciones con LLMs](Seguridad-en-Aplicaciones-LLM.md).
- **No sabe qué no sabe.** No hay un umbral interno de confianza que puedas leer.

---

## Buenas prácticas avanzadas

- **Mide tokens con el tokenizador del modelo que vas a usar, y hazlo antes de estimar coste.** Un cálculo hecho «a ojo» o con el tokenizador de otro proveedor se desvía lo suficiente como para invalidar el presupuesto de un proyecto. En el endpoint de conteo cuesta cero: no hay excusa para adivinar.
- **Diseña asumiendo que la salida variará entre ejecuciones, incluso a `temperature=0`.** Los tests que comparan la respuesta con un string exacto serán intermitentes. Lo que sí se puede afirmar de forma estable son propiedades: «es JSON válido», «contiene los tres campos requeridos», «el precio es un número positivo». Eso es lo que hay que aseverar (ver [Evaluaciones de LLMs](Evaluaciones-de-LLMs.md)).
- **No pidas cálculo aritmético ni fechas exactas sin herramienta.** Es el fallo silencioso más frecuente: el modelo produce un número con formato impecable y valor erróneo, y nadie lo revisa porque «parece bien». Si el número importa, que lo calcule código.
- **El error que confunde a más gente: creer que el modelo «se olvidó» de una instrucción.** Casi nunca es olvido. Suele ser que la instrucción quedó enterrada entre 200.000 tokens de historial, o que otra instrucción posterior la contradice, o que la reformulaste de forma ambigua. La solución no es repetirla en mayúsculas: es revisar qué hay realmente en el contexto y en qué orden.
- **Nunca dejes que el razonamiento y la respuesta compitan por el mismo `max_tokens` sin margen.** Si activas `thinking` sobre un `max_tokens` calculado para la respuesta, la salida se cortará a mitad de frase con `stop_reason: "max_tokens"`. Sube el límite o baja el esfuerzo, pero decídelo tú en vez de descubrirlo en producción.
- **Trata la fecha de corte de conocimiento como un dato de diseño.** Todo modelo tiene una fecha a partir de la cual no sabe nada. Para cualquier cosa que cambie —versiones de librerías, precios, APIs— la respuesta correcta no es «preguntar mejor», es meter la información actual en el contexto.

## Recursos didácticos

- **[Tiktokenizer](https://tiktokenizer.vercel.app/)** — pega texto y ve cómo se parte en tokens, con colores. Diez segundos aquí explican mejor que cualquier párrafo por qué el código consume más tokens que la prosa.
- **[LLM Visualization](https://bbycroft.net/llm)** — un LLM real recorrido en 3D, capa por capa, mientras procesa un texto. La mejor forma de intuir que dentro no hay magia sino álgebra lineal.
- **[But what is a GPT? — 3Blue1Brown](https://www.youtube.com/watch?v=wjZofJX0v4M)** — el mecanismo explicado visualmente y sin matemáticas previas.
- **[Documentación de la API de Anthropic](https://platform.claude.com/docs)** — la referencia de todo lo que aparece en esta colección: ventanas de contexto, *thinking*, esfuerzo, precios.

---

*En resumen: un LLM solo sabe predecir el siguiente token — todo lo que parece inteligencia, memoria o certeza lo construyes tú alrededor de esa única operación.*
