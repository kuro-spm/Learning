---
name: crear-tutorial
description: >-
  Crea colecciones de "tutoriales" o guías de tecnología siguiendo una metodología fija: una carpeta
  nueva en la raíz del proyecto con un README que hace de índice (con enlaces) y fichas en español,
  autónomas y dirigidas a perfiles fullstack junior-medio, legibles por cualquiera sin contexto de un
  proyecto concreto.
  Usa esta skill siempre que la usuaria pida crear, añadir, escribir o documentar un tutorial, una guía,
  una ficha o una colección de guías sobre una librería, framework, lenguaje, herramienta o concepto,
  aunque no use literalmente la palabra "tutorial".
---

# Crear tutorial (guía de tecnología)

Esta skill genera **colecciones de guías de tecnología** replicando una metodología concreta: una carpeta con un README que actúa de índice (enlazando a cada documento) y una ficha por tecnología o concepto. La prioridad número uno es la **consistencia de formato** entre todas las fichas de la colección.

Los tutoriales son **autónomos y genéricos**: cualquier persona debe poder leerlos sin conocer ningún proyecto concreto. Explican la tecnología en sí (qué es, por qué existe, cómo se usa), no cómo se aplica en un código privado.

> **Autorización permanente de commit y push.** La usuaria ha autorizado de antemano, como parte de esta skill, que al terminar y verificar un trabajo se haga `git add` + `git commit` + `git push` sin pedir confirmación adicional (ver «Flujo de trabajo», paso 7). Esta autorización cubre específicamente el contenido creado por esta skill; no autoriza a subir cambios ajenos que estuvieran ya presentes en el working tree antes de empezar la tarea (esos se dejan tal cual, sin stagear).

## Dónde se crea el contenido

El proyecto agrupa las colecciones de tutoriales en **carpetas de categoría** en la raíz. Cada tema vive dentro de la categoría a la que pertenece:

```
<raíz>/
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

Lee 2-3 documentos ya existentes en el repositorio para calibrar **estructura y tono** (no su contenido). Son la referencia de formato por encima de esta skill si hay discrepancias, **con una excepción**: la sección «Buenas prácticas avanzadas» es una incorporación posterior y algunas fichas antiguas no la tienen; inclúyela igualmente en las fichas nuevas.

- `bases-de-datos/acceso-a-datos-dotnet/Dapper.md` — ejemplo de **ficha completa**.
- `desarrollo-web/frontend-react/clsx.md` — ejemplo de **ficha compacta**.
- `desarrollo-web/frontend-react/README.md` — ejemplo de **README-índice**.

## Principios fundamentales

Estos pilares no se negocian:

1. **Idioma: español.** La prosa va en español. Nombres de tecnologías, comandos, código y términos técnicos consolidados (*proxy*, *bundler*, *change tracking*) se mantienen en su idioma original.
2. **Audiencia: perfil backend junior-medio.** El lector programa a diario y puede conocer algunas partes del stack(HTTP, APIs REST, bases de datos, componentes de UI, npm/NuGet, Git...), pero puede ser nuevo en la tecnología concreta que documentas. No le expliques qué es una API o un bucle; sí define los términos propios de la tecnología antes de usarlos y no des por sabidos sus detalles internos.
3. **Analogías y lenguaje claro.** Explica cada concepto nuevo anclándolo a algo que el lector ya domina. Con este perfil, las comparaciones con tecnologías fullstack comunes (SQL, C#, java, html, css, Docker...) pueden ir en el cuerpo del texto como explicación principal; reserva el blockquote opcional (`> Si ya conoces X, piensa en Y como...`) para analogías con tecnologías menos universales, que nunca deben ser requisito para entender el texto.
4. **Tutoriales autónomos, no atados a ningún proyecto.** El lector puede ser cualquiera, sin acceso a un código concreto. No menciones proyectos, módulos ni dominios privados. Para ilustrar "para qué se usa", emplea escenarios genéricos y reconocibles (una tienda online, un blog, una app de tareas, un formulario de registro...). La guía debe seguir teniendo sentido fuera de cualquier repositorio.
5. **Brevedad introductoria.** Son guías de *introducción*, no documentación exhaustiva. El lector debe poder leerlas en pocos minutos y quedarse con "lo mínimo para no perderse". La única sección que mira más allá de la introducción es «Buenas prácticas avanzadas», y aun así debe ser concisa: pocos puntos, muy escogidos.
6. **Género del lector: preferentemente neutro, y si no, masculino.** Por convención, redacta en **género neutro** siempre que se pueda: fórmulas impersonales, «quien…», «la persona…», segunda persona («tú», «necesitas», «verás») sin marca de género, y reformulaciones que eviten adjetivos o sustantivos con género. Cuando el neutro resulte forzado o artificioso, usa el **masculino genérico** (no el femenino). Mantén el criterio coherente dentro de cada colección. No hace falta reescribir fichas antiguas ya redactadas en otro género: esta preferencia aplica al contenido nuevo.

## Convención de nombres de archivo

- Nombre propio simple → ese nombre: `Dapper.md`, `React.md`, `TypeScript.md`.
- Paquete npm con `@scope/` o puntos → reemplaza `/`, `@` y `.` por guiones: `@testing-library/react` → `testing-library-react.md`.
- Patrones/conceptos → nombre descriptivo con guiones: `Clean-Architecture.md`, `Layer-Pattern.md`.

## Formato de la ficha

Elige una de las dos variantes según la importancia del tema. Las secciones son fijas: **no inventes secciones nuevas**.

### Variante completa (temas centrales)

Para tecnologías o conceptos importantes. Modelo: `technologies/backend/Dapper.md`.

```markdown
# <NombreTecnologia>

