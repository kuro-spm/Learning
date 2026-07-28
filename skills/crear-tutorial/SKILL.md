---
name: crear-tutorial
description: >-
  Crea colecciones de "tutoriales" o guías de tecnología siguiendo una metodología fija: una carpeta
  nueva en la raíz del repositorio de tutoriales (C:\Users\MaccionUser\GitRepos\Learning) con un README
  que hace de índice (con enlaces) y guías completas en español, autónomas y dirigidas a perfiles
  backend junior-medio, legibles por cualquiera sin contexto de un proyecto concreto.
  Usa esta skill siempre que la usuaria pida crear, añadir, escribir o documentar un tutorial, una guía,
  una ficha o una colección de guías sobre una librería, framework, lenguaje, herramienta o concepto,
  aunque no use literalmente la palabra "tutorial".
---

# Crear tutorial (guía de tecnología)

Esta skill genera **colecciones de guías de tecnología** replicando una metodología concreta: una carpeta con un README que actúa de índice (enlazando a cada documento) y una ficha por tecnología o concepto.

Las fichas son **guías completas**, no resúmenes: empiezan por una introducción accesible y a partir de ahí van tan a fondo como pida el tema. Lo que se mantiene idéntico entre fichas es el **esqueleto** (apertura, cierre y tono); el desarrollo intermedio lo dicta cada tecnología.

Los tutoriales son **autónomos y genéricos**: cualquier persona debe poder leerlos sin conocer ningún proyecto concreto. Explican la tecnología en sí (qué es, por qué existe, cómo se usa), no cómo se aplica en un código privado.

> **Autorización permanente de commit y push.** La usuaria ha autorizado de antemano, como parte de esta skill, que al terminar y verificar un trabajo se haga `git add` + `git commit` + `git push` sin pedir confirmación adicional (ver «Flujo de trabajo», paso 7). Esta autorización cubre específicamente el contenido creado por esta skill; no autoriza a subir cambios ajenos que estuvieran ya presentes en el working tree antes de empezar la tarea (esos se dejan tal cual, sin stagear).

## Dónde se crea el contenido

> **Repositorio destino fijo.** Esta skill es **global**: se puede invocar desde cualquier carpeta. Pero **siempre** escribe en el repositorio de tutoriales, sea cual sea el directorio de trabajo actual:
>
> ```
> C:\Users\MaccionUser\GitRepos\Learning
> ```
>
> En toda esta guía, `<raíz>` se refiere a esa ruta. **No** crees contenido en el directorio de trabajo actual si es otro distinto. Todas las rutas relativas de ejemplo (`bases-de-datos/…`, `seguridad/…`, etc.) cuelgan de esa raíz, y todas las operaciones de git del paso 7 (`git add`, `commit`, `push`) se ejecutan **dentro de ese repositorio** (por ejemplo con `git -C "C:\Users\MaccionUser\GitRepos\Learning" …` si el directorio de trabajo actual es otro).

El repositorio agrupa las colecciones de tutoriales en **carpetas de categoría** en la raíz. Cada tema vive dentro de la categoría a la que pertenece:

```
<raíz>/                       ← C:\Users\MaccionUser\GitRepos\Learning
└── <categoria>/             ← carpeta de categoría existente (kebab-case: "devops", "seguridad"...)
    └── <tema>/               ← carpeta nueva (kebab-case: "git-basico", "docker", "patrones-async")
        ├── README.md         ← índice: orden de lectura + enlaces a cada ficha
        └── <Concepto>.md     ← una ficha por tecnología o concepto
```

Categorías existentes: `arquitectura-de-software/`, `desarrollo-web/`, `lenguajes/`, `bases-de-datos/`, `testing/`, `devops/`, `seguridad/`, `redes/`, `herramientas/`, `odoo/`, `ia/`.

