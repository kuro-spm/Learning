# Coste, latencia y fiabilidad

## ¿Qué es?

Es la parte operativa de un sistema con LLMs: cuánto cuesta cada llamada, cuánto tarda y qué pasa cuando falla. Los tres van juntos porque casi todas las decisiones mueven los tres a la vez, normalmente en direcciones opuestas.

## ¿Por qué existe?

Porque una llamada a un LLM rompe tres supuestos con los que estás acostumbrado a trabajar:

- **El coste es variable y proporcional al uso.** Una consulta a tu base de datos cuesta lo mismo se ejecute mil veces o un millón. Una llamada a un LLM se factura por token, así que la factura escala con el tráfico y con la verbosidad de las respuestas.
- **La latencia se mide en segundos, no en milisegundos.** Y con razonamiento activado, en decenas de segundos o minutos. Eso no cabe en un endpoint HTTP síncrono sin pensarlo.
- **Los fallos son de una clase nueva.** Además de los 5xx habituales, existen la respuesta truncada, el rechazo por filtros de seguridad y la salida que no cumple el formato. Los tres devuelven código 200.

> Si has operado un servicio que consume una API de terceros con cuota, el marco mental es el mismo: cuota, reintentos, degradación y observabilidad. Lo distinto es que aquí la unidad de coste no es la petición sino el token, y eso cambia dónde están las palancas.

## ¿Cuándo y para qué se usa?

Antes de que el sistema esté en producción, no después. Los tres errores clásicos —descubrir la factura a final de mes, descubrir que el endpoint da *timeout* con carga real, y descubrir que un 429 tira la funcionalidad entera— se previenen con decisiones que cuestan poco al principio y mucho después.

---

# Parte 1 — Coste

## Cómo se factura

Dos precios distintos por millón de tokens, y **la salida cuesta unas cinco veces más que la entrada**:

| Modelo | Entrada $/1M | Salida $/1M |
|---|---|---|
| Claude Opus 5 | 5,00 | 25,00 |
| Claude Sonnet 5 | 3,00 | 15,00 |
| Claude Haiku 4.5 | 1,00 | 5,00 |

Esa asimetría es la primera cosa que hay que internalizar: **una respuesta larga pesa mucho más en la factura de lo que su tamaño sugiere.** Un *prompt* de 2.000 tokens que produce 500 de salida cuesta, en Opus, 10.000 + 12.500 = 22.500 unidades de precio: la salida, que es la cuarta parte del texto, es más de la mitad del coste.

## Estimar antes de construir

La estimación se hace con dos números y una multiplicación, y hay que hacerla antes de escribir la aplicación:

```python
PRECIO = {  # $ por millón de tokens (entrada, salida)
    "claude-opus-5":   (5.00, 25.00),
    "claude-sonnet-5": (3.00, 15.00),
    "claude-haiku-4-5":(1.00,  5.00),
}

def coste_mensual(modelo: str, tokens_entrada: int, tokens_salida: int,
                  peticiones_dia: int) -> float:
    pe, ps = PRECIO[modelo]
    por_peticion = (tokens_entrada / 1e6) * pe + (tokens_salida / 1e6) * ps
    return por_peticion * peticiones_dia * 30

# Asistente de soporte: prompt de sistema largo + historial
print(coste_mensual("claude-opus-5",   tokens_entrada=8000, tokens_salida=400, peticiones_dia=2000))
print(coste_mensual("claude-sonnet-5", tokens_entrada=8000, tokens_salida=400, peticiones_dia=2000))
print(coste_mensual("claude-haiku-4-5",tokens_entrada=8000, tokens_salida=400, peticiones_dia=2000))
```

Salida:

```
3000.0
1800.0
600.0
```

Tres mil dólares al mes frente a seiscientos, por el mismo tráfico. Ese cálculo, hecho al principio, es lo que convierte «qué modelo usamos» de una preferencia en una decisión de producto. Y para los tokens de entrada no adivines: cuéntalos con el endpoint de conteo sobre un *prompt* real (ver [Cómo piensa un LLM](Como-Piensa-un-LLM.md)).

## Medir lo que gastas de verdad

Cada respuesta trae cuatro números, y los cuatro importan:

```python
u = respuesta.usage
print(u.input_tokens)               # entrada NO cacheada, a precio completo
print(u.cache_creation_input_tokens)# escritos en caché, a ~1,25x
print(u.cache_read_input_tokens)    # leídos de caché, a ~0,1x
print(u.output_tokens)              # salida, al precio de salida
```

El error de contabilidad más común: **creer que `input_tokens` es el tamaño del *prompt***. No lo es —es solo la parte que no venía de caché. El *prompt* completo es la suma de los tres primeros. Si tu agente lleva veinte iteraciones y `input_tokens` marca 4.000, el resto se sirvió de caché; mirar solo ese campo te da una imagen falsa del consumo.

## Las cinco palancas, por orden de impacto

### 1. Caché de *prompt* (hasta 90 % de ahorro en la parte estable)

Es, con diferencia, la de mejor relación esfuerzo/resultado. La parte estable del *prompt* se procesa una vez y las llamadas siguientes la leen a una décima parte del precio:

```python
respuesta = client.messages.create(
    model="claude-opus-5",
    max_tokens=4096,
    system=[{
        "type": "text",
        "text": SYSTEM_PROMPT + CATALOGO_DE_CATEGORIAS + EJEMPLOS,  # estable
        "cache_control": {"type": "ephemeral"},
    }],
    messages=[{"role": "user", "content": mensaje}],   # variable, después
)

assert respuesta.usage.cache_read_input_tokens > 0    # verificar que funciona
```

Tres reglas que hay que entender o la caché no funcionará:

- **Es una coincidencia de prefijo, byte a byte.** Cualquier cambio invalida todo lo que va después. El orden de renderizado es `tools` → `system` → `messages`, así que lo estable va delante y lo variable detrás.
- **Los invalidadores silenciosos son la causa del 90 % de los fallos.** Un `datetime.now()` en el `system`, un ID de sesión interpolado al principio, un `json.dumps()` sin `sort_keys=True`, o un conjunto de herramientas que varía por usuario. Cualquiera de los cuatro deja el prefijo distinto en cada llamada.
- **La economía tiene umbral.** Escribir en caché cuesta ~1,25× (TTL de 5 minutos) o ~2× (TTL de una hora). Con el TTL corto, se amortiza a la segunda llamada; con el largo, hace falta la tercera. Para tráfico continuo el TTL corto sobra, porque cada petición renueva la entrada.

Y la verificación no es opcional: **si `cache_read_input_tokens` es 0 llamada tras llamada, estás pagando el sobrecoste de escritura sin leer nunca**, es decir, la caché te está costando dinero en lugar de ahorrarlo. El desarrollo completo está en [Prompt Caching](../context-engineering/Prompt-Caching.md).

### 2. Modelo por ruta

La segunda palanca es no usar el mismo modelo para todo. Con un clasificador barato delante, el modelo caro solo atiende lo que lo necesita:

```python
def atender(mensaje: str) -> str:
    tipo = clasificar(mensaje, modelo="claude-haiku-4-5")   # ~0,1 ¢ por llamada
    if tipo in ("saludo", "agradecimiento", "fuera_de_alcance"):
        return RESPUESTAS_FIJAS[tipo]                       # 0 ¢: ni siquiera hay llamada
    if tipo == "consulta_simple":
        return responder(mensaje, modelo="claude-sonnet-5")
    return resolver_con_herramientas(mensaje, modelo="claude-opus-5")
```

En un asistente de soporte real, la mayoría del tráfico cae en las dos primeras ramas. La reducción de coste suele ser de un orden de magnitud, y la calidad percibida sube porque las respuestas triviales son instantáneas.

### 3. Procesamiento por lotes (50 %)

Cualquier proceso sin nadie esperando —reprocesar históricos, enriquecer catálogos, informes nocturnos— va a la API de lotes al 50 % de precio. Es el descuento más grande disponible sin tocar nada más que el endpoint (ver [Llamadas a la API](Llamadas-a-la-API.md)).

### 4. Acortar la salida

Como la salida cuesta cinco veces más, es donde más rinde cada token que quitas:

```python
system = (SYSTEM_PROMPT +
          "\n\nResponde en un máximo de tres frases. Sin preámbulos ni resúmenes "
          "de la pregunta: ve directo a la respuesta.")
```

