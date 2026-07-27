# Agentes con LLMs

## ¿Qué es?

Un agente es un LLM en un bucle: recibe un objetivo, decide qué herramienta usar, ve el resultado, decide otra vez, y así hasta que considera que ha terminado. La característica que lo define no es que use herramientas, es que **el número y el orden de los pasos los decide el modelo en tiempo de ejecución**, no tú al escribir el código.

## ¿Por qué existe?

Porque hay problemas cuyos pasos no se pueden escribir de antemano. «Extrae los datos de esta factura» tiene una secuencia fija y se resuelve con un flujo normal. «Averigua por qué este pedido no se ha enviado» no: puede requerir consultar el pedido, ver el estado del pago, revisar el stock de una línea concreta, buscar el envío en el sistema del transportista o ninguna de esas cosas, dependiendo de lo que se vaya descubriendo.

Cuando el árbol de posibilidades es demasiado ancho para codificarlo, delegar la decisión al modelo es lo que hace el problema abordable. Y a cambio pagas un precio muy concreto: coste variable, latencia impredecible y un sistema mucho más difícil de depurar.

> Si has escrito un `switch` de estados que fue creciendo hasta ser inmanejable porque cada caso nuevo abría tres ramas más, un agente es la alternativa a ese `switch`. La ventaja es que no tienes que enumerar los casos; la desventaja es que ya no puedes leer el código y saber qué va a pasar.

## ¿Cuándo y para qué se usa?

Casi nunca, y esa es la parte importante. La escala de opciones, de menos a más complejidad, con la regla de quedarse en el escalón más bajo que resuelva el problema:

| Nivel | Qué es | Cuándo |
|---|---|---|
| **Una llamada** | *prompt* → respuesta | clasificar, extraer, resumir, traducir, generar texto |
| **Cadena fija** | varias llamadas en un orden que tú escribes | procesos de pasos conocidos (extraer → validar → resumir) |
| **Enrutado** | una llamada clasifica y decide qué rama seguir | tipos de petición con tratamientos distintos |
| **Flujo con herramientas** | tú controlas el bucle y las llamadas | la mayoría de los casos que «parecen» necesitar un agente |
| **Agente** | el modelo controla el bucle | el número de pasos depende de lo que se descubra |

La mayoría de los proyectos que empiezan con «vamos a hacer un agente» acaban funcionando mejor —más baratos, más rápidos y depurables— como una cadena fija con una o dos llamadas al modelo en los puntos que requieren criterio.

### La prueba de las cuatro preguntas

Antes de construir un agente, las cuatro condiciones que deben cumplirse **todas**:

1. **Complejidad** — ¿la tarea es de varios pasos y realmente no se puede especificar de antemano? «Convierte este documento de diseño en un *pull request*» sí; «extrae el título de este PDF» no.
2. **Valor** — ¿el resultado justifica un coste y una latencia varias veces superiores?
3. **Viabilidad** — ¿el modelo es capaz de hacer esta clase de tarea? Un agente no arregla una tarea que el modelo falla en una sola llamada; la repite muchas veces.
4. **Coste del error** — ¿se pueden detectar y revertir los errores? Con tests, revisión o *rollback*, sí. Si cada paso equivocado envía un email a un cliente, no.

Un «no» en cualquiera de las cuatro significa bajar un escalón.

---

## Antes del agente: los patrones de flujo

Estos patrones resuelven la mayoría de los casos y son deterministas, así que se depuran leyendo el código.

**Cadena** — cada paso alimenta al siguiente. El LLM se usa solo donde hace falta criterio:

```python
def procesar_factura(pdf: bytes) -> Factura:
    texto  = extraer_texto(pdf)                # código
    datos  = extraer_campos(texto)             # LLM con salida estructurada
    validar_reglas_negocio(datos)              # código
    return guardar(datos)                      # código
```

**Enrutado** — una llamada barata clasifica, y cada rama usa el modelo y el *prompt* adecuados:

```python
TIPOS = ["incidencia_pedido", "consulta_producto", "devolucion", "otro"]

def atender(mensaje: str) -> str:
    tipo = clasificar(mensaje, modelo="claude-haiku-4-5")   # barato y rápido
    if tipo == "incidencia_pedido":
        return resolver_incidencia(mensaje, modelo="claude-opus-5")   # necesita herramientas
    if tipo == "consulta_producto":
        return responder_con_catalogo(mensaje, modelo="claude-sonnet-5")
    return escalar_a_humano(mensaje)
```