- Antes de crear la carpeta del tema, **decide en qué categoría encaja** mirando las categorías existentes (y las colecciones que ya contienen, para calibrar). Si el tema encaja en varias o en ninguna, **pregúntalo** a la usuaria en vez de asumir.
- Si el tema justifica una categoría nueva, créala igual que las demás (kebab-case) y añádele su propio `README.md` índice.
- El nombre de la carpeta del tema describe el tema en kebab-case. Si no está claro, **pregúntalo**.
- Si la colección de un tema es grande y conviene agrupar sus propias fichas, puedes crear subcarpetas dentro de ella, cada una con su propio `README.md` índice.
- Tras crear o mover una carpeta de tema, actualiza el `README.md` de su categoría (añade el enlace) y el `README.md` raíz si la categoría es nueva.
- Si te piden documentar el stack de un proyecto concreto (no un tutorial genérico), **pregunta primero** dónde debe vivir: por defecto no tiene cabida en este repositorio de tutoriales autónomos (ver principio 4).

## Antes de escribir nada

Lee 2-3 documentos ya existentes en el repositorio para calibrar **tono, registro y estilo de los ejemplos** (no su contenido).

- `devops/despliegue-en-vps/UFW.md` — ejemplo de tono y de apertura/cierre.
- `bases-de-datos/acceso-a-datos-dotnet/Dapper.md` — ejemplo de guía completa con tablas de decisión y errores frecuentes.
- `devops/despliegue-en-vps/README.md` — ejemplo de **README-índice**.

> **Ojo con las fichas antiguas.** El repositorio se escribió al principio como colección de introducciones breves y aún queda contenido en ese formato: fichas de 60-120 líneas, sin «Buenas prácticas avanzadas» y con secciones tipo «Lo mínimo que necesitas saber» o «Lo que NO hace». **No las uses como referencia de profundidad ni de índice de secciones**: para eso manda esta skill. Las tres citadas arriba sí están en el formato actual.
>
> Si la usuaria pide ampliar o reescribir una ficha antigua concreta, se lleva al formato actual respetando sus decisiones de contenido y sus analogías.

## Principios fundamentales

Estos pilares no se negocian:

1. **Idioma: español.** La prosa va en español. Nombres de tecnologías, comandos, código y términos técnicos consolidados (*proxy*, *bundler*, *change tracking*) se mantienen en su idioma original.
2. **Audiencia: perfil backend junior-medio.** El lector programa a diario y puede conocer algunas partes del stack (HTTP, APIs REST, bases de datos, componentes de UI, npm/NuGet, Git...), pero puede ser nuevo en la tecnología concreta que documentas. No le expliques qué es una API o un bucle; sí define los términos propios de la tecnología antes de usarlos y no des por sabidos sus detalles internos.
3. **Analogías y lenguaje claro.** Explica cada concepto nuevo anclándolo a algo que el lector ya domina. Con este perfil, las comparaciones con tecnologías fullstack comunes (SQL, C#, java, html, css, Docker...) pueden ir en el cuerpo del texto como explicación principal; reserva el blockquote opcional (`> Si ya conoces X, piensa en Y como...`) para analogías con tecnologías menos universales, que nunca deben ser requisito para entender el texto.
4. **Tutoriales autónomos, no atados a ningún proyecto.** El lector puede ser cualquiera, sin acceso a un código concreto. No menciones proyectos, módulos ni dominios privados. Para ilustrar "para qué se usa", emplea escenarios genéricos y reconocibles (una tienda online, un blog, una app de tareas, un formulario de registro...). La guía debe seguir teniendo sentido fuera de cualquier repositorio.
5. **Guías completas, con entrada suave.** No son resúmenes ni chuletas: el objetivo es que quien las lea acabe **sabiendo usar la tecnología de verdad**, sin tener que ir a buscar lo importante a otro sitio. La ficha arranca siempre con una **introducción accesible** (qué es, por qué existe, para qué sirve) para que nadie se pierda en el primer minuto, y a partir de ahí desarrolla el tema con la profundidad que pida: casos de uso reales, opciones de configuración relevantes, cómo encaja con lo que ya usa el lector, qué pasa cuando algo falla. La extensión la marca el tema, no una cuota: cubre lo que hay que cubrir y para cuando esté cubierto. Lo que sigue prohibido es el **relleno**: repetir lo ya dicho, parafrasear la documentación oficial o enumerar la API entera sin explicar nada.
6. **Todo lo que se explica sobre código va acompañado de un ejemplo guiado.** Nunca describas en prosa una sintaxis, una llamada, una opción de configuración o un patrón sin enseñar el código correspondiente. Y el ejemplo no es un bloque suelto: es *guiado*, es decir, va precedido de una frase que dice qué vamos a ver y seguido (o comentado por dentro) de la explicación de qué hace cada parte y qué resultado produce. Cuando ayude, muestra también la **salida esperada** (respuesta JSON, log de consola, error concreto) o el contraste **antes/después** o **mal/bien**. Regla práctica: si un párrafo habla de código y no hay snippet cerca, falta el snippet.
7. **Género del lector: preferentemente neutro, y si no, masculino.** Por convención, redacta en **género neutro** siempre que se pueda: fórmulas impersonales, «quien…», «la persona…», segunda persona («tú», «necesitas», «verás») sin marca de género, y reformulaciones que eviten adjetivos o sustantivos con género. Cuando el neutro resulte forzado o artificioso, usa el **masculino genérico** (no el femenino). Mantén el criterio coherente dentro de cada colección. No hace falta reescribir fichas antiguas ya redactadas en otro género: esta preferencia aplica al contenido nuevo.

## Convención de nombres de archivo

- Nombre propio simple → ese nombre: `Dapper.md`, `React.md`, `TypeScript.md`.
- Paquete npm con `@scope/` o puntos → reemplaza `/`, `@` y `.` por guiones: `@testing-library/react` → `testing-library-react.md`.
- Patrones/conceptos → nombre descriptivo con guiones: `Clean-Architecture.md`, `Layer-Pattern.md`.

## Formato de la ficha

Hay **un solo formato** para todas las fichas nuevas: la guía completa. No existe ya una "variante compacta" — aunque el tema parezca pequeño, se escribe con este esqueleto (si de verdad da para muy poco, las secciones intermedias serán pocas y cortas, pero la estructura es la misma).

El esqueleto tiene **apertura y cierre fijos** y un **desarrollo intermedio libre**:

```markdown
# <NombreTecnologia>

## ¿Qué es?                          ← FIJA

## ¿Por qué existe?                  ← FIJA

## <las secciones que pida el tema>  ← LIBRES: tantas como haga falta
## <...>
## <...>

## Buenas prácticas avanzadas        ← FIJA

## Recursos didácticos               ← FIJA (omitible si no hay nada que valga la pena)

---

*En resumen: ...*                    ← FIJA
```

### Apertura fija: la introducción

Estas dos secciones abren **siempre** la ficha, en este orden, y son deliberadamente **cortas y accesibles**: sirven para que quien no conoce nada del tema no se pierda antes de llegar al desarrollo.

```markdown
## ¿Qué es?

Una o dos frases que definan qué es, en lenguaje llano.

## ¿Por qué existe?

El problema que resuelve, explicado para alguien que llega de nuevo. Analogía
sencilla y, si ayuda, un blockquote opcional:

> Si ya conoces X, piensa en esta tecnología como "...".
```

Justo después conviene (casi siempre) una sección de contexto tipo `## ¿Cuándo y para qué se usa?`: en qué situaciones aparece y qué problemas reales resuelve, con escenarios genéricos (una tienda online, un blog, una app de tareas...). No es obligatoria, pero si la omites asegúrate de que el "para qué" queda claro en otro sitio.

### Desarrollo libre: el cuerpo de la guía

Aquí es donde la ficha deja de ser una introducción. **Elige las secciones `##` que pida la tecnología** y ordénalas como una progresión de aprendizaje: de lo básico a lo avanzado, cada sección apoyándose en la anterior. Usa subsecciones `###` cuando una sección crezca.

No hay lista cerrada de secciones. Estas son ideas frecuentes, no una plantilla que rellenar (usa solo las que aporten, y renómbralas para que digan algo concreto del tema):

- Instalación / puesta en marcha, y un primer ejemplo que funcione de principio a fin.
- Conceptos y vocabulario propios de la tecnología, definidos antes de usarlos.
- Uso habitual, caso por caso, cada uno con su ejemplo.
- Configuración: las opciones que de verdad se tocan, y qué cambia cada una.
- Cómo encaja con el resto del stack (integraciones, alternativas, con qué se combina).
- Errores frecuentes y cómo se diagnostican, con el mensaje de error real.
- Rendimiento, seguridad o límites, si son relevantes en esa tecnología.
- Un ejemplo completo de cierre que junte varias piezas.

Reglas del desarrollo:

- **Progresión, no catálogo.** Cada sección debe poder leerse sabiendo solo lo anterior. No adelantes conceptos sin definir.
- **Todo lo que toque código lleva su ejemplo guiado** (principio 6): frase que introduce → snippet → qué hace y qué devuelve. Y cuando aporte, la salida esperada o el contraste mal/bien.
- **Profundidad sí, relleno no.** Si una sección no enseña nada que el lector no dedujera ya, fuera.
- **Sin enumerar la API entera.** Para eso está la documentación oficial; enlázala. La guía cubre lo que se usa de verdad y explica *por qué*.

### Cierre fijo

```markdown
## Buenas prácticas avanzadas

Lo que distingue a quien domina la tecnología de quien solo la usa. Entre 3 y 6
puntos con título en negrita: hábitos de nivel experto, errores sutiles que casi
todo el mundo comete, decisiones de diseño o rendimiento que marcan la diferencia
en producción.

- **<Práctica>** — en qué consiste, por qué la aplican los mejores y qué pasa si no.
- **<Error sutil a evitar>** — por qué es fácil caer en él y cómo detectarlo.

## Documentación oficial

Las fuentes canónicas de la tecnología, para que quien lea la ficha sepa a dónde
ir cuando necesite el detalle que la guía no cubre o quiera contrastarlo. De 1 a
4 enlaces, y cada uno con una línea que diga **qué parte merece la pena y cuándo
ir ahí** — un enlace pelado no aporta nada.

- [<Documentación del proyecto>](url) — qué cubre bien y qué sección consultar primero.
- [<Especificación / RFC>](url) — la fuente normativa, para cuando la duda es "¿qué dice el estándar?".
- [<Repositorio>](url) — útil si el código fuente responde preguntas que la documentación no.

## Recursos didácticos

Recursos que ayuden a **fijar o explorar** el concepto: herramientas
interactivas, visualizaciones, comparativas, calculadoras, buenos artículos de
terceros. Si no hay ninguno que valga la pena, omite la sección. Y si encuentras
alguno divertido o interactivo, mejor todavía: por ejemplo, para explicar los
códigos de error HTTP, mencionar https://http.cat/.

---

*En resumen: <una frase memorable que capture la esencia de la tecnología>.*
```

> El título de la sección es siempre `## Recursos didácticos` (sin "divertidos"), aunque el recurso concreto sea divertido.

> **La división entre las dos secciones finales.** `Documentación oficial` es la fuente canónica; `Recursos didácticos` es todo lo que ayuda a aprender pero no es la fuente. Un enlace no va en las dos. Si la tecnología no tiene fabricante (un patrón, un concepto como el caching o las migraciones sin downtime), la fuente canónica es la que haya: un RFC, una *cheat sheet* de OWASP, la especificación del motor de base de datos, o el artículo que acuñó el término. Si de verdad no existe ninguna fuente canónica, omite la sección — pero antes piénsalo dos veces, porque casi siempre hay una.

### Reglas de formato comunes

- **Nombres genéricos y reconocibles** en el código (`User`, `productId`, `/api/products`, `Order`...), nunca `foo`/`bar` ni nombres de un proyecto privado. Mantén los mismos nombres a lo largo de la ficha: si el ejemplo de la sección 2 usa `Product`, el de la 6 también.
- Snippets **cortos y centrados en una idea**. Si un ejemplo es largo, pártelo en pasos con su explicación entre medias en vez de soltar un muro de código.
- Lenguaje de los snippets coherente con la tecnología (`csharp`, `tsx`/`ts`, `python`, `bash`/`yaml`...), siempre declarado en el bloque.
- El cierre en cursiva `*En resumen: ...*`, precedido de `---`, es obligatorio.
- En «Buenas prácticas avanzadas», cada punto debe ser accionable y específico de la tecnología (el criterio: ¿esto lo sabe solo el 1% que de verdad domina el tema?). Nada de consejos genéricos tipo "escribe tests" o "lee la documentación". Si un punto necesita código para entenderse, un snippet corto está permitido.
- Los `---` separadores entre secciones son opcionales; úsalos con coherencia dentro de una misma ficha (o en todas, o en ninguna, salvo el `---` obligatorio antes del `*En resumen:*`).

## Formato del README-índice

El `README.md` de la carpeta hace de índice. Modelo: `devops/despliegue-en-vps/README.md`. Estructura:

1. Título `# <Tema> — Guía de tecnologías` (o un subtítulo adecuado).
2. Párrafo introductorio (a quién va dirigido, qué encontrará).
3. `---`
4. `## Orden de lectura recomendado`: si hay varios bloques temáticos, subsecciones numeradas (`### 1. ...`), cada una con una frase de contexto y una tabla. La numeración de la columna `#` es **continua** (no se reinicia entre subsecciones).

   ```markdown
   | # | Archivo | Por qué leerlo aquí |
   |---|---|---|
   | 1 | [Concepto](Concepto.md) | Una frase de por qué leerlo en esta posición. |
   ```
5. `---`
6. (Opcional, si hay muchas fichas) `## Índice completo` dentro de un bloque colapsable `<details><summary>Ver todos los archivos</summary>` con los enlaces agrupados.
7. Nota de cierre opcional en blockquote enlazando a temas relacionados.

## Flujo de trabajo

Cuando se pida crear uno o varios tutoriales:

1. **Lee los ejemplos** dentro de `<raíz>` para calibrar tono (no extensión: ver «Antes de escribir nada»).
2. **Determina la categoría y el tema, y crea la carpeta** dentro de la categoría correspondiente (bajo `<raíz>`, kebab-case). Si la categoría, el nombre o el alcance no están claros, pregunta.
3. **Esboza el índice de secciones de cada ficha** antes de redactarla: apertura fija + las secciones `##` que pida el tema, ordenadas como progresión de aprendizaje + cierre fijo. Si el tema es amplio, plantéate si conviene partirlo en varias fichas enlazadas desde el README en vez de una sola inabarcable.
4. **Escribe las fichas** respetando el esqueleto, con ejemplos genéricos y autónomos, y un ejemplo guiado cada vez que se hable de código.
5. **Crea o actualiza los `README.md`-índice afectados**, de dentro hacia fuera:
   - **El de la carpeta del tema**: añade cada ficha a la tabla de orden de lectura (en su posición temática, renumerando `#` si hace falta) y, si procede, al índice colapsable.
   - **El de la categoría**: si el tema es nuevo, enlázalo desde el `README.md` de su categoría; si el tema ya existía pero ha cambiado de alcance, revisa que su descripción siga siendo fiel.
   - **El `README.md` raíz** (`<raíz>/README.md`): revísalo **siempre**, no solo cuando la categoría es nueva. Tiene una descripción de una línea por categoría; si el trabajo que acabas de hacer amplía o cambia lo que esa categoría cubre, actualiza esa línea para que no se quede obsoleta. Si la categoría es nueva, añádele su entrada.
6. **Verifica los enlaces.** Toda ruta relativa debe resolver (nombre de archivo exacto, mayúsculas incluidas). Ten en cuenta la profundidad: un enlace a un tema de otra categoría necesita `../../`, no `../`.
7. **Haz commit y push directamente, sin pedir confirmación, si la sesión ha sido satisfactoria.** Cuando el trabajo esté terminado y verificado (todas las fichas creadas, el índice actualizado y los enlaces comprobados), haz `git add` **solo de los ficheros y carpetas que ha tocado esta skill** (nunca `git add -A` ni `git add .`, para no arrastrar cambios ajenos que ya estuvieran en el working tree), un `git commit` con un mensaje descriptivo en español (p. ej. `Añade guía de <tema>`) y un `git push`. **Todos estos comandos de git se ejecutan sobre el repositorio destino `<raíz>`**: si el directorio de trabajo actual es otro, usa `git -C "C:\Users\MaccionUser\GitRepos\Learning" …`. Esta es una instrucción permanente de la usuaria: no hace falta preguntar "¿hago commit y push?" cada vez, el paso 7 ya lo autoriza. Si algo quedó incompleto, falló, la usuaria no está conforme, o el trabajo se hizo en varios subagentes en paralelo y conviene una revisión de conjunto antes de subir, **no** hagas commit ni push: deja los cambios en el working tree y coméntalo explicando por qué.

## Checklist final

Antes de dar por terminado:

- [ ] El contenido se ha creado dentro del repositorio destino `<raíz>` (`C:\Users\MaccionUser\GitRepos\Learning`), no en el directorio de trabajo actual si era otro.
- [ ] El contenido está en una carpeta de tema dentro de la categoría que le corresponde.
- [ ] Cada ficha está en español, dirigida a un perfil backend junior-medio: sin explicar fundamentos de programación, pero definiendo los términos propios de la tecnología.
- [ ] Es autónoma: se entiende sin conocer ningún proyecto concreto (sin módulos ni dominios privados).
- [ ] El género usado para el lector es coherente en toda la colección.
- [ ] Cada ficha respeta el esqueleto: abre con `## ¿Qué es?` y `## ¿Por qué existe?`, y cierra con «Buenas prácticas avanzadas», «Recursos didácticos» (si hay) y el `*En resumen: ...*`.
- [ ] Es una **guía completa**, no un resumen: el desarrollo intermedio va a fondo y quien la lea sabe usar la tecnología sin tener que buscar lo importante en otro sitio.
- [ ] Las secciones intermedias forman una progresión de aprendizaje (nada se usa antes de definirse) y ninguna es relleno.
- [ ] **No hay ningún párrafo que hable de código sin su ejemplo guiado al lado** (frase introductoria → snippet → qué hace y qué devuelve).
- [ ] «Buenas prácticas avanzadas» tiene puntos específicos y accionables, no consejos genéricos.
- [ ] El código de ejemplo usa nombres genéricos y reconocibles, no `foo`/`bar` ni nombres de un proyecto privado, y son coherentes entre secciones de la misma ficha.
- [ ] Cada ficha cierra con `---` y la frase `*En resumen: ...*`.
- [ ] La carpeta tiene un `README.md`-índice con enlaces a todas las fichas, y todos los enlaces funcionan.
- [ ] Se han revisado los índices de nivel superior: el `README.md` de la categoría y el `README.md` raíz, actualizando sus descripciones si el nuevo contenido cambia lo que la categoría cubre.
- [ ] Si la sesión ha sido satisfactoria, se ha hecho `commit` y `push` de los cambios **sin pedir confirmación previa** (staging selectivo, solo lo tocado por esta skill), y si no lo ha sido, los cambios se han dejado sin subir y se ha avisado explicando por qué.