## ¿Qué es?

Una o dos frases que definan qué es, en lenguaje llano.

## ¿Por qué existe?

El problema que resuelve, explicado para alguien que llega de nuevo. Analogía
sencilla y, si ayuda, un blockquote opcional:

> Si ya conoces X, piensa en esta tecnología como "...".

## ¿Cuándo y para qué se usa?

En qué situaciones aparece y qué problemas reales resuelve, con ejemplos genéricos
(una tienda online, un blog, una app de tareas...). Sin atarlo a ningún proyecto.

## Lo mínimo que necesitas saber

Lista numerada de conceptos/usos esenciales, cada uno con título en negrita y un
bloque de código corto y realista con nombres genéricos y reconocibles.

**1. <Caso de uso>**

​```csharp
// código mínimo y realista con nombres genéricos (User, productId, /api/products...)
​```

**2. <Otro caso>**

​```csharp
...
​```

## Lo que NO hace

- **No hace X** — explicación corta.
- **No hace Y** — explicación corta.

## Buenas prácticas avanzadas

Lo que distingue a quien domina la tecnología de quien solo la usa. Entre 3 y 6
puntos con título en negrita: hábitos de nivel experto, errores sutiles que casi
todo el mundo comete, decisiones de diseño o rendimiento que marcan la diferencia
en producción.

- **<Práctica>** — en qué consiste, por qué la aplican los mejores y qué pasa si no.
- **<Error sutil a evitar>** — por qué es fácil caer en él y cómo detectarlo.

## Recursos didácticos

Recursos que ayuden a fijar o explorar el concepto (documentación navegable, herramientas interactivas, visualizaciones...), si existen y aportan de verdad; si no hay ninguno que valga la pena, omite la sección. Y si encuentras alguno **divertido o interactivo**, mejor todavía: por ejemplo, para explicar los códigos de error HTTP, mencionar https://http.cat/.

> El título de la sección es siempre `## Recursos didácticos` (sin "divertidos"), aunque el recurso concreto sea divertido.


---

*En resumen: <una frase memorable que capture la esencia de la tecnología>.*
```

### Variante compacta (temas pequeños)

Para utilidades o conceptos menores. Modelo: `technologies/frontend/clsx.md`. Igual que la completa pero:

- Abre con `**¿Qué es?**` en negrita inline (no como encabezado `##`) seguido de la definición y un `---`.
- El resto de secciones se mantienen, separadas por `---`.
- «Buenas prácticas avanzadas» es *opcional* en esta variante: inclúyela (con 2-3 puntos) solo si el tema da para consejos de nivel experto que no quepan en «Lo mínimo»; en una utilidad trivial es mejor omitirla que rellenarla con obviedades.
- Cierra igual: `*En resumen: ...*`.

### Reglas comunes a ambas variantes

- Bloques de código cortos y realistas con nombres genéricos y reconocibles (`User`, `productId`, `/api/products`, `Order`...), nunca `foo`/`bar` ni nombres de un proyecto privado.
- El cierre en cursiva `*En resumen: ...*`, precedido de `---`, es obligatorio.
- Lenguaje de los snippets coherente con la tecnología (`csharp`, `tsx`/`ts`, `python`, `bash`/`yaml`...).
- En «Buenas prácticas avanzadas», cada punto debe ser accionable y específico de la tecnología (el criterio: ¿esto lo sabe solo el 1% que de verdad domina el tema?). Nada de consejos genéricos tipo "escribe tests" o "lee la documentación". Si un punto necesita código para entenderse, un snippet corto está permitido.

