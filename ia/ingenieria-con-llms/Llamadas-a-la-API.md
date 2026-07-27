# Llamadas a la API

## ¿Qué es?

Es la petición HTTP con la que tu aplicación habla con un modelo: le envías un array de mensajes y recibes una respuesta con el texto generado y los metadatos de lo que ha costado. Prácticamente todo lo que se puede hacer con un LLM —conversar, extraer datos, usar herramientas, razonar— pasa por este mismo endpoint.

## ¿Por qué existe?

Porque un LLM es demasiado grande para ejecutarlo en tu servidor. Un modelo de frontera necesita hardware especializado y decenas de gigabytes de memoria solo para cargar los pesos, así que el patrón dominante es el mismo que con cualquier servicio gestionado: el proveedor lo opera y tú lo consumes por HTTP.

> Si has consumido una API REST de terceros, aquí no hay nada nuevo en el transporte: es un `POST` con JSON y una clave de API en una cabecera. Lo distinto son tres cosas —que se factura por tokens, que la respuesta puede tardar minutos, y que el servidor no guarda nada entre llamadas.

## ¿Cuándo y para qué se usa?

Cada vez que quieras usar un LLM desde tu propio código: un clasificador de mensajes de contacto, un extractor de datos de facturas, un asistente de soporte, un generador de descripciones de producto. Es la diferencia entre usar un LLM como herramienta personal —desde un chat o desde el editor— y convertirlo en una pieza de tu sistema.

---

## Preparar el cliente

El SDK oficial se instala como cualquier dependencia:

```bash
pip install anthropic
```

Y el cliente se construye sin argumentos, dejando que resuelva las credenciales del entorno:

```python
import anthropic

client = anthropic.Anthropic()   # lee ANTHROPIC_API_KEY del entorno
```

Ese constructor vacío no es pereza, es lo correcto: la clave viaja en una variable de entorno y no aparece en el código ni en Git. **La alternativa —`anthropic.Anthropic(api_key="sk-ant-...")`— es el error de seguridad más repetido en proyectos con LLM**, porque una clave en el repositorio sigue ahí después de borrarla del fichero: está en el historial. El manejo correcto está en [Gestión de secretos en desarrollo](../../seguridad/gestion-de-secretos-en-desarrollo/README.md).

---

## La petición mínima

Tres campos obligatorios y nada más:

```python
respuesta = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Resume en una frase qué hace un índice B-tree."}
    ],
)

print(respuesta.content[0].text)
```

Salida:

```
Un índice B-tree mantiene los datos ordenados en un árbol equilibrado para
que las búsquedas por rango o por igualdad se resuelvan en pocos saltos en
lugar de recorriendo la tabla entera.
```

Qué es cada cosa:

- **`model`** — el ID exacto del modelo. Se escribe literalmente como lo publica el proveedor: `claude-sonnet-5`, sin sufijos de fecha inventados (ver [Modelos y parámetros](Modelos-y-Parametros.md)).
- **`max_tokens`** — techo de tokens de salida. Obligatorio.
- **`messages`** — la conversación, como lista de turnos.

Aunque ese `respuesta.content[0].text` funciona en el ejemplo, **es el patrón que más falla en producción** y merece corregirse desde el principio: `content` es una lista de bloques de tipos distintos, y el primero no siempre es texto.

---

## La respuesta: bloques, no un string

```python
respuesta = client.messages.create(
    model="claude-opus-5",
    max_tokens=4096,
    thinking={"type": "adaptive"},
    messages=[{"role": "user", "content": "¿Por qué esta consulta hace un seq scan?"}],
)

# 1. Comprobar SIEMPRE por qué paró antes de leer el contenido
if respuesta.stop_reason == "refusal":
    return manejar_rechazo(respuesta.stop_details)
if respuesta.stop_reason == "max_tokens":
    registrar_truncamiento(respuesta.id)

# 2. Recorrer los bloques filtrando por tipo
texto = "".join(b.text for b in respuesta.content if b.type == "text")

# 3. Los metadatos que sí interesan
print(respuesta.usage.input_tokens, respuesta.usage.output_tokens)
print(respuesta.model)     # qué modelo respondió realmente
```

Con `thinking` activado, `content` empieza con bloques de tipo `thinking`; con herramientas, aparecen bloques `tool_use`. Un `content[0].text` revienta con `AttributeError` en el primer caso, y con `IndexError` si el modelo rechaza la petición y devuelve `content` vacío.

