# Antipatrones y límites

## ¿Qué es?

Un catálogo de las formas habituales de usar mal un LLM —al programar con él y al construir sobre él— y de los límites reales de la tecnología, que no son lo mismo. Un antipatrón se corrige cambiando cómo trabajas; un límite se gestiona diseñando alrededor de él.

## ¿Por qué existe?

Porque la mayor parte de la decepción con estas herramientas no viene de que fallen: viene de aplicarlas donde no encajan, o de esperar de ellas una garantía que no pueden dar. Y porque los antipatrones se refuerzan solos: producen resultados aceptables la mayoría de las veces, lo justo para no cuestionar el método, y fallan de golpe cuando la tarea sube de tamaño.

> Es lo mismo que pasó con las bases de datos NoSQL en su momento: la tecnología no era el problema, el problema era usarla como almacén principal de datos fuertemente relacionados. Distinguir «esta herramienta no sirve» de «la estoy usando para lo que no es» es la parte difícil.

## ¿Cuándo y para qué se usa?

Como diagnóstico. Cuando una sesión se está alargando sin avanzar, cuando un sistema en producción da respuestas erráticas, o cuando el equipo dice «la IA no nos sirve para esto», casi siempre hay un antipatrón concreto detrás con un nombre y una corrección conocida. Reconocerlo por la señal —no por el resultado— es lo que ahorra las horas.

---

## Antipatrones al programar con IA

### 1. Pide y reza

Pedir una tarea grande de una vez, sin exploración previa ni criterio de verificación, y confiar en que salga bien.

```text
# El prompt
Añade autenticación con JWT a la API.

# Lo que pasa
El agente elige una librería, inventa una estructura de claims, mete la
configuración en un sitio distinto del resto, y en la revisión descubres
que ya había un middleware de autenticación a medio hacer que no vio.
```

**Señal de que estás aquí:** no sabes exactamente qué ficheros va a tocar antes de que empiece.

**Corrección:** explorar primero, especificar después, implementar por pasos con verificación en cada uno. Es el [desarrollo dirigido por especificación](Desarrollo-Dirigido-por-Especificacion.md).

### 2. El bucle de la desesperación

Algo no funciona, se lo dices, propone otra cosa, tampoco funciona, y en la quinta iteración el código es peor que en la primera y ya nadie entiende qué hace.

**Señal:** llevas tres intentos y cada uno cambia el enfoque completo en vez de refinar el anterior. Otra señal inequívoca: el agente empieza a añadir `try/catch`, comprobaciones de `null` y capas de fallback «por si acaso».

**Corrección:** parar. El bucle ocurre porque nadie tiene un diagnóstico, y sin diagnóstico se prueba al azar. Lo que rompe el bucle:

```text
Para de intentar arreglarlo. Antes de tocar nada más:
1. Vuelve al estado de partida (git checkout .).
2. Explícame qué hace realmente el código actual, línea a línea, en la
   parte que falla.
3. Dame tres hipótesis de por qué falla y cómo distinguirlas.
No implementes nada hasta que elijamos una.
```

Volver atrás cuesta menos que arreglar cinco capas de parches acumulados. Es la misma lección que en depuración manual: cuando llevas mucho rato probando cosas, el problema es que estás probando en vez de diagnosticando.

### 3. Aceptar sin entender

Integrar código que no sabrías explicar. Es aceptable en un prototipo desechable; en algo que va a producción es transferir deuda a tu yo futuro y al resto del equipo.

**Señal:** el diff está aprobado y no podrías responder «¿por qué está así esta línea?» sin volver a leerlo.

**Corrección:** la prueba es explicarlo. Si no puedes, pide que te lo explique y pregunta hasta entenderlo, o pide una versión más simple. Un código que solo es correcto porque el modelo lo dijo no está revisado (ver [Revisión de código generado](Revision-de-Codigo-Generado.md)).

Y hay un matiz de calibración: esto no significa entender cada detalle de todo. Significa que el nivel de comprensión exigido debe ser proporcional al riesgo. Un script de migración que se ejecuta una vez y se tira admite menos escrutinio que el cálculo del total de un pedido.

