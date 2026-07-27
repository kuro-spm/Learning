# Modelos y parámetros

## ¿Qué es?

Elegir modelo y ajustar sus parámetros es la decisión de configuración más barata y con más impacto de un sistema con LLMs: la misma aplicación, sin cambiar una línea de lógica, puede costar diez veces más, tardar cinco veces más o acertar mucho menos según el modelo y los parámetros que le pongas.

## ¿Por qué existe?

Un proveedor no publica «un modelo», publica una **familia**: variantes con el mismo entrenamiento base pero distinto tamaño y coste. Existe esa gama porque las tareas no son iguales: clasificar un mensaje en tres categorías y refactorizar un módulo de 4.000 líneas son problemas de dificultad radicalmente distinta, y pagar el precio del segundo para resolver el primero es tirar dinero. Los parámetros existen por lo mismo: dentro de un mismo modelo hace falta poder decir «esto es fácil, ve rápido» o «esto es difícil, piénsalo».

> Si vienes de dimensionar máquinas o contenedores, la intuición es la misma que al elegir el tamaño de una instancia: no metes la carga de un cron nocturno en la misma máquina que la API de producción, y tampoco pagas la máquina grande para el cron. La diferencia es que aquí el «tamaño» también afecta a la *calidad* del resultado, no solo a su velocidad.

## ¿Cuándo y para qué se usa?

Cada vez que escribes una llamada nueva. En una aplicación real conviven varias decisiones distintas: la tienda online usa un modelo pequeño para clasificar los mensajes del formulario de contacto, uno mediano para redactar las descripciones de producto y el grande para el asistente que investiga incidencias complejas de pedidos. Tratar «el modelo» como una constante global de la aplicación es el primer síntoma de un sistema mal dimensionado.

---

## Los tres ejes de una familia de modelos

Sea quien sea el proveedor, las variantes se ordenan por el mismo compromiso:

| Eje | Modelo pequeño | Modelo grande |
|---|---|---|
| **Capacidad** | tareas acotadas y bien definidas | razonamiento de varios pasos, trabajo agéntico largo |
| **Latencia** | respuesta casi inmediata | de segundos a minutos |
| **Coste por token** | el más bajo de la familia | varias veces superior |

En la familia de Anthropic, a fecha de esta guía:

| Modelo | ID exacto | Contexto | Entrada $/1M | Salida $/1M |
|---|---|---|---|---|
| Claude Opus 5 | `claude-opus-5` | 1M | 5,00 | 25,00 |
| Claude Sonnet 5 | `claude-sonnet-5` | 1M | 3,00 | 15,00 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 200K | 1,00 | 5,00 |

Dos detalles operativos que causan errores reales:

- **Los IDs se escriben exactamente así.** `claude-opus-5`, no `claude-opus-5-20260101`. Inventarse un sufijo de fecha devuelve un **404**, y es un error clásico al copiar código antiguo.
- **Los precios de entrada y salida son distintos** —la salida cuesta unas cinco veces más—, así que una respuesta larga pesa mucho más en la factura de lo que su tamaño sugiere. Esto se desarrolla en [Coste, latencia y fiabilidad](Coste-Latencia-y-Fiabilidad.md).

### Cómo se traduce esto a una decisión

La regla operativa que funciona: **empieza por el modelo más capaz, mide, y baja mientras la calidad se mantenga.** Al revés no funciona, porque si arrancas con el pequeño y los resultados son mediocres no sabes si el problema es el modelo o tu *prompt*, y acabas invirtiendo horas en pulir un *prompt* que ningún ajuste iba a salvar.

En código, ese «baja mientras se mantenga» se materializa en no dejar el modelo cableado en medio de la lógica:

```python
# Mal: el modelo enterrado en la función, imposible de cambiar por ruta
def clasificar(mensaje: str) -> str:
    return client.messages.create(
        model="claude-opus-5", max_tokens=16,
        messages=[{"role": "user", "content": f"Clasifica: {mensaje}"}],
    )

# Bien: modelo por caso de uso, en un único sitio
MODELOS = {
    "clasificacion": "claude-haiku-4-5",   # tarea acotada, alto volumen
    "redaccion":     "claude-sonnet-5",    # calidad media-alta, coste contenido
    "asistente":     "claude-opus-5",      # razonamiento y herramientas
}

def clasificar(mensaje: str) -> str:
    return client.messages.create(
        model=MODELOS["clasificacion"], max_tokens=16,
        messages=[{"role": "user", "content": f"Clasifica: {mensaje}"}],
    )
```

La versión de abajo permite cambiar de modelo en una ruta concreta y medir el efecto, que es exactamente el experimento que necesitas para bajar de tier con datos y no por intuición.

---

## `max_tokens`: el techo de la respuesta

`max_tokens` es obligatorio y limita **cuántos tokens de salida** puede generar el modelo. No es una sugerencia: al llegar al límite, la generación se corta en seco, incluso a mitad de una palabra.

Cómo se detecta que ha pasado:

```python
respuesta = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=100,   # deliberadamente bajo
    messages=[{"role": "user", "content": "Explica en detalle qué es un índice B-tree."}],
)

print(respuesta.stop_reason)
```

Salida:

```
max_tokens
```

`stop_reason` es el campo que hay que mirar siempre. Sus valores habituales:

| `stop_reason` | Significa |
|---|---|
| `end_turn` | terminó de forma natural — el caso normal |
| `max_tokens` | se quedó sin presupuesto de salida — respuesta truncada |
| `tool_use` | quiere llamar a una herramienta — hay que ejecutarla y continuar |
| `refusal` | los filtros de seguridad han declinado la petición |

**Leer `content` sin comprobar antes `stop_reason` es un bug esperando a ocurrir.** Con `refusal`, por ejemplo, `content` puede llegar vacío, y un `respuesta.content[0].text` revienta con un `IndexError` en producción.

Criterio para elegir el valor:

- **Clasificación o extracción de un dato corto**: ajústalo (`16`, `256`). Es un tope de seguridad contra respuestas desbocadas.
- **Respuestas normales sin *streaming***: unos `16000`. Por encima de eso, las peticiones no *streaming* empiezan a chocar con los tiempos máximos de conexión HTTP del SDK.
- **Respuestas largas**: usa *streaming* y sube el límite (hasta 128.000 en los modelos actuales). Con *streaming* la conexión no se queda muda esperando, así que el problema del *timeout* desaparece.
- **Con `thinking` activado**: recuerda que el razonamiento sale del mismo presupuesto. Añade margen o la respuesta final se truncará.

---

## Esfuerzo: la palanca de calidad frente a coste

En los modelos recientes, el control principal de «cuánto se esmera» no es la temperatura, es el **esfuerzo**. Va dentro de `output_config` —no en la raíz de la petición, un error de colocación muy común— y admite cinco niveles:

```python
respuesta = client.messages.create(
    model="claude-opus-5",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},   # low | medium | high | xhigh | max
    messages=[{"role": "user", "content": "Migra este repositorio a la nueva API de pagos."}],
)
```

Qué hace cada nivel, en la práctica:

| Nivel | Comportamiento observable | Cuándo |
|---|---|---|
| `low` | pocas llamadas a herramientas, respuestas breves, casi sin exploración | tareas cortas, subagentes, rutas sensibles a latencia |
| `medium` | equilibrio razonable | trabajo rutinario donde el coste importa |
| `high` | por defecto; explora y verifica antes de responder | la mayoría del trabajo que exige criterio |
| `xhigh` | el mejor punto para código y trabajo agéntico | refactors, migraciones, agentes largos |
| `max` | techo de capacidad, sin límite de gasto | problemas muy difíciles donde acertar importa más que el coste |

Dos ideas contraintuitivas que casi nadie aplica:

1. **Más esfuerzo puede salir más barato en trabajo agéntico.** Un `xhigh` que planifica bien y resuelve la tarea en 4 turnos gasta menos que un `medium` que da 15 vueltas. El coste que hay que comparar es el de la tarea completa, no el de una llamada.
2. **Los niveles bajos rinden mejor de lo que la gente supone en los modelos nuevos.** Heredar el nivel que te iba bien en un modelo anterior es casi siempre dejar dinero sobre la mesa: al cambiar de modelo, vuelve a barrer los niveles con tus propias evaluaciones.

---

## `thinking`: cuándo dejar que razone

Ya visto en [Cómo piensa un LLM](Como-Piensa-un-LLM.md); aquí importa el detalle operativo, porque es donde más ha cambiado la API y donde más código antiguo se rompe.

```python
# Forma actual — el modelo decide cuánto razonar
thinking={"type": "adaptive"}

# Forma antigua — presupuesto fijo de tokens de razonamiento.
# En los modelos actuales devuelve un error 400.
thinking={"type": "enabled", "budget_tokens": 10000}
```

Tres reglas de la versión actual:

- **`{"type": "adaptive"}` es la única forma recomendada.** El presupuesto fijo (`budget_tokens`) está eliminado en los modelos nuevos: quien controla la profundidad es `effort`.
- **En `claude-opus-5` el razonamiento está activado por defecto.** Omitir el parámetro `thinking` **no** lo desactiva. Si vienes de un modelo donde omitirlo significaba «sin razonamiento», tu coste y tu `max_tokens` cambian de golpe sin que hayas tocado nada.
- **El texto del razonamiento no se devuelve por defecto.** Llega un bloque `thinking` con el texto vacío. Si tu interfaz muestra el razonamiento al usuario, hay que pedirlo explícitamente:

```python
thinking={"type": "adaptive", "display": "summarized"}
```

Sin `display`, un panel que muestre el razonamiento se queda en blanco y da la impresión de que la aplicación está colgada durante todo el rato que el modelo está pensando.

---

## El *system prompt*: dónde van las reglas del sistema

Las instrucciones que definen el comportamiento del asistente no van en el mensaje del usuario, van en un campo aparte: `system`. Es un canal distinto y con más autoridad.

```python
respuesta = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    system=(
        "Eres el asistente de soporte de una tienda online. "
        "Responde solo sobre pedidos, envíos y devoluciones. "
        "Si preguntan otra cosa, redirige amablemente al tema."
    ),
    messages=[{"role": "user", "content": "¿Puedes escribirme un poema?"}],
)
```

Meter esas mismas reglas dentro del mensaje del usuario funciona *casi* igual y es peor por dos razones: pierden prioridad frente a lo que el usuario diga después, y —más importante— si el texto del usuario se concatena con tus reglas, un usuario puede escribir instrucciones que se lean como reglas del sistema. Separar los canales es la primera defensa contra la *prompt injection*.

**Además, el `system` es el bloque estable de tu petición.** Eso lo convierte en el candidato natural para la caché de *prompt*, la optimización de coste con mejor relación esfuerzo/beneficio:

```python
respuesta = client.messages.create(
    model="claude-opus-5",
    max_tokens=4096,
    system=[{
        "type": "text",
        "text": SYSTEM_PROMPT_LARGO,          # instrucciones, ejemplos, esquemas
        "cache_control": {"type": "ephemeral"},
    }],
    messages=[{"role": "user", "content": pregunta}],
)

print(respuesta.usage.cache_read_input_tokens)   # >0 significa que la caché funcionó
```

Con `cache_control`, la parte estable se procesa una vez y las llamadas siguientes la leen de caché a una décima parte del precio. `usage.cache_read_input_tokens` es la métrica que confirma que está funcionando: si sale 0 llamada tras llamada, algo está cambiando el prefijo (una fecha, un UUID) e invalidando la caché. El mecanismo completo está en [Prompt Caching](../context-engineering/Prompt-Caching.md).

Una consecuencia de diseño que se deriva de esto: **no interpoles datos variables al principio del `system`**. Un `f"Fecha actual: {datetime.now()}"` en la primera línea invalida la caché en cada petición. Los datos que cambian van al final, en los mensajes.