Los valores de `stop_reason` que hay que contemplar:

| Valor | Significa | Qué hacer |
|---|---|---|
| `end_turn` | terminó de forma natural | el caso normal |
| `max_tokens` | se agotó el presupuesto de salida | la respuesta está truncada: no la guardes como completa |
| `tool_use` | quiere llamar a una herramienta | ejecutarla y devolver el resultado |
| `refusal` | los filtros de seguridad declinaron | `content` puede venir vacío |

El campo `usage` es el que alimenta cualquier control de coste real, porque trae la cuenta exacta de tokens de entrada, de salida y de caché. Se desarrolla en [Coste, latencia y fiabilidad](Coste-Latencia-y-Fiabilidad.md).

---

## Los tres roles

- **`system`** — no es un mensaje, es un campo aparte de la petición. Define quién es el asistente y qué reglas sigue.
- **`user`** — lo que dice la persona (o lo que le pasa tu aplicación).
- **`assistant`** — lo que respondió el modelo en turnos anteriores.

```python
respuesta = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    system="Eres el asistente de soporte de una tienda online. Responde solo "
           "sobre pedidos, envíos y devoluciones.",
    messages=[
        {"role": "user", "content": "¿Cuánto tarda un envío?"},
        {"role": "assistant", "content": "Entre 2 y 4 días laborables en península."},
        {"role": "user", "content": "¿Y a Canarias?"},
    ],
)
```

Dos reglas del array de mensajes:

- **El primer mensaje debe ser `user`.** Empezar por `assistant` devuelve un 400.
- **El historial lo mantienes tú.** El servidor no guarda nada: si quieres que el modelo «recuerde» el turno anterior, tiene que estar en el array (ver [Cómo piensa un LLM](Como-Piensa-un-LLM.md)).

Eso hace que gestionar una conversación sea, en esencia, gestionar una lista:

```python
class Conversacion:
    def __init__(self, system: str, model: str = "claude-sonnet-5"):
        self.system = system
        self.model = model
        self.mensajes: list[dict] = []

    def enviar(self, texto_usuario: str) -> str:
        self.mensajes.append({"role": "user", "content": texto_usuario})

        respuesta = client.messages.create(
            model=self.model, max_tokens=4096,
            system=self.system, messages=self.mensajes,
        )
        texto = "".join(b.text for b in respuesta.content if b.type == "text")

        # Sin esta línea, el siguiente turno no sabrá qué contestó
        self.mensajes.append({"role": "assistant", "content": texto})
        return texto
```

Y expone el problema que llega con el uso: la lista crece sin parar. Cada turno reenvía todo el historial, así que el coste por turno sube con la longitud de la conversación. Las estrategias para manejarlo están en [Compactación de Contexto](../context-engineering/Compactacion-de-Contexto.md).

---

## *Streaming*: obligatorio para respuestas largas

Sin *streaming*, la conexión HTTP se queda muda mientras el modelo genera, y una respuesta larga puede superar el tiempo máximo de conexión y perderse entera. Con *streaming*, los tokens llegan a medida que se producen.

```python
with client.messages.stream(
    model="claude-opus-5",
    max_tokens=64000,
    messages=[{"role": "user", "content": "Escribe la guía de despliegue completa."}],
) as stream:
    for fragmento in stream.text_stream:
        print(fragmento, end="", flush=True)

    # Al terminar, el mensaje completo con usage y stop_reason
    final = stream.get_final_message()
    print(f"\n\nTokens de salida: {final.usage.output_tokens}")
```

`text_stream` entrega solo los fragmentos de texto, que es lo que quieres para pintar en pantalla. `get_final_message()` devuelve el objeto completo, así que **no pierdes `usage` ni `stop_reason` por usar *streaming***: es el motivo por el que conviene usarlo incluso cuando no muestras nada en tiempo real.

Cuándo usarlo:

- **Siempre que `max_tokens` pase de unos 16.000.** Por encima de ahí, las peticiones sin *streaming* empiezan a chocar con los *timeouts* del SDK. Con `thinking` o esfuerzo alto, el umbral llega antes de lo que parece.
- **Siempre que haya una persona esperando.** Ver el texto aparecer cambia por completo la latencia percibida, aunque el total tarde lo mismo.
- **No hace falta** en procesos por lotes sin interfaz y con respuestas cortas.

---