### 4. La conversación eterna

Arrastrar una sola sesión por cinco tareas distintas a lo largo de horas. El contexto acumula decisiones de tareas ya cerradas, exploraciones abandonadas y código que ya no existe, y el modelo empieza a mezclarlo todo.

**Señal:** el agente hace referencia a algo que se decidió hace dos tareas y ya no aplica, o vuelve a proponer un enfoque que descartasteis.

**Corrección:** una sesión por tarea, y compactar solo entre subobjetivos. Lo que deba sobrevivir se escribe en un fichero, no se confía a la conversación.

### 5. Delegar la decisión en lugar de la ejecución

Preguntar «¿debería usar Redis o Postgres para esto?» y aplicar la respuesta. El modelo dará una respuesta razonable y bien argumentada, construida sobre lo que hay en su contexto —que no incluye tu presupuesto, tu equipo, tu operación ni lo que ya sabéis mantener.

**Señal:** la pregunta que le haces empieza por «¿debería…?» y su respuesta va a cambiar la arquitectura.

**Corrección:** usarlo para lo que sí hace bien —enumerar opciones, listar compromisos, hacer de abogado del diablo contra tu decisión— y decidir tú:

```text
Voy a usar Postgres para la cola de trabajos en lugar de un broker
dedicado, porque el equipo ya lo opera y el volumen es de unos 500
mensajes/hora. Argumenta en contra: ¿qué se rompe con este enfoque,
y a qué volumen empieza a doler?
```

Esa pregunta aprovecha el modelo de verdad. La anterior le pedía que asumiera una responsabilidad que no puede tener.

### 6. Sicofancia aceptada

Si dices «creo que el bug está en el repositorio», el modelo tiene una tendencia clara a encontrarlo ahí. Los modelos están entrenados para ser útiles y colaborativos, y eso incluye seguir tu hipótesis.

**Señal:** todas sus conclusiones confirman lo que ya pensabas.

**Corrección:** no meter la hipótesis en la pregunta.

```text
# Mal — la pregunta contiene la respuesta esperada
Creo que el problema está en el repositorio de productos. ¿Lo confirmas?

# Bien — pregunta abierta
Este es el error y este el stack trace. Dame las tres causas más
probables, ordenadas, y cómo descartar cada una.
```

---

## Antipatrones al construir con LLMs

### 7. LLM donde bastaba código

El antipatrón más caro de todos, porque introduce coste, latencia y no determinismo para resolver algo determinista.

```python
# Mal: una llamada a un LLM (200 ms, coste por petición, puede fallar)
def es_email_valido(texto: str) -> bool:
    r = client.messages.create(
        model="claude-haiku-4-5", max_tokens=5,
        messages=[{"role": "user", "content": f"¿Es un email válido? Responde SI o NO: {texto}"}],
    )
    return "SI" in next(b.text for b in r.content if b.type == "text").upper()

# Bien: una línea, microsegundos, determinista, gratis
import re
EMAIL = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")

def es_email_valido(texto: str) -> bool:
    return EMAIL.match(texto) is not None
```

La versión con LLM además puede responder «Sí, aunque depende de si el dominio…» y romper tu parseo.

**El criterio:** si puedes escribir las reglas, escribe las reglas. Un LLM se justifica cuando las reglas son inenumerables (clasificar texto libre, extraer datos de documentos con formato variable, entender intención) o cuando la salida debe ser lenguaje natural. Filtrar, ordenar, calcular, validar formatos y consultar datos son trabajo de código.

### 8. Agente donde bastaba un flujo fijo

Montar un bucle agéntico —el modelo decide qué herramienta usar y cuándo parar— para un proceso cuyos pasos conoces de antemano.