---

## Consultar capacidades en tiempo de ejecución

Las tablas de precios y contextos envejecen. Cuando necesites el dato real —para mostrarlo en un panel, para elegir modelo por capacidad, para validar configuración al arrancar— hay una API para eso:

```python
modelo = client.models.retrieve("claude-opus-5")

print(modelo.display_name)        # "Claude Opus 5"
print(modelo.max_input_tokens)    # 1000000  -> ventana de contexto
print(modelo.max_tokens)          # 128000   -> techo de salida
print(modelo.capabilities["image_input"]["supported"])   # True
```

Esto permite, por ejemplo, validar al arrancar que el modelo configurado soporta lo que tu código va a pedirle, en vez de descubrirlo con un 400 en la primera petición de un usuario:

```python
caps = client.models.retrieve(MODELOS["asistente"]).capabilities
if not caps["thinking"]["types"]["adaptive"]["supported"]:
    raise RuntimeError("El modelo configurado no soporta thinking adaptativo")
```

---

## Buenas prácticas avanzadas

- **Trata la elección de modelo como una decisión por ruta, nunca como una constante global.** Un mapa `caso_de_uso -> modelo` en configuración te da dos cosas que un modelo cableado no: poder abaratar una ruta concreta con una medición, y poder subir de tier solo donde la calidad lo justifique. Es un cambio de cinco minutos que se paga solo el primer mes.
- **Vuelve a calibrar `effort` cada vez que cambies de modelo, con tus propias evaluaciones.** Los niveles no son equivalentes entre generaciones: un nivel que era el punto óptimo en un modelo puede ser derrochador en el siguiente, y el intervalo de calidad-por-token se ha movido en cada lanzamiento reciente. Heredar la configuración sin medir es el desperdicio más silencioso que hay.
- **Comprueba `stop_reason` antes de tocar `content`, siempre.** Es una línea de código y evita dos clases de incidencia: la respuesta truncada que se guarda en base de datos como si estuviera completa, y el `content[0]` que revienta cuando el modelo declina la petición.
- **Congela el prefijo de tus peticiones y pon la caché ahí.** El error que arruina la caché no es la falta de `cache_control`, es un dato dinámico al principio del `system`: una fecha, un ID de sesión, un `json.dumps` sin `sort_keys=True`. Cualquiera de los tres invalida el prefijo en cada llamada y te deja pagando el sobrecoste de escritura de caché sin llegar a leerla nunca.
- **No cambies de modelo ni de conjunto de herramientas a mitad de conversación.** Las cachés son por modelo, y las herramientas se serializan al principio del *prompt*: cambiar cualquiera de las dos invalida todo el contexto cacheado de la conversación. Si necesitas un modelo distinto para una subtarea, lánzala como llamada aparte en lugar de conmutar el modelo del bucle principal.
- **Ten un plan para `stop_reason: "refusal"`.** En los modelos con salvaguardas reforzadas, trabajo legítimo de seguridad o de ciencias de la vida puede activar un filtro. Llega como un **200** con `content` vacío, no como una excepción: si no lo contemplas, se manifiesta como un bug raro e intermitente en vez de como lo que es.

## Recursos didácticos

- **[Models overview de Anthropic](https://platform.claude.com/docs/en/about-claude/models/overview)** — la tabla viva de IDs, contextos y precios. Consúltala antes de fijar un modelo en configuración.
- **[Guía de migración de modelos](https://platform.claude.com/docs/en/about-claude/models/migration-guide)** — qué se rompe al pasar de un modelo a otro, parámetro a parámetro. Cuando algo devuelva un 400 tras cambiar de modelo, la respuesta está aquí.
- **[Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook)** — notebooks ejecutables con configuraciones reales por tipo de tarea; muy útil para ver qué modelo y qué parámetros usa la gente en cada escenario.

---

*En resumen: elige el modelo por ruta y no por proyecto, empieza por arriba y baja midiendo, y recuerda que en los modelos actuales la palanca de calidad es el esfuerzo, no la temperatura.*
