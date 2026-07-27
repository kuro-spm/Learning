# Agentes de codificación

## ¿Qué es?

Un agente de codificación es un LLM al que se le han dado herramientas para actuar sobre tu proyecto —leer ficheros, escribirlos, ejecutar comandos— y un bucle para seguir actuando hasta terminar la tarea. La diferencia con un asistente de chat es que no te *dice* qué hacer: lo hace, comprueba el resultado y corrige.

## ¿Por qué existe?

El autocompletado en el editor resolvió el problema pequeño: escribir la línea siguiente. Pero la mayoría del trabajo real no es escribir código, es **entender el que hay** —buscar dónde está definida una cosa, ver cómo se usa en otros sitios, ejecutar los tests, leer el error, volver atrás. Un modelo que solo puede leer el fichero abierto no puede hacer nada de eso.

Darle herramientas cierra el círculo: ahora puede buscar en todo el repositorio, ejecutar la *suite* de tests y leer el fallo. Y eso cambia la clase de tarea que puede resolver, de «rellena esta línea» a «migra estos 30 ficheros al nuevo cliente HTTP y deja los tests en verde».

> Si has usado un depurador paso a paso, la intuición es parecida: lo valioso no es leer el código, es poder *ejecutarlo y observar*. Un agente sin herramientas es como razonar sobre un bug leyendo el fuente; con herramientas, es poner un *breakpoint*.

## ¿Cuándo y para qué se usa?

Va especialmente bien donde el trabajo es mecánico pero disperso, o donde existe una forma automática de saber si está bien:

- **Cambios repetitivos y amplios** — renombrar un concepto en todo el proyecto, migrar de una librería a otra, actualizar 40 llamadas a una API que cambió de firma.
- **Trabajo con verificación automática** — arreglar un test que falla, hacer que compile, cumplir un *linter*. El agente tiene un juez objetivo y puede iterar solo.
- **Exploración de código desconocido** — «¿dónde se decide el precio final de un pedido?» en un repositorio que acabas de heredar.
- **Escribir tests de código existente** — tarea tediosa, con criterio de éxito claro y bajo riesgo.

Y va mal donde no hay señal de corrección o donde el criterio es tácito: decisiones de arquitectura con implicaciones de negocio, ajustes de rendimiento que solo se validan bajo carga real, o cualquier cosa donde «está bien» dependa de algo que no está escrito en ningún sitio.

---

## Anatomía: el bucle

Todos los agentes de codificación funcionan igual por dentro, independientemente de la marca:

```text
1. Se le da una tarea y el contexto del proyecto.
2. El modelo decide: ¿respondo, o uso una herramienta?
3. Si usa una herramienta (leer fichero, ejecutar comando...), el sistema
   la ejecuta y le devuelve el resultado.
4. Ese resultado entra en el contexto y se vuelve al paso 2.
5. Cuando el modelo no pide más herramientas, ha terminado.
```

El conjunto de herramientas típico es corto y siempre el mismo:

| Herramienta | Para qué |
|---|---|
| Leer fichero | ver el código real, no imaginárselo |
| Buscar (`glob` / `grep`) | localizar dónde está algo en un repositorio grande |
| Escribir / editar | aplicar el cambio |
| Ejecutar comandos (`bash`) | compilar, tests, `git`, cualquier herramienta del proyecto |
| Buscar en web / leer URL | documentación actualizada, mensajes de error poco conocidos |

Esto importa por una razón práctica: **entender que hay un bucle explica todos los comportamientos que de fuera parecen raros.** Cuando el agente da muchas vueltas, es que cada iteración no le está dando información nueva. Cuando «se olvida» de algo, es que el resultado de una herramienta llenó el contexto y desplazó lo anterior. Cuando insiste en un enfoque equivocado, es que nada en el bucle le está diciendo que va mal.

El mecanismo genérico del bucle, aplicado a tus propias aplicaciones, es el tema de [Agentes con LLMs](Agentes-LLM.md); aquí nos quedamos en cómo aprovecharlo como herramienta de trabajo.

---

## El panorama de herramientas

Cuatro formas distintas de integrar esto en el día a día, con compromisos diferentes:

| Forma | Ejemplos | Fuerte en | Flojo en |
|---|---|---|---|
| **Autocompletado en el editor** | GitHub Copilot, Cursor Tab | velocidad línea a línea, cero fricción | no ve el proyecto entero, no verifica |
| **Chat integrado en el editor** | Copilot Chat, Cursor, Continue | contexto de lo que tienes abierto, iteración cómoda | tú aplicas los cambios y tú ejecutas los tests |
| **Agente de terminal** | Claude Code, Aider, Codex CLI | ve y modifica todo el repositorio, ejecuta comandos, cierra el bucle solo | requiere confiar en él para ejecutar cosas |
| **Agente remoto / asíncrono** | agentes sobre *pull requests*, agentes en la nube | trabaja mientras haces otra cosa, tareas largas | ciclo de corrección más lento, sin ver tu entorno local |