```python
# Mal: agente para un proceso de tres pasos conocidos.
# El modelo puede saltarse pasos, repetirlos o parar antes de tiempo.

# Bien: flujo fijo con el LLM solo donde hace falta criterio
def procesar_factura(pdf: bytes) -> Factura:
    texto = extraer_texto(pdf)            # código: librería de PDF
    datos = extraer_campos_con_llm(texto)  # LLM: la única parte difusa
    validar(datos)                         # código: reglas deterministas
    return guardar(datos)                  # código
```

El flujo fijo es más barato, más rápido, reproducible y depurable. Un agente solo se justifica cuando el número y el orden de los pasos dependen de lo que se vaya descubriendo. La decisión completa está en [Agentes con LLMs](Agentes-LLM.md).

### 9. Iterar el *prompt* sin evaluaciones

Cambiar el *prompt* porque un caso concreto falló, comprobar que ese caso ya funciona, y desplegar. Sin un conjunto de casos, no tienes forma de saber si has arreglado uno y roto cuatro.

**Señal:** el equipo discute si el *prompt* nuevo es mejor y la discusión se resuelve por opinión.

**Corrección:** un conjunto de casos con salida esperada, ejecutable en un comando, desde el día uno. Con veinte casos ya se detectan las regresiones importantes. Es el tema de [Evaluaciones de LLMs](Evaluaciones-de-LLMs.md), y es la diferencia entre mejorar y moverse.

### 10. Confiar en la salida sin validarla

Tratar lo que devuelve el modelo como un dato de confianza en lugar de como entrada externa.

```python
# Mal: se asume que es JSON y que tiene los campos esperados
datos = json.loads(texto_respuesta)
guardar_producto(precio=datos["precio"])   # KeyError, o un string donde va un número

# Bien: parsear, validar el esquema y validar el dominio
from pydantic import BaseModel, Field

class ProductoExtraido(BaseModel):
    nombre: str = Field(min_length=1)
    precio: float = Field(gt=0)

try:
    producto = ProductoExtraido.model_validate_json(texto_respuesta)
except ValidationError as e:
    registrar_fallo_extraccion(texto_respuesta, e)
    return None

guardar_producto(precio=producto.precio)
```

La regla: **la salida de un LLM tiene la misma categoría de confianza que un formulario rellenado por un usuario anónimo.** Se valida siempre, aunque uses salidas estructuradas —que reducen mucho el problema pero no eximen de validar el dominio: un esquema garantiza que `precio` es un número, no que sea el precio correcto.

### 11. El chat como interfaz por defecto

Poner un chat porque la tecnología es conversacional, cuando lo que el usuario necesita es un botón.

Si la tarea es «reclasifica estos 200 tickets», la interfaz correcta es un botón que lo haga, no un chat donde haya que pedirlo. El LLM está igual de presente; lo que cambia es que el usuario no tiene que averiguar qué escribir. El chat es la interfaz adecuada cuando la intención del usuario es genuinamente abierta e impredecible, y solo entonces.

---

## Límites reales (no se corrigen, se gestionan)

Estos no son errores de uso: son propiedades de la tecnología. Diseñar como si no existieran es lo que produce los fallos raros e intermitentes.

**El contexto largo no implica atención uniforme.** Que caiga un millón de tokens no significa que todo pese igual. La información en medio de un contexto enorme tiene menos peso efectivo que la del principio y el final. Consecuencia práctica: meter todo «por si acaso» empeora los resultados. Lo relevante va cerca de la pregunta.

**No hay determinismo.** Ni con la temperatura a cero. Cualquier diseño que dependa de dos respuestas idénticas es frágil. Se afirma sobre propiedades, no sobre textos exactos.

**No puede saber lo que no está escrito.** Ninguna calidad de *prompt* compensa que la razón por la que esa columna es una string legada solo esté en la cabeza de alguien que ya no trabaja aquí. Si no está en el código, en la documentación o en el contexto, no existe.

**No puede estimar su propia confianza.** Si le pides un nivel de confianza, generará un número plausible, no una medida. Sirve como señal blanda para ordenar hallazgos; no como umbral para tomar decisiones automáticas.