El ahorro es grande: solo la rama que de verdad necesita razonamiento paga el modelo caro, y el 70 % del tráfico se resuelve con el barato.

**Paralelización** — subtareas independientes a la vez. Si hay que revisar un diff desde tres perspectivas (corrección, seguridad, rendimiento), son tres llamadas concurrentes, no un agente dando tres vueltas.

**Evaluador-optimizador** — una llamada genera, otra critica con criterios explícitos, la primera corrige. Funciona bien cuando existe un criterio de calidad claro y el número de vueltas está acotado.

---

## El bucle mínimo de un agente

Cuando de verdad hace falta, así es por dentro:

```python
def agente(objetivo: str, herramientas: list[dict], max_iteraciones: int = 15) -> str:
    mensajes = [{"role": "user", "content": objetivo}]

    for iteracion in range(max_iteraciones):
        respuesta = client.messages.create(
            model="claude-opus-5",
            max_tokens=8192,
            thinking={"type": "adaptive"},
            output_config={"effort": "high"},
            tools=herramientas,
            messages=mensajes,
        )

        mensajes.append({"role": "assistant", "content": respuesta.content})

        # El modelo ha dejado de pedir herramientas: ha terminado
        if respuesta.stop_reason != "tool_use":
            return "".join(b.text for b in respuesta.content if b.type == "text")

        resultados = []
        for bloque in respuesta.content:
            if bloque.type != "tool_use":
                continue
            try:
                salida, es_error = ejecutar(bloque.name, bloque.input), False
            except Exception as e:                      # un fallo no rompe el bucle
                salida, es_error = f"Error: {e}", True
            resultados.append({
                "type": "tool_result",
                "tool_use_id": bloque.id,
                "content": str(salida)[:20_000],        # cota al tamaño del resultado
                "is_error": es_error,
            })

        mensajes.append({"role": "user", "content": resultados})

    return "No se pudo completar la tarea en el número máximo de iteraciones."
```

Son treinta líneas, y ahí está todo un agente. Los cuatro detalles que no son opcionales:

- **`max_iteraciones`.** Sin tope, un modelo que se atasca puede dar vueltas indefinidamente gastando dinero. No es una hipótesis: es el modo de fallo número uno.
- **Los errores de herramienta se devuelven, no se propagan.** El `try/except` que convierte la excepción en un `tool_result` con `is_error` es lo que permite al modelo reintentar con otro parámetro. Dejar que la excepción suba mata la tarea entera por un fallo recuperable.
- **El tamaño del resultado está acotado.** Una herramienta que devuelve 300.000 tokens llena el contexto en una iteración. El recorte —o mejor, guardar el resultado en un fichero y devolver la ruta— es lo que hace viables las tareas largas.
- **Salir por `stop_reason != "tool_use"`, no por «parece que ha acabado».** La condición de terminación es explícita y viene de la API.

---

## Los controles que no se negocian

Un agente sin límites es una factura sin límites. Cuatro capas, de la más simple a la más elaborada:

**1. Tope de iteraciones.** Ya en el bucle de arriba.

**2. Presupuesto de tokens.** Acumular el consumo y cortar. Nótese que hay que sumar de todas las iteraciones, no leer la última:

```python
gastados = 0
PRESUPUESTO = 200_000

for iteracion in range(max_iteraciones):
    respuesta = client.messages.create(...)
    gastados += respuesta.usage.input_tokens + respuesta.usage.output_tokens
    if gastados > PRESUPUESTO:
        return f"Presupuesto agotado tras {iteracion + 1} iteraciones."
```

También existe la vía nativa: `output_config.task_budget` le comunica al modelo cuántos tokens tiene para la tarea completa, de modo que **se administre él** y termine ordenadamente en lugar de ser cortado a mitad. Es distinto de `max_tokens`, que es un techo por respuesta del que el modelo no es consciente.

**3. Tiempo máximo de reloj.** Un agente que tarda veinte minutos detrás de un endpoint HTTP con un límite de 30 segundos es un fallo garantizado. Si la tarea es larga, la arquitectura correcta es asíncrona: se acepta el trabajo, se devuelve un identificador y el cliente consulta el estado.

**4. Aprobación humana en lo irreversible.** La clasificación de herramientas en solo lectura y de escritura, con confirmación obligatoria en las segundas:

```python
HERRAMIENTAS_SEGURAS = {"consultar_pedido", "consultar_stock", "buscar_producto"}

def ejecutar(nombre: str, entrada: dict):
    if nombre not in HERRAMIENTAS_SEGURAS:
        if not pedir_confirmacion(nombre, entrada):
            return "El usuario ha denegado esta acción."
        # devolver la denegación como resultado, no lanzar: el modelo puede
        # explicárselo al usuario o proponer otra cosa
    return REGISTRO[nombre](**entrada)
```

---

## Cómo fallan los agentes

Reconocer el patrón es la mitad del arreglo:

| Síntoma | Causa habitual | Corrección |
|---|---|---|
| Da vueltas repitiendo la misma llamada | el resultado de la herramienta no aporta información nueva | mejorar lo que devuelve la herramienta, no el *prompt* |
| Deriva del objetivo en tareas largas | el objetivo quedó sepultado bajo miles de tokens de resultados | reinyectar el objetivo periódicamente; compactar |
| Se rinde antes de terminar | condición de éxito no explícita | escribir el criterio de terminado en el *prompt* |
| El contexto explota en tres pasos | herramientas que devuelven demasiado | acotar, paginar o volcar a fichero y devolver la ruta |
| Usa siempre la herramienta equivocada | descripciones ambiguas o solapadas | reescribir descripciones con la condición de uso |
| Coste diez veces el estimado | esfuerzo alto en tareas que no lo necesitan | bajar `effort`; enrutar por complejidad |

Nótese que **casi ninguna se arregla escribiendo un mejor *prompt***. Se arreglan mejorando lo que devuelven las herramientas, acotando el contexto o cambiando el diseño del bucle. La reescritura del *prompt* es el primer recurso de casi todo el mundo y el que menos veces funciona.

---

## Observabilidad: sin trazas no hay depuración

Un flujo determinista se depura leyendo el código. Un agente, no: hay que ver qué hizo en esa ejecución concreta. El mínimo imprescindible es registrar, por iteración: qué herramientas pidió, con qué parámetros, qué devolvieron, cuántos tokens costó y por qué paró.

```python
logger.info("agente.iteracion", extra={
    "traza_id": traza_id,
    "iteracion": iteracion,
    "herramientas": [b.name for b in respuesta.content if b.type == "tool_use"],
    "stop_reason": respuesta.stop_reason,
    "tokens_entrada": respuesta.usage.input_tokens,
    "tokens_salida": respuesta.usage.output_tokens,
    "tokens_cache_leidos": respuesta.usage.cache_read_input_tokens,
})
```

Con un `traza_id` común a toda la ejecución puedes reconstruir la sesión completa cuando alguien reporte «el agente hizo algo raro». Sin él, la única información será la queja. Los principios generales están en la colección de [observabilidad](../../devops/observabilidad/Observabilidad.md); lo específico aquí es que la unidad de traza no es la petición HTTP, es la **tarea completa** con todas sus iteraciones.

---

## Subagentes y aislamiento de contexto

Un agente puede delegar a otro con su propio contexto. El valor es de aislamiento: si una subtarea genera 50.000 tokens de exploración y solo importan tres líneas de conclusión, hacerla en un subagente mantiene el contexto principal limpio.

Cuándo compensa: exploraciones amplias, trabajos genuinamente independientes en paralelo, revisiones desde perspectivas distintas. Cuándo no: cualquier cosa que se resuelva en dos o tres llamadas a herramientas, porque el subagente paga el coste de arrancar, reconstruir contexto y reportar.

Un detalle que sorprende: los subagentes **no comparten conversación**. Lo que el subagente necesite saber hay que decírselo explícitamente en la tarea que se le encarga. Muchos fallos de sistemas multiagente son en realidad esto —un subagente al que se le pidió algo sin darle el contexto para hacerlo. El fundamento está en [Aislamiento de Contexto](../context-engineering/Aislamiento-de-Contexto.md).

---

## Construirlo tú o usar una plataforma

Tres opciones, y la elección depende de qué parte quieres poseer:

| Opción | Tú escribes | Quién ejecuta |
|---|---|---|
| **Bucle propio** | el `while` completo | tú, en tu infraestructura |
| ***Tool runner* del SDK** | solo las funciones de las herramientas | tú, con el bucle del SDK |
| **Agentes gestionados** | la configuración y tus herramientas | el proveedor, con un entorno aislado por sesión |