No son excluyentes y quien saca más partido usa varias: autocompletado mientras escribe a mano, agente de terminal para las tareas de varios ficheros, agente asíncrono para lo tedioso que puede esperar. La elección real no es «qué herramienta es mejor» sino **cuánta autonomía quieres darle a cada tarea**.

---

## El fichero de instrucciones del repositorio

Es la pieza con mejor relación esfuerzo/resultado de todo el trabajo con agentes, y la que más gente se salta.

Los agentes leen al arrancar un fichero de instrucciones del repositorio —`CLAUDE.md`, `AGENTS.md`, `.cursorrules` o el equivalente de tu herramienta— y lo tratan como contexto permanente. Ahí van las cosas que un desarrollador nuevo necesitaría saber y que no se deducen del código:

```markdown
# Proyecto: API de tienda online

## Comandos
- Compilar: `dotnet build`
- Tests: `dotnet test` (unitarios) / `dotnet test --filter Category=Integration`
- Levantar en local: `docker compose up -d && dotnet run --project src/Api`

## Convenciones
- Importes monetarios: `decimal`, nunca `double`.
- Acceso a datos con Dapper y SQL explícito. No añadas EF Core.
- Las excepciones no se capturan en controladores: suben al middleware.
- Nombres de dominio en español (`Pedido`), técnicos en inglés (`Repository`).

## Cosas que no son obvias
- `Productos.Estado` es una string legada, no un enum. No la refactorices
  sin migración: hay integraciones externas que dependen de esos valores.
- Los tests de integración necesitan Docker levantado.
```

Fíjate en qué tipo de información contiene: **comandos que no adivinaría** y **decisiones cuya razón no está en el código**. Eso es lo que hay que escribir. Lo que sí se deduce leyendo el proyecto —qué framework se usa, cómo se llaman las clases— es relleno que gasta contexto sin aportar.

La forma correcta de construirlo no es sentarse a escribirlo entero: es **añadir una línea cada vez que corriges lo mismo dos veces**. Si has tenido que decir «los tests de integración necesitan Docker» en dos sesiones distintas, esa frase va al fichero. Al mes tienes un documento que refleja el conocimiento tácito real del proyecto, sin haber dedicado una tarde a ello.

---

## Permisos: el compromiso central

Un agente que puede ejecutar comandos puede hacer daño. No por malicia: por un `rm` con una ruta mal construida, un `git reset --hard` sobre trabajo sin *commit*, un `terraform apply` en el entorno equivocado.

Las herramientas ofrecen niveles de autonomía, y elegir el nivel *por tarea* es lo que distingue el uso maduro:

| Nivel | Comportamiento | Cuándo |
|---|---|---|
| **Pregunta siempre** | cada comando espera aprobación | código desconocido, entornos con acceso a producción |
| **Lista de permitidos** | los comandos seguros pasan solos, el resto pregunta | el modo de trabajo habitual |
| **Autonomía amplia** | ejecuta sin preguntar | tareas mecánicas, en entorno aislado y con Git limpio |

Una lista de permitidos razonable pone del lado libre lo que solo lee, y deja preguntando todo lo que escribe fuera del repositorio o toca la red:

```json
{
  "permissions": {
    "allow": [
      "Bash(dotnet build)",
      "Bash(dotnet test:*)",
      "Bash(git status)",
      "Bash(git diff:*)",
      "Bash(git log:*)"
    ]
  }
}
```

Con esa configuración, el agente compila y ejecuta tests las veces que necesite sin interrumpirte —que es justo lo que quieres para que itere solo— pero sigue pidiendo permiso para hacer *commit*, instalar paquetes o llamar a una API externa.

Dos precauciones que valen más que cualquier configuración:

- **Trabaja siempre sobre Git limpio y en una rama.** No es una precaución teórica: es la diferencia entre «deshago el desastre con `git checkout .`» y «he perdido dos horas de trabajo».
- **Nunca des autonomía amplia en un entorno con credenciales de producción.** El modelo no distingue tu entorno de pruebas del real; lo único que lo distingue es lo que hay en las variables de entorno de esa terminal.

---

## Darle un bucle cerrado: la técnica que más cambia los resultados