## Errores y reintentos

Los códigos que vas a ver y su tratamiento:

| Código | Tipo | ¿Reintentar? | Causa habitual |
|---|---|---|---|
| 400 | `invalid_request_error` | No | petición mal formada, parámetro no soportado por el modelo |
| 401 | `authentication_error` | No | clave ausente o inválida |
| 404 | `not_found_error` | No | ID de modelo inventado |
| 429 | `rate_limit_error` | **Sí** | límite de peticiones o de tokens por minuto |
| 500 / 529 | `api_error` / `overloaded_error` | **Sí** | problema del proveedor o sobrecarga |

Lo importante: **el SDK ya reintenta los reintentables con espera exponencial** (dos veces por defecto). Escribir tu propio bucle de reintentos encima suele empeorar las cosas, porque multiplica los intentos y alarga el tiempo total de espera de forma difícil de predecir. Lo que sí conviene es ajustar los parámetros y capturar por tipo:

```python
import anthropic

client = anthropic.Anthropic(max_retries=4, timeout=120.0)

try:
    respuesta = client.messages.create(...)
except anthropic.NotFoundError:
    # ID de modelo mal escrito: fallo de configuración, no reintentar
    raise
except anthropic.RateLimitError as e:
    espera = int(e.response.headers.get("retry-after", "60"))
    encolar_para_mas_tarde(espera)
except anthropic.APIStatusError as e:
    if e.status_code >= 500:
        registrar_incidencia_proveedor(e)
    else:
        raise
except anthropic.APIConnectionError:
    # fallo de red antes de recibir respuesta
    registrar_fallo_red()
```

La cadena va **de lo más específico a lo más genérico**, con una rama por cada categoría que tratas distinto. Un único `except APIStatusError` genérico es el antipatrón habitual: descarta la información que distingue un fallo transitorio de un error de configuración, y acaba reintentando peticiones que nunca van a funcionar.

Un detalle con los tiempos de espera: **el `timeout` es por intento, no total.** Con `timeout=120` y `max_retries=4`, el peor caso son unos ocho minutos de reloj. Si estás detrás de un endpoint HTTP con su propio límite, hay que dimensionar los dos juntos.

---

## Imágenes y documentos

El mismo endpoint acepta imágenes y PDF como bloques dentro del mensaje del usuario:

```python
import base64

with open("factura.pdf", "rb") as f:
    pdf_b64 = base64.standard_b64encode(f.read()).decode("utf-8")

respuesta = client.messages.create(
    model="claude-opus-5",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "document",
                "source": {"type": "base64", "media_type": "application/pdf", "data": pdf_b64},
            },
            {"type": "text", "text": "Extrae el número de factura, la fecha y el total."},
        ],
    }],
)
```

Fíjate en el orden: **el documento va antes del texto.** No es un capricho de la API, es que el modelo lee en orden y la instrucción funciona mejor cuando el material ya está delante.

Cuando el mismo fichero se va a usar en varias peticiones, subirlo una vez con la Files API y referenciarlo por `file_id` evita reenviar el base64 cada vez. Y para procesar el mismo documento con varias preguntas, la combinación con caché de *prompt* es lo que hace la diferencia de coste.

---

## Procesamiento por lotes

Cuando no hay nadie esperando —reclasificar el histórico de tickets, generar descripciones de 5.000 productos— la API de lotes procesa las peticiones de forma asíncrona **al 50 % del precio**:

```python
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

lote = client.messages.batches.create(requests=[
    Request(
        custom_id=f"producto-{p.sku}",
        params=MessageCreateParamsNonStreaming(
            model="claude-haiku-4-5",
            max_tokens=512,
            messages=[{"role": "user", "content": f"Describe este producto: {p.nombre}"}],
        ),
    )
    for p in productos
])

# Más tarde: consultar estado y recoger resultados
resultados = {}
for r in client.messages.batches.results(lote.id):
    if r.result.type == "succeeded":
        resultados[r.custom_id] = r.result.message.content[0].text
```

Dos detalles que causan bugs si se pasan por alto:

- **Los resultados llegan en cualquier orden.** Hay que indexarlos por `custom_id`, nunca por posición. Es la causa del bug clásico de este endpoint: descripciones asignadas al producto equivocado.
- **Cada resultado puede haber fallado individualmente.** Comprobar `r.result.type` antes de leer el mensaje no es opcional.