Y para clasificación o extracción, un `max_tokens` ajustado (`16`, `64`) además de la instrucción: es un tope duro contra una respuesta desbocada.

### 5. Acortar la entrada

La última porque es la más laboriosa y la que menos rinde por unidad de esfuerzo. Aun así, dos cosas valen la pena: no meter documentos completos cuando basta el fragmento relevante ([RAG](../context-engineering/RAG.md)), y compactar el historial de conversaciones largas.

## El coste crece de forma cuadrática

El efecto que sorprende en la primera factura: en una conversación o en un agente, **cada turno reenvía todo el historial**. Con turnos de tamaño parecido, el coste acumulado de entrada crece con el cuadrado del número de turnos.

```text
Turno 1:  1.000 tokens de entrada
Turno 2:  2.000
Turno 5:  5.000
Turno 20: 20.000
                     Total acumulado: ~210.000, no 20.000
```

Consecuencias directas: un agente de veinte iteraciones no cuesta veinte veces la primera llamada, cuesta mucho más; y la caché de *prompt* no es una optimización opcional en conversaciones largas, es lo que las hace viables.

## Atribuir y alertar

Sin atribución no se puede optimizar. Registra el consumo con las etiquetas que te van a hacer falta:

```python
logger.info("llm.uso", extra={
    "funcionalidad": "soporte_chat",
    "usuario_id": usuario.id,
    "modelo": respuesta.model,
    "tokens_entrada": respuesta.usage.input_tokens,
    "tokens_cache_leidos": respuesta.usage.cache_read_input_tokens,
    "tokens_salida": respuesta.usage.output_tokens,
    "coste_estimado": estimar_coste(respuesta),
})
```

Con eso puedes responder a las tres preguntas que siempre acaban surgiendo: qué funcionalidad se lleva el presupuesto, si hay un usuario consumiendo de forma anómala, y si la caché está funcionando. Y pon una alerta sobre el gasto diario, no sobre el mensual: enterarse el día 3 permite reaccionar; enterarse el día 30, no.

---

# Parte 2 — Latencia

## De qué depende

Por orden de peso:

1. **Los tokens de salida.** Es el factor dominante y con diferencia: la generación es secuencial, token a token. Una respuesta del doble de larga tarda aproximadamente el doble.
2. **El razonamiento.** Con `thinking` activado, los tokens de razonamiento son tokens generados: cuentan íntegros en el tiempo.
3. **El tamaño del modelo.** Uno pequeño genera bastante más rápido por token.
4. **Los tokens de entrada.** Influyen mucho menos, porque el procesamiento del *prompt* es paralelizable. Y la parte cacheada casi no cuenta.

De ahí sale la regla contraintuitiva: **para bajar la latencia, recorta la salida antes que la entrada.** Un *prompt* de 20.000 tokens con respuesta de 100 es mucho más rápido que uno de 500 con respuesta de 2.000.

## Percibida frente a total

Con *streaming*, el usuario ve el primer token en cientos de milisegundos aunque la respuesta completa tarde treinta segundos. No es maquillaje: cambia por completo la experiencia, y **es la mejora de latencia más barata que existe** porque no requiere renunciar a nada.

Para cargas donde la velocidad de generación es crítica hay además modos acelerados que ejecutan el mismo modelo con más tokens por segundo a precio superior; y siempre queda bajar `effort`, que reduce cuánto razona y explora antes de responder.

## Arquitectura: lo que tarda minutos no va en un endpoint síncrono

Un agente puede tardar varios minutos. Metido detrás de un endpoint HTTP con un límite de 30 segundos, es un fallo garantizado. El patrón correcto es asíncrono:

```python
@app.post("/api/analisis")
async def iniciar(peticion: PeticionAnalisis) -> dict:
    tarea_id = cola.encolar(analizar, peticion)      # devuelve de inmediato
    return {"tarea_id": tarea_id, "estado": "en_proceso"}

@app.get("/api/analisis/{tarea_id}")
async def consultar(tarea_id: str) -> dict:
    return cola.estado(tarea_id)                     # el cliente consulta o recibe un webhook
```

