# Ingeniería con LLMs — Guía completa

Todo lo que necesita saber quien programa para trabajar a alto nivel con modelos de lenguaje, en dos frentes que se refuerzan entre sí: **programar con IA** (usar agentes de codificación de forma que el resultado sea código que puedas defender) y **construir software con LLMs** (integrar modelos en tus aplicaciones de forma que aguanten producción).

Está escrita para perfiles backend junior-medio: se da por sabido HTTP, APIs, bases de datos y Git, y no se da por sabido nada específico de LLMs. Cada ficha empieza por lo básico y llega hasta el detalle que separa a quien ha leído un tutorial de quien ha llevado esto a producción: el fallo silencioso, el parámetro que rompe al migrar, la palanca de coste que casi nadie usa.

Dos ideas atraviesan la colección entera y conviene tenerlas presentes desde el principio:

- **Un LLM solo predice el siguiente token.** Todo lo que parece memoria, certeza o razonamiento lo construyes tú alrededor.
- **La calidad no sale del *prompt*.** Sale de lo que le das (contexto, herramientas), de cómo verificas lo que devuelve (tests, evaluaciones) y de los límites que le pones (permisos, presupuestos).

---

## Orden de lectura recomendado

### 1. Fundamentos

El modelo mental sin el que todo lo demás son recetas. Dos fichas cortas que explican por qué los sistemas con LLM fallan como fallan.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Cómo piensa un LLM](Como-Piensa-un-LLM.md) | Tokens, probabilidades, contexto y alucinaciones. Casi todos los problemas que verás después son consecuencia directa de lo que hay aquí. |
| 2 | [Modelos y parámetros](Modelos-y-Parametros.md) | Qué modelo usar en cada ruta y qué hacen de verdad `max_tokens`, el esfuerzo y el *thinking*. La decisión de configuración con más impacto y menos coste. |

### 2. Programar con IA

Cómo usar agentes de codificación a nivel experto. El hilo conductor: lo que decide el resultado no es cómo pides las cosas, es qué información das y cómo se verifica lo que sale.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [Prompting para programadores](Prompting-para-Programadores.md) | Qué hace que una petición produzca código integrable, qué técnicas funcionan y cuáles son folclore. Base de las tres fichas siguientes. |
| 4 | [Agentes de codificación](Agentes-de-Codificacion.md) | El bucle, las herramientas, los permisos, el fichero de instrucciones del repositorio y la técnica que más cambia los resultados: darle un bucle cerrado. |
| 5 | [Desarrollo dirigido por especificación](Desarrollo-Dirigido-por-Especificacion.md) | Cómo abordar tareas grandes: explorar, especificar, planificar, implementar por pasos. Porque cuando implementar es barato, lo caro es decidir. |
| 6 | [Revisión de código generado](Revision-de-Codigo-Generado.md) | Cómo falla el código generado —que no es como falla el humano— y cómo revisarlo sin que el aspecto impecable te desarme. |

### 3. Construir software con LLMs

El lado de ingeniería: poner un modelo dentro de tu aplicación. De la llamada más simple al bucle agéntico.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 7 | [Llamadas a la API](Llamadas-a-la-API.md) | La petición, los roles, los bloques de respuesta, el *streaming*, los errores y los lotes. La base mecánica de todo lo demás. |
| 8 | [Salidas estructuradas y tool use](Salidas-Estructuradas-y-Tool-Use.md) | Cómo convertir texto en datos con esquema garantizado y cómo darle al modelo capacidad de actuar. Las dos piezas que lo hacen integrable. |
| 9 | [Agentes con LLMs](Agentes-LLM.md) | El bucle donde el modelo decide los pasos, cuándo *no* construirlo (casi siempre) y los límites sin los que es una factura abierta. |
| 10 | [MCP](MCP.md) | El protocolo estándar para conectar modelos con tus sistemas: cuándo compensa, cómo escribir un servidor y por qué es la superficie de seguridad más delicada. |

### 4. Llevarlo a producción

Lo que distingue un prototipo que funciona en una demo de un sistema que aguanta tráfico real.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 11 | [Evaluaciones de LLMs](Evaluaciones-de-LLMs.md) | Cómo medir si un cambio mejora, con un componente no determinista. Sin esto, iterar un *prompt* es apostar. |
| 12 | [Coste, latencia y fiabilidad](Coste-Latencia-y-Fiabilidad.md) | Dónde está el dinero, dónde el tiempo, y los tres modos de fallo que llegan con código 200 y nadie detecta. |
| 13 | [Seguridad en aplicaciones con LLMs](Seguridad-en-Aplicaciones-LLM.md) | *Prompt injection* directa e indirecta, fuga de datos, exceso de agencia. La seguridad no está en el *prompt*: está en lo que puede hacer el sistema cuando el *prompt* falle. |

### 5. Cierre

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 14 | [Antipatrones y límites](Antipatrones-y-Limites.md) | El catálogo de formas de usar mal esto, con su señal de alarma y su corrección, y los límites reales que no se corrigen sino que se gestionan. Se lee mejor al final, cuando ya reconoces cada escenario. |

---

## Índice completo

<details>
<summary>Ver todos los archivos</summary>

**Fundamentos**
- [Cómo piensa un LLM](Como-Piensa-un-LLM.md)
- [Modelos y parámetros](Modelos-y-Parametros.md)

**Programar con IA**
- [Prompting para programadores](Prompting-para-Programadores.md)
- [Agentes de codificación](Agentes-de-Codificacion.md)
- [Desarrollo dirigido por especificación](Desarrollo-Dirigido-por-Especificacion.md)
- [Revisión de código generado](Revision-de-Codigo-Generado.md)

**Construir software con LLMs**
- [Llamadas a la API](Llamadas-a-la-API.md)
- [Salidas estructuradas y tool use](Salidas-Estructuradas-y-Tool-Use.md)
- [Agentes con LLMs](Agentes-LLM.md)
- [MCP](MCP.md)

**Producción**
- [Evaluaciones de LLMs](Evaluaciones-de-LLMs.md)
- [Coste, latencia y fiabilidad](Coste-Latencia-y-Fiabilidad.md)
- [Seguridad en aplicaciones con LLMs](Seguridad-en-Aplicaciones-LLM.md)

**Cierre**
- [Antipatrones y límites](Antipatrones-y-Limites.md)

</details>

---

## Si tienes poco tiempo

Tres recorridos cortos según lo que necesites hoy:

- **Quiero programar mejor con un agente** → [Cómo piensa un LLM](Como-Piensa-un-LLM.md) → [Agentes de codificación](Agentes-de-Codificacion.md) → [Revisión de código generado](Revision-de-Codigo-Generado.md).
- **Tengo que integrar un LLM en una aplicación** → [Llamadas a la API](Llamadas-a-la-API.md) → [Salidas estructuradas y tool use](Salidas-Estructuradas-y-Tool-Use.md) → [Coste, latencia y fiabilidad](Coste-Latencia-y-Fiabilidad.md).
- **Ya tengo algo en producción y me da miedo** → [Evaluaciones de LLMs](Evaluaciones-de-LLMs.md) → [Seguridad en aplicaciones con LLMs](Seguridad-en-Aplicaciones-LLM.md) → [Antipatrones y límites](Antipatrones-y-Limites.md).

---

> Esta colección explica **qué** meter en el contexto y qué hacer con lo que sale. Para el **cómo** gestionar ese contexto cuando crece —ventana, RAG, compactación, memoria, caché— la colección hermana es [Context Engineering](../context-engineering/README.md).