La mitad de precio a cambio de latencia (normalmente menos de una hora, con un máximo de 24) es el mejor descuento disponible sin tocar el modelo. Cualquier proceso que hoy recorra una lista llamando a la API de una en una es candidato.

---

## Desde otros lenguajes

Hay SDK oficiales para Python, TypeScript, Java, Go, Ruby, C# y PHP, y todos exponen la misma forma. En C#:

```csharp
using Anthropic;
using Anthropic.Models.Messages;

AnthropicClient client = new();   // lee ANTHROPIC_API_KEY del entorno

var mensaje = await client.Messages.Create(new MessageCreateParams
{
    Model = "claude-sonnet-5",
    MaxTokens = 1024,
    Messages = [new() { Role = Role.User, Content = "¿Qué es un índice B-tree?" }],
});

foreach (var texto in mensaje.Content.Select(b => b.Value).OfType<TextBlock>())
    Console.WriteLine(texto.Text);
```

El patrón es idéntico —cliente sin argumentos, `model` + `maxTokens` + `messages`, respuesta como lista de bloques que hay que filtrar por tipo—, así que lo que aprendas en un lenguaje se traslada. Lo único que cambia son las convenciones de nombres (`max_tokens` en Python, `MaxTokens` en C#, `maxTokens` en PHP).

---

## Buenas prácticas avanzadas

- **Nunca leas `content[0].text` en código que vaya a producción.** Es la línea que aparece en todos los ejemplos de introducción y la que rompe en cuanto activas `thinking` o herramientas, o en cuanto el modelo declina una petición y devuelve `content` vacío. Filtra por `type` y acumula: son dos caracteres más y elimina una clase entera de incidencias.
- **Comprueba `stop_reason` antes de tocar el contenido, y trata `max_tokens` como un error de datos.** Una respuesta truncada que se guarda en base de datos como completa es corrupción silenciosa: el JSON queda a medias, el resumen se corta a mitad de frase y nadie se enterará hasta que alguien lo lea. Registrar el truncamiento con el ID del mensaje te da la trazabilidad para arreglarlo después.
- **Configura `max_retries` y `timeout` en el cliente, y no escribas tu propio bucle de reintentos.** El SDK ya implementa espera exponencial sobre los códigos correctos; añadir otro bucle encima multiplica los intentos y hace el peor caso imposible de razonar. Y recuerda que el tiempo de espera es por intento: multiplica por `max_retries + 1` antes de ponerlo detrás de un endpoint con su propio límite.
- **Usa *streaming* también cuando no pintes nada en pantalla.** `get_final_message()` te devuelve el mensaje completo con `usage` y `stop_reason`, así que no pierdes nada, y a cambio te quitas de encima la clase de fallo más frustrante: la petición larga que se cae por *timeout* después de haber generado —y facturado— la respuesta entera.
- **Mueve a la API de lotes todo lo que no tenga a nadie esperando.** Es un 50 % de descuento sin cambiar de modelo ni tocar la calidad, y la mayoría de los proyectos tienen al menos un proceso —reprocesar históricos, enriquecer catálogos, generar informes nocturnos— que hoy va llamada a llamada porque nadie se planteó lo contrario. Al hacerlo, indexa por `custom_id`: el orden de los resultados no está garantizado.
- **Captura excepciones por tipo, en cadena de lo específico a lo genérico.** Un `NotFoundError` es un error de configuración que hay que hacer subir de inmediato; un 429 es una espera; un 5xx es un incidente del proveedor que merece una métrica propia. Agruparlos en un solo `except` convierte tres problemas con tres respuestas distintas en un log indistinguible.

## Recursos didácticos

- **[Referencia de la Messages API](https://platform.claude.com/docs/en/api/messages)** — la especificación campo a campo. Es la fuente de verdad cuando una respuesta no tiene la forma que esperabas.
- **[Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook)** — notebooks ejecutables con los patrones completos: *streaming*, lotes, PDF, herramientas. Copiar de aquí evita la mayoría de los errores de forma.
- **[HTTP Cats](https://http.cat/)** — la tabla de códigos HTTP ilustrada con gatos. Sirve de verdad para memorizar la diferencia entre 429 y 529, que es exactamente la que decide si reintentas o no.

---

*En resumen: una llamada a un LLM es un POST con JSON — lo que la hace distinta es que no guarda estado, que se factura por token y que la respuesta viene en bloques que hay que filtrar antes de leer.*