Y hay que dimensionar los tiempos de espera **en cadena**: si tu cliente HTTP corta a los 60 segundos, el SDK tiene un *timeout* de 120 y reintenta dos veces, la petición se cancela desde arriba mientras abajo sigue reintentando. Los tres números tienen que ser coherentes entre sí.

---

# Parte 3 — Fiabilidad

## Los seis modos de fallo

| Fallo | Cómo se manifiesta | Tratamiento |
|---|---|---|
| **429** rate limit | excepción, con cabecera `retry-after` | reintento con espera (lo hace el SDK) |
| **5xx / 529** | excepción | reintento con espera |
| **Timeout** | excepción de conexión | reintento; revisar si `max_tokens` es realista |
| **Truncado** | **200** con `stop_reason: "max_tokens"` | no guardar como completo; subir el límite |
| **Rechazo** | **200** con `stop_reason: "refusal"` y `content` vacío | mensaje al usuario o modelo alternativo |
| **Salida inválida** | 200 con contenido que no cumple el formato | validar y reintentar una vez, o pasar a revisión |

Los tres últimos son los peligrosos porque **son códigos 200**: no lanzan excepción, no aparecen en las métricas de error y se cuelan hasta la base de datos si nadie los comprueba.

## Reintentos: usa los del SDK

El SDK ya reintenta 408, 409, 429 y 5xx con espera exponencial. Escribir otro bucle encima multiplica los intentos y hace el peor caso imposible de razonar:

```python
client = anthropic.Anthropic(max_retries=4, timeout=120.0)
```

Con esa configuración, el peor caso son ~10 minutos de reloj (5 intentos × 120 s). Ese número tiene que caber en el presupuesto de tiempo de quien te llama.

Y una regla que se olvida: **si la operación tiene efectos, hazla idempotente.** Un reintento tras un *timeout* puede duplicar el trabajo, y con LLM esto es especialmente traicionero porque el *timeout* puede llegar después de que la respuesta se generara —y se facturara— entera.

## Degradación en cascada

Un fallo no debería tirar la funcionalidad. Los niveles, del mejor al peor resultado aceptable:

```python
def responder(mensaje: str) -> str:
    try:
        return llamar(modelo="claude-opus-5", mensaje=mensaje)
    except anthropic.RateLimitError:
        # 1. mismo trabajo, modelo alternativo (distinta cuota)
        try:
            return llamar(modelo="claude-sonnet-5", mensaje=mensaje)
        except anthropic.APIError:
            pass
    except anthropic.APIStatusError as e:
        if e.status_code < 500:
            raise            # error de configuración: no lo tapes
    # 2. respuesta útil sin LLM
    if respuesta_cacheada := cache.buscar_similar(mensaje):
        return respuesta_cacheada
    # 3. degradación honesta
    return "Ahora mismo no puedo procesar tu consulta. Te derivo con una persona."

```

Fíjate en el `raise` de la rama 4xx: **un error de cliente no se degrada, se propaga.** Si el modelo no existe o el parámetro no está soportado, un *fallback* silencioso convierte un error de configuración en una anomalía de calidad que nadie sabrá diagnosticar.

Y para el caso concreto del rechazo por filtros de seguridad, existe además un mecanismo nativo —el parámetro `fallbacks`— que reintenta en otro modelo dentro de la misma llamada, sin que tengas que orquestarlo.

## Qué registrar

Lo mínimo para poder diagnosticar en producción:

```python
logger.info("llm.llamada", extra={
    "traza_id": traza_id,
    "funcionalidad": "soporte_chat",
    "modelo": respuesta.model,          # el que respondió, no el que pediste
    "stop_reason": respuesta.stop_reason,
    "latencia_ms": latencia_ms,
    "tokens_entrada": respuesta.usage.input_tokens,
    "tokens_salida": respuesta.usage.output_tokens,
    "request_id": respuesta._request_id,  # para abrir un ticket con el proveedor
})
```

Dos campos que casi nadie registra y que son los que más falta hacen: **`stop_reason`**, porque es la única forma de detectar la degradación silenciosa por truncamiento, y **`request_id`**, porque es lo que el proveedor necesita para investigar un caso concreto. Los principios generales están en la colección de [observabilidad](../../devops/observabilidad/Observabilidad.md).