## Formato del README-índice

El `README.md` de la carpeta hace de índice. Modelo: `technologies/frontend/README.md`. Estructura:

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

1. **Lee los ejemplos** de `technologies/` para calibrar formato y tono.
2. **Determina la categoría y el tema, y crea la carpeta** dentro de la categoría correspondiente (kebab-case). Si la categoría, el nombre o el alcance no están claros, pregunta.
3. **Elige la variante** de cada ficha (completa o compacta) según su importancia.
4. **Escribe las fichas** respetando el formato, con ejemplos genéricos y autónomos.
5. **Crea o actualiza los `README.md`-índice afectados**, de dentro hacia fuera:
   - **El de la carpeta del tema**: añade cada ficha a la tabla de orden de lectura (en su posición temática, renumerando `#` si hace falta) y, si procede, al índice colapsable.
   - **El de la categoría**: si el tema es nuevo, enlázalo desde el `README.md` de su categoría; si el tema ya existía pero ha cambiado de alcance, revisa que su descripción siga siendo fiel.
   - **El `README.md` raíz** (`<raíz>/README.md`): revísalo **siempre**, no solo cuando la categoría es nueva. Tiene una descripción de una línea por categoría; si el trabajo que acabas de hacer amplía o cambia lo que esa categoría cubre, actualiza esa línea para que no se quede obsoleta. Si la categoría es nueva, añádele su entrada.
6. **Verifica los enlaces.** Toda ruta relativa debe resolver (nombre de archivo exacto, mayúsculas incluidas). Ten en cuenta la profundidad: un enlace a un tema de otra categoría necesita `../../`, no `../`.
7. **Haz commit y push directamente, sin pedir confirmación, si la sesión ha sido satisfactoria.** Cuando el trabajo esté terminado y verificado (todas las fichas creadas, el índice actualizado y los enlaces comprobados), haz `git add` **solo de los ficheros y carpetas que ha tocado esta skill** (nunca `git add -A` ni `git add .`, para no arrastrar cambios ajenos que ya estuvieran en el working tree), un `git commit` con un mensaje descriptivo en español (p. ej. `Añade guía de <tema>`) y un `git push`. Esta es una instrucción permanente de la usuaria: no hace falta preguntar "¿hago commit y push?" cada vez, el paso 7 ya lo autoriza. Si algo quedó incompleto, falló, la usuaria no está conforme, o el trabajo se hizo en varios subagentes en paralelo y conviene una revisión de conjunto antes de subir, **no** hagas commit ni push: deja los cambios en el working tree y coméntalo explicando por qué.

## Checklist final

Antes de dar por terminado:

- [ ] El contenido está en una carpeta de tema dentro de la categoría que le corresponde.
- [ ] Cada ficha está en español, dirigida a un perfil fullstack junior-medio: sin explicar fundamentos de programación, pero definiendo los términos propios de la tecnología.
- [ ] Es autónoma: se entiende sin conocer ningún proyecto concreto (sin módulos ni dominios privados).
- [ ] El género usado para el lector es coherente en toda la colección.
- [ ] Cada ficha sigue una de las dos variantes de formato, sin secciones inventadas.
- [ ] Toda ficha completa incluye «Buenas prácticas avanzadas» con puntos específicos y accionables (y las compactas, solo si el tema lo merece).
- [ ] El código de ejemplo usa nombres genéricos y reconocibles, no `foo`/`bar` ni nombres de un proyecto privado.
- [ ] Cada ficha cierra con `---` y la frase `*En resumen: ...*`.
- [ ] La carpeta tiene un `README.md`-índice con enlaces a todas las fichas, y todos los enlaces funcionan.
- [ ] Se han revisado los índices de nivel superior: el `README.md` de la categoría y el `README.md` raíz, actualizando sus descripciones si el nuevo contenido cambia lo que la categoría cubre.
- [ ] Si la sesión ha sido satisfactoria, se ha hecho `commit` y `push` de los cambios **sin pedir confirmación previa** (staging selectivo, solo lo tocado por esta skill), y si no lo ha sido, los cambios se han dejado sin subir y se ha avisado explicando por qué.