Un agente rinde de forma radicalmente distinta según pueda o no comprobar su propio trabajo. Compara estas dos formas de pedir lo mismo:

```text
# Bucle abierto: el agente escribe y se va. Tú descubres los fallos después.
Añade validación de stock al añadir un producto al carrito.

# Bucle cerrado: el agente tiene un juez y no para hasta que dice que sí.
Añade validación de stock al añadir un producto al carrito.
Cuando termines, ejecuta `dotnet test --filter Carrito` y arregla lo que
falle. No des la tarea por buena hasta que pase en verde.
```

La segunda versión suele tardar más en la primera respuesta y llegar mucho más lejos, porque el agente detecta y corrige sus propios errores en vez de dejártelos. Es el mismo principio que hace útil el TDD, aplicado a un colaborador que puede iterar muy rápido.

Llevado al extremo, el patrón más potente es **pedir el test primero**:

```text
1. Escribe un test que falle y que capture el comportamiento que quiero:
   añadir al carrito más unidades de las que hay en stock debe devolver
   un error de validación, no lanzar una excepción.
2. Enséñamelo antes de implementar.
3. Cuando lo apruebe, implementa hasta que pase.
```

El test es una especificación ejecutable: mucho más difícil de malinterpretar que un párrafo en prosa. Y como lo revisas *antes*, lo que estás revisando es el criterio, que es lo barato de corregir. Esta idea llevada a tareas grandes es el [desarrollo dirigido por especificación](Desarrollo-Dirigido-por-Especificacion.md).

---

## Gestionar el contexto en sesiones largas

El contexto se llena, y cuando se llena la calidad baja: instrucciones del principio que dejan de tenerse en cuenta, decisiones que se repiten, el agente redescubriendo algo que ya había averiguado.

Tres hábitos que lo evitan:

- **Una sesión por tarea.** El error más común es arrastrar una conversación de tres horas por cinco tareas distintas. Los 200.000 tokens de la tarea anterior no ayudan a la siguiente: la estorban. Empieza limpio cuando cambies de tema.
- **Compactar en un punto natural.** Casi todas las herramientas tienen una orden para resumir la conversación y seguir (`/compact` y equivalentes). Úsala **después** de terminar un subobjetivo, no en medio de una depuración: lo que se resume, se pierde en detalle.
- **Guardar en un fichero lo que debe sobrevivir.** Si el agente ha averiguado algo que hará falta más adelante —«el `Estado` legado admite estos seis valores»— pídele que lo escriba en el fichero de instrucciones o en un `.md` de notas. Lo que está en un fichero se puede volver a leer; lo que solo está en la conversación se pierde al compactar.

El fundamento de todo esto está en [Compactación de Contexto](../context-engineering/Compactacion-de-Contexto.md) y [Memoria de Agentes](../context-engineering/Memoria-de-Agentes.md).

---

## Subagentes y paralelismo

Los agentes pueden lanzar **subagentes**: instancias con su propio contexto que hacen una subtarea y devuelven solo la conclusión. Su valor real es de aislamiento de contexto: si buscar en qué sitios se usa una función requiere leer 30 ficheros, hacerlo en un subagente deja esos 30 ficheros fuera de tu contexto principal y solo trae la lista final.

Cuándo compensa y cuándo no:

- **Sí:** exploraciones amplias («busca en todo el repositorio los sitios donde se calcula IVA»), trabajos independientes en paralelo, revisiones desde perspectivas distintas sobre el mismo diff.
- **No:** cualquier cosa que resolverías con dos o tres lecturas de fichero. Cada subagente paga el coste de arrancar, reconstruir contexto y reportar; para una tarea pequeña ese coste es mayor que la tarea.

El paralelismo de verdad se consigue con **varios *worktrees* de Git**: cada agente en su propia copia del repositorio, sin pisarse. Es la forma limpia de tener tres tareas independientes avanzando a la vez:

```bash
# Un worktree por tarea, cada uno en su rama
git worktree add ../proyecto-cupones -b feature/cupones
git worktree add ../proyecto-migracion -b chore/migracion-http

# Y un agente trabajando en cada carpeta, sin conflictos entre ellos
```

Sin *worktrees*, dos agentes sobre la misma carpeta se sobrescriben los cambios; con ellos, cada uno tiene su árbol y su rama y luego se integran como cualquier otra rama.

---

## Automatización: ganchos y comandos propios

Cuando un flujo se repite, se puede fijar. Dos mecanismos habituales:

**Ganchos (*hooks*)** — comandos que la herramienta ejecuta automáticamente al ocurrir algo, por ejemplo formatear siempre después de editar un fichero:

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{ "type": "command", "command": "dotnet format --no-restore" }]
    }]
  }
}
```

Esto es determinista: lo ejecuta el sistema, no el modelo. Y eso es exactamente por lo que conviene usarlo para las reglas que no se pueden negociar. Pedirle al modelo «acuérdate de formatear» funciona la mayoría de las veces; un gancho funciona siempre.

**Comandos propios** — un fichero Markdown con un flujo que repites, invocable por nombre. Por ejemplo un `/revisar` que siempre haga la misma revisión con los mismos criterios. Convierte un *prompt* largo que reescribes cada vez en algo reutilizable y mejorable por todo el equipo.

Y para conectar el agente a sistemas externos —tu gestor de incidencias, tu base de datos, tu documentación interna— el estándar es **MCP**, que tiene su propia ficha: [MCP](MCP.md).

---

## Buenas prácticas avanzadas

- **Da siempre un criterio de verificación en el propio *prompt*, no lo dejes implícito.** «Ejecuta los tests y arregla lo que falle» convierte al agente de generador en iterador, y es la diferencia entre un cambio que hay que repasar entero y uno que llega verificado. Si la tarea no tiene forma automática de comprobarse, el primer paso es *crearla* —un test, un script, un comando de validación— y solo después pedir el cambio.
- **Escribe el fichero de instrucciones de forma incremental, a partir de correcciones repetidas.** Sentarse a redactarlo entero produce documentos genéricos que no evitan ningún error. La regla es: la segunda vez que corriges lo mismo, no lo corrijas —escríbelo. Y limita el fichero a comandos no obvios y decisiones con razón oculta; todo lo que se deduce leyendo el código es contexto malgastado.
- **Una tarea, una sesión; y compacta solo entre subobjetivos.** Arrastrar contexto entre tareas distintas es la causa número uno de degradación de calidad en sesiones largas: el modelo empieza a mezclar decisiones de la tarea anterior. Y compactar en medio de una depuración destruye justo los detalles que estabas usando.
- **Ajusta el nivel de permisos por tarea, no una vez y para siempre.** El fallo típico no es dar demasiada autonomía: es dar el *mismo* nivel a «renombra esta variable en 20 ficheros» y a «arregla el despliegue». Autonomía amplia en entorno aislado con Git limpio es productiva; el mismo nivel en una terminal con credenciales de producción es un incidente esperando su turno.
- **Usa subagentes por aislamiento de contexto, no por sensación de paralelismo.** El criterio: ¿la subtarea va a generar mucha lectura que no necesito conservar? Si sí, subagente. Si lo resolverías con dos lecturas de fichero, hacerlo directamente es más rápido y más barato. Delegar una tarea pequeña cuesta más que hacerla.
- **Fija con ganchos lo que no se puede olvidar, y deja al modelo lo que requiere criterio.** Formateo, *linter*, comprobaciones previas a *commit*: ganchos, porque son deterministas y no negociables. Decidir cómo estructurar un cambio: el modelo. Confundir los dos —pedirle al modelo lo que debería ser un gancho, o intentar automatizar con un script lo que necesita juicio— es la forma más habitual de tener un flujo frágil.
- **Revisa el diff completo antes de aceptar, siempre, incluso cuando los tests pasan.** Los tests en verde prueban que no rompiste lo que estaba cubierto; no prueban que el cambio sea el correcto ni que no haya añadido tres abstracciones que nadie pidió. Ese repaso es el tema de [Revisión de código generado](Revision-de-Codigo-Generado.md), y es la parte que no se puede delegar.

## Recursos didácticos

- **[Documentación de Claude Code](https://code.claude.com/docs)** — la referencia de un agente de terminal completo: ficheros de instrucciones, permisos, ganchos, subagentes, MCP. Útil incluso si usas otra herramienta, porque los conceptos son los mismos.
- **[Best practices for agentic coding (Anthropic)](https://www.anthropic.com/engineering/claude-code-best-practices)** — prácticas destiladas de uso real, con énfasis en el bucle de verificación y la gestión de contexto.
- **[Aider — LLM code editing benchmarks](https://aider.chat/docs/leaderboards/)** — comparativas reproducibles de modelos en tareas reales de edición de código. Buen antídoto contra elegir modelo por impresión.
- **[git worktree](https://git-scm.com/docs/git-worktree)** — la documentación oficial del mecanismo que hace posible tener varios agentes trabajando en paralelo sin pisarse.

---

*En resumen: un agente de codificación vale lo que valga el bucle que le des — sin forma de comprobar su trabajo es un generador de texto rápido; con tests que ejecutar, es un colaborador que corrige sus propios errores.*