---

## Buenas prácticas avanzadas

- **Verifica que la caché de *prompt* está funcionando, no supongas que sí.** `cache_read_input_tokens` en 0 llamada tras llamada significa que estás pagando el sobrecoste de escritura sin leer nunca: la caché te cuesta dinero en vez de ahorrarlo. La causa es casi siempre un invalidador silencioso —una fecha, un UUID, un `json.dumps` sin ordenar— al principio del prefijo, y se encuentra comparando los bytes del *prompt* renderizado entre dos peticiones.
- **Recorta la salida antes que la entrada, tanto para coste como para latencia.** Es contraintuitivo porque la entrada suele ser mucho más grande, pero la salida cuesta cinco veces más por token y es el factor dominante del tiempo de generación. Una instrucción de brevedad más un `max_tokens` ajustado da más ahorro que horas de poda del *prompt*.
- **Suma `input_tokens + cache_creation + cache_read` para conocer el tamaño real del *prompt*.** Leer solo `input_tokens` es el error de contabilidad más frecuente: en un sistema con caché bien puesta, ese campo puede ser una fracción del total y te da una imagen tranquilizadora y falsa de cuánto contexto estás mandando.
- **Trata `stop_reason` como una métrica de calidad de primer nivel, no como un detalle.** Un `max_tokens` en el 3 % de las peticiones es un 3 % de respuestas truncadas guardándose como completas: JSON a medias, resúmenes cortados, datos corruptos que nadie relaciona con la causa. Es la degradación más silenciosa que existe porque llega con código 200 y no aparece en ninguna métrica de error.
- **Propaga los 4xx y degrada solo los transitorios.** Un `NotFoundError` por un ID de modelo mal escrito o un 400 por un parámetro no soportado son errores de configuración: si el *fallback* los captura, tu sistema funciona «bien» con el modelo alternativo y nadie descubre el problema hasta que alguien audita la factura. Degrada 429 y 5xx; lo demás debe hacer ruido.
- **Dimensiona los tiempos de espera en cadena, de arriba abajo.** El *timeout* del SDK es por intento, así que el peor caso real es `timeout × (max_retries + 1)`. Si eso supera el límite del cliente HTTP que te llama, tendrás peticiones canceladas desde arriba mientras abajo se siguen generando —y facturando— respuestas que nadie va a leer.
- **Etiqueta el consumo por funcionalidad desde la primera llamada, y alerta sobre el gasto diario.** Sin atribución, la única pregunta que puedes responder es «cuánto gastamos en total», que no permite optimizar nada. Y una alerta mensual llega cuando el dinero ya está gastado: el umbral útil es diario, porque da margen a reaccionar el día 3 en lugar del 30.
- **Mueve a lotes todo lo que no tenga a nadie esperando.** Un 50 % de descuento sin tocar el modelo ni la calidad, y casi todos los proyectos tienen al menos un proceso que hoy va llamada a llamada solo porque nadie se lo planteó. Es el ahorro más grande que se consigue sin negociar calidad.

## Recursos didácticos

- **[Prompt caching (Anthropic)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)** — el mecanismo con la lista de invalidadores silenciosos. Léelo entero antes de poner tu primer `cache_control`; ahorra la semana de descubrir a base de facturas que la caché no estaba funcionando.
- **[Batch processing (Anthropic)](https://platform.claude.com/docs/en/build-with-claude/batch-processing)** — el 50 % de descuento, con el flujo completo de creación, consulta y recogida de resultados.
- **[Rate limits (Anthropic)](https://platform.claude.com/docs/en/api/rate-limits)** — los límites por nivel y las cabeceras de cuota. Consultarlo antes de desplegar evita que el primer día de tráfico real sea también el primer día de 429.
- **[Release It! (Michael Nygard)](https://pragprog.com/titles/mnee2/release-it-second-edition/)** — el libro de referencia sobre patrones de estabilidad: *timeouts*, *circuit breakers*, degradación. Anterior a los LLM y directamente aplicable, porque los problemas son los de siempre.

---

*En resumen: la salida cuesta cinco veces más que la entrada y domina la latencia, la caché de prompt es la palanca de mayor retorno, y los fallos que más daño hacen llegan con código 200.*