Para la mayoría de las aplicaciones, el *tool runner* del SDK es el punto correcto: te ahorra el bucle sin quitarte el control (puedes interceptar cada iteración antes de que se ejecuten las herramientas). El bucle propio se justifica cuando necesitas un flujo de control que el *runner* no contempla. Las plataformas gestionadas compensan cuando además quieres que alguien te aloje el entorno donde se ejecutan las herramientas —un contenedor con sistema de ficheros y comandos— y no quieres operarlo tú.

---

## Buenas prácticas avanzadas

- **Empieza por el escalón más bajo y sube solo con evidencia.** El sesgo por defecto es el contrario: se diseña el agente primero porque es lo interesante. Prueba una llamada, luego una cadena, y solo cuando tengas casos reales que no se pueden resolver así, construye el bucle. La mayoría de los «agentes» en producción son cadenas fijas con dos llamadas, y funcionan mejor por eso: son deterministas, se depuran leyendo el código y cuestan una fracción.
- **Cuando el agente da vueltas, arregla las herramientas antes que el *prompt*.** El bucle infinito casi siempre significa que la herramienta devuelve algo que no permite avanzar: un error genérico sin decir qué parámetro estaba mal, una lista vacía sin decir si es que no hay resultados o si el filtro era inválido, un resultado tan grande que la información útil se pierde. Un mensaje de error que dice exactamente qué corregir resuelve en una iteración lo que veinte reformulaciones del *prompt* no arreglan.
- **Acota el tamaño de todo lo que entra al contexto desde una herramienta.** Es el fallo que mata las tareas largas y el que menos gente anticipa: un `SELECT *` sin límite o el volcado de un fichero grande consumen en una iteración el contexto de toda la tarea. Pagina, recorta con un aviso explícito de que se ha recortado, o escribe el resultado completo en un fichero y devuelve la ruta para que el modelo lea solo lo que necesite.
- **Reinyecta el objetivo en tareas de muchas iteraciones.** La deriva no es que el modelo «se despiste»: es que el objetivo original está a 80.000 tokens de distancia y compite con cientos de resultados de herramientas. Un recordatorio breve del objetivo y del criterio de terminado cada cierto número de iteraciones cuesta muy pocos tokens y evita el fallo más caro de todos, que es un agente trabajando mucho en la dirección equivocada.
- **Clasifica las herramientas en lectura y escritura en el propio código, no en la cabeza.** Un conjunto explícito de herramientas seguras, con confirmación obligatoria para el resto, es la única forma de que la distinción sobreviva a la siguiente persona que añada una herramienta. Y devuelve la denegación como `tool_result` en lugar de lanzar: así el modelo puede explicárselo al usuario o proponer una alternativa, en lugar de que la tarea muera.
- **Instrumenta por tarea, no por petición, y con un identificador de traza compartido.** Una llamada aislada no te dice nada de un agente; lo que necesitas es la secuencia completa de decisiones. Registra herramientas, parámetros, `stop_reason` y tokens por iteración desde el primer día: cuando llegue el primer «hizo algo raro» en producción, o tienes la traza o tienes una anécdota.
- **Acumula el consumo de todas las iteraciones para calcular el coste, nunca lo estimes por la última llamada.** Cada iteración reenvía el historial completo, así que el consumo de entrada crece de forma cuadrática con el número de pasos. Un agente de quince iteraciones puede costar mucho más de quince veces lo que costó la primera, y es la sorpresa clásica de la primera factura.

## Recursos didácticos

- **[Building effective agents (Anthropic)](https://www.anthropic.com/engineering/building-effective-agents)** — la referencia sobre la escala flujo→agente y los patrones de flujo. Su tesis central es que la mayoría de los casos no necesitan un agente, y está mejor argumentada aquí que en ningún otro sitio.
- **[Writing effective tools for agents (Anthropic)](https://www.anthropic.com/engineering/writing-tools-for-agents)** — cómo diseñar la superficie de herramientas, que es donde se decide de verdad si un agente funciona.
- **[ReAct: Synergizing Reasoning and Acting (paper)](https://arxiv.org/abs/2210.03629)** — el artículo que formalizó el patrón razonar-actuar-observar en el que se basa todo bucle agéntico. Corto y muy legible; ver el origen de la idea aclara por qué el bucle tiene la forma que tiene.

---

*En resumen: un agente es un bucle donde el modelo decide los pasos — lo que decide si funciona no es el prompt, son las herramientas que le das, lo que devuelven y los límites que le pones.*
