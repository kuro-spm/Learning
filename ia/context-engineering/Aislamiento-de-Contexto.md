# Aislamiento de Contexto

## ¿Qué es?

El aislamiento de contexto es la práctica de repartir una tarea compleja entre varios agentes o llamadas a un LLM, cada uno con su propia ventana de contexto independiente, en vez de acumular todo el trabajo (búsquedas, resultados intermedios, herramientas ejecutadas) en un único contexto compartido y cada vez más grande.

## ¿Por qué existe?

Un solo agente que investiga, ejecuta herramientas y razona sobre una tarea larga va acumulando en su contexto cada resultado intermedio, cada búsqueda, cada paso dado, aunque la mayoría de ese detalle ya no sea relevante para el paso siguiente. Además de agotar antes la ventana de contexto (ver [Compactación de Contexto](Compactacion-de-Contexto.md)), ese ruido acumulado puede distraer al modelo o hacer que pierda el foco de la tarea principal. Repartir subtareas independientes a agentes separados, cada uno con su propio contexto limpio, evita que el detalle de una subtarea contamine el razonamiento de las demás: cada agente solo ve lo que necesita para su parte, y devuelve al proceso principal solo el resultado, no todo el camino que recorrió para llegar a él.

> Piensa en delegar una investigación a varias personas de un equipo en vez de hacerla todo tú: cada una investiga su parte con sus propias notas y solo te entrega la conclusión, no las cien páginas de búsquedas que hizo por el camino; tu propia cabeza (tu contexto) no se llena con el detalle de un trabajo que no era el tuyo.

## ¿Cuándo y para qué se usa?

Cuando una tarea se puede dividir en subtareas razonablemente independientes: investigar varias fuentes en paralelo, revisar distintas partes de un código de forma independiente, buscar información desde ángulos distintos antes de combinar los hallazgos. También cuando una subtarea genera mucho ruido intermedio (leer documentos extensos, probar varias veces algo hasta acertar) que no aporta nada al resto del proceso una vez resuelta.

## Lo mínimo que necesitas saber

**1. Un agente coordinador delega en agentes especializados, cada uno con su propio contexto**

```csharp
var resultados = await Task.WhenAll(
    agenteInvestigador.Ejecutar("Busca información sobre la competencia A"),
    agenteInvestigador.Ejecutar("Busca información sobre la competencia B"),
    agenteInvestigador.Ejecutar("Busca información sobre la competencia C")
);
// cada llamada tiene su propio contexto, ninguna ve el ruido de búsqueda de las otras
```

**2. Solo el resultado final cruza de vuelta al contexto principal**

```csharp
var resumen = await agenteCoordinador.Sintetizar(resultados);
// el coordinador solo recibe las 3 conclusiones, no las docenas de búsquedas intermedias de cada agente
```

**3. Cada agente puede tener instrucciones y herramientas distintas, ajustadas a su subtarea**

Un agente de investigación puede tener acceso a búsqueda web; un agente de redacción, ninguno; eso evita que un agente use, por error o por exceso de opciones disponibles, una herramienta que no le corresponde a su parte del trabajo.

## Lo que NO hace

- **No elimina la necesidad de sintetizar al final** — dividir el trabajo no resuelve por sí solo cómo combinar resultados parciales, potencialmente contradictorios, en una respuesta única y coherente; ese paso de síntesis sigue haciendo falta y es responsabilidad de quien coordina.
- **No es gratis** — cada agente adicional es una o varias llamadas más al modelo, con su coste y su latencia propios; repartir en agentes solo compensa cuando el ruido evitado o el paralelismo ganado justifica ese coste extra.
- **No sirve para cualquier tarea** — una tarea muy secuencial, donde cada paso depende estrechamente del contexto completo del paso anterior, no se beneficia de aislarse en agentes independientes; el aislamiento ayuda con subtareas genuinamente separables.

## Buenas prácticas avanzadas

- **Decide qué cruza la frontera entre agentes con la misma disciplina que una API entre servicios** — igual que un servicio no debería exponer su implementación interna a quien lo consume, un agente delegado no debería devolver todo su razonamiento interno al coordinador, solo el resultado que este necesita; menos superficie compartida, menos ruido y menos acoplamiento.
- **No repartas en agentes independientes una tarea con dependencias fuertes entre pasos** — si el paso 2 necesita saber exactamente cómo se llegó al resultado del paso 1 (no solo el resultado), aislarlos en contextos separados obliga a reconstruir esa información, lo que puede acabar costando más que mantenerlos en el mismo contexto.
- **Vigila el coste total, no solo el de cada agente individual** — diez agentes en paralelo, cada uno barato por separado, pueden sumar un coste y una carga total mayor que una única llamada bien diseñada; el aislamiento aporta valor cuando el problema realmente lo pide, no como configuración por defecto.
- **Da a cada agente el mínimo de herramientas e instrucciones que necesita, no todas las disponibles** — un agente con acceso a herramientas que no le corresponden a su subtarea concreta es más difícil de predecir y de depurar cuando algo sale mal; limitar su superficie de acción es parte del propio aislamiento.

---

*En resumen: el aislamiento de contexto reparte una tarea entre agentes con ventanas de contexto propias, para que el ruido y el detalle de una subtarea no contaminen el razonamiento de las demás — cada agente aporta solo su resultado, no todo el camino que recorrió para llegar a él.*