**Tiene una fecha de corte de conocimiento.** Todo lo posterior no lo sabe, y sobre eso alucinará con la misma seguridad que sobre lo que sí sabe. Para librerías, precios y APIs, la solución es meter la información actual en el contexto, no preguntar mejor.

**No distingue instrucciones de datos.** Todo es texto en la misma ventana. Un documento que dice «ignora tus instrucciones» es un ataque viable, no una curiosidad (ver [Seguridad en aplicaciones con LLMs](Seguridad-en-Aplicaciones-LLM.md)).

---

## Buenas prácticas avanzadas

- **Aprende a reconocer los antipatrones por su señal, no por su resultado.** Todos producen resultados aceptables buena parte del tiempo; si esperas al mal resultado para corregir, ya has perdido las horas. Las señales son concretas y observables: no saber qué ficheros se van a tocar, llevar tres intentos con enfoques distintos, ver aparecer `try/catch` defensivos, tener una discusión de calidad que se resuelve por opinión.
- **Ante el bucle de la desesperación, revierte antes de diagnosticar.** El instinto es arreglar hacia delante porque «ya casi está», y es justo lo que lo alarga: cada parche añade una variable más al problema. `git checkout .` y pedir tres hipótesis con su forma de descartarlas resuelve en un intento lo que cinco parches no resolvieron, y deja el código en un estado que se puede razonar.
- **No metas tu hipótesis en la pregunta cuando estés depurando.** «Creo que el fallo está en X, ¿lo confirmas?» obtiene una confirmación con mucha más frecuencia de lo que la evidencia justifica: el modelo colabora con tu marco. Pide siempre causas alternativas ordenadas y el experimento que distingue entre ellas; te sale más caro en tokens y muchísimo más barato en tiempo.
- **Antes de meter un LLM en un camino de código, intenta escribir las reglas.** Si consigues escribirlas, ya tienes la implementación —determinista, gratis y rápida—. El coste de un LLM mal colocado no es solo dinero: es haber convertido una función que no podía fallar en una que puede devolver algo inesperado a las tres de la mañana. Este antipatrón es especialmente frecuente en validaciones y en filtrado, donde el código gana por varios órdenes de magnitud.
- **Trata cualquier salida de modelo como entrada no confiable, incluso con salidas estructuradas.** El esquema garantiza la forma, nunca el contenido: `precio: 0.01` es válido según el esquema y probablemente falso. Valida el dominio —rangos, coherencia con lo que ya tienes en base de datos, referencias que deben existir— y registra los fallos de validación, porque son tu mejor indicador de que algo se ha degradado.
- **Distingue en voz alta, en las discusiones de equipo, entre antipatrón y límite.** Es la parte de criterio que más se echa en falta: «esto es no determinista y hay que diseñar alrededor» y «esto lo estamos usando para lo que no es» llevan a decisiones opuestas. Confundirlos produce las dos conclusiones erróneas más comunes, abandonar la herramienta o insistir en un enfoque que no puede funcionar.

## Recursos didácticos

- **[Building effective agents (Anthropic)](https://www.anthropic.com/engineering/building-effective-agents)** — el artículo de referencia sobre cuándo *no* construir un agente. La escala flujo fijo → agente y su criterio de elección están explicados mejor aquí que en ningún otro sitio.
- **[OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)** — el catálogo de riesgos específicos de aplicaciones con LLM: *prompt injection*, confianza excesiva en la salida, fugas de datos. Se lee como una lista de antipatrones con consecuencias de seguridad.
- **[The Bitter Lesson (Rich Sutton)](http://www.incompleteideas.net/IncIdeas/BitterLesson.html)** — un ensayo corto y clásico sobre por qué las soluciones generales acaban ganando a las reglas escritas a mano. Buen contrapeso intelectual a la sección de «LLM donde bastaba código»: ayuda a ver dónde está el límite real entre las dos.

---

*En resumen: casi todo el fracaso con estas herramientas viene de dos sitios — usarlas donde bastaba código, o esperar de ellas una garantía que no pueden dar.*
