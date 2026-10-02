# Markdown a HTML en build-time

## ¿Qué es?

Convertir ficheros Markdown en HTML **antes** de que la aplicación llegue al navegador, con un script que se ejecuta una vez (en local o en el build) y deja el resultado listo para usar. En esta guía se hace con [`marked`](https://marked.js.org/) y un script de Node.

## ¿Por qué existe?

Una página de «Ayuda» o un manual de usuario se escribe mucho más cómodo en Markdown que en HTML: se versiona bien, se revisa en un PR sin ruido y lo puede editar quien no programa. Pero el navegador no entiende Markdown, así que alguien tiene que convertirlo.

Hay dos momentos posibles para hacerlo:

| | En runtime (en el navegador) | En build-time (script previo) |
|---|---|---|
| Cuándo se convierte | Cada vez que alguien abre la página | Una vez, al generar |
| Qué viaja al cliente | El `.md` **y** la librería de conversión | Solo el HTML ya hecho |
| Coste en el cliente | Descarga de la librería + CPU en cada visita | Ninguno |
| Errores en el Markdown | Los descubre el usuario | Los descubre el script, antes del despliegue |

Si el contenido cambia solo cuando alguien edita el repositorio (un manual, unas FAQ, unas condiciones legales), convertir en runtime es repetir en cada visita un trabajo que ya se podía haber hecho.

> Si ya conoces la compilación de SASS a CSS, piensa en esto igual: el navegador recibe el resultado, no la fuente ni el compilador.

## ¿Cuándo y para qué se usa?

Es el patrón habitual para contenido **propio y estático**: la página de ayuda de una app de tareas, el manual de una tienda online, un changelog o una sección de preguntas frecuentes. No sirve para contenido que escribe el usuario en la propia app ni para contenido que cambia sin redesplegar (ese viene de una API y se trata aparte; ver [dangerouslySetInnerHTML y XSS](dangerouslySetInnerHTML-y-XSS.md)).

El ejemplo conductor es una app de tareas con una página de ayuda escrita en `docs/ayuda.md`, que acaba convertida en un módulo TypeScript que el componente importa.

## Instalación y primer ejemplo de principio a fin

```bash
pnpm add -D marked
```

Va como dependencia de desarrollo porque solo la usa el script, no el código que llega al navegador. Esta guía usa `marked` 18; las APIs cambian entre versiones mayores, así que comprueba la tuya si usas otra.

Contenido de `docs/ayuda.md`:

```markdown
# Ayuda

## Crear una tarea

Pulsa **Nueva tarea**, escribe un título y guarda.

## Archivar una tarea

Abre la tarea y elige *Archivar*.
```

Y el script `scripts/generar-ayuda.mjs` (la extensión `.mjs` hace que Node lo trate como módulo ESM sin configurar nada más):

```js
import { readFileSync, writeFileSync } from 'node:fs'
import { marked } from 'marked'

const markdown = readFileSync('docs/ayuda.md', 'utf-8')
const html = marked.parse(markdown)

writeFileSync('src/ayuda.generated.ts', `export const AYUDA_HTML = ${JSON.stringify(html)}\n`)
```

Se ejecuta con `node scripts/generar-ayuda.mjs`. Qué hace: lee el Markdown, `marked.parse` devuelve el HTML como `string` y se escribe un módulo TypeScript que exporta ese HTML. El resultado en `src/ayuda.generated.ts` empieza así:

```ts
export const AYUDA_HTML = "<h1>Ayuda</h1>\n<h2>Crear una tarea</h2>\n<p>Pulsa <strong>Nueva tarea</strong>..."
```

`marked.parse` es síncrono mientras no uses extensiones asíncronas; si activas la opción `async`, devuelve una `Promise` y hay que hacer `await`.

Para no teclear la ruta, se registra como script de `package.json`:

```json
{
  "scripts": {
    "gen:ayuda": "node scripts/generar-ayuda.mjs"
  }
}
```

## Un renderer propio: reescribir solo lo que necesitas

Problema típico: en el Markdown las imágenes se referencian con rutas relativas a la carpeta de documentación (`![Lista](images/lista.png)`), pero en la app se sirven desde `/ayuda/images/`. Hay que reescribir esa ruta.

La tentación es hacer un `html.replace('images/', '/ayuda/images/')` sobre el HTML final. Es frágil: también tocaría un párrafo que mencione «images/» como texto. La forma correcta es engancharse al **renderer** de `marked`, que se invoca solo cuando se renderiza una imagen:

```js
import { Marked } from 'marked'

const marked = new Marked({
  renderer: {
    image({ href, title, text }) {
      const src = href.startsWith('images/') ? `/ayuda/${href}` : href
      const tituloAttr = title ? ` title="${title}"` : ''
      return `<img src="${src}" alt="${text}"${tituloAttr}>`
    },
  },
})

marked.parse('![Lista de tareas](images/lista.png)')
// <p><img src="/ayuda/images/lista.png" alt="Lista de tareas"></p>
```

Dos detalles de la API:

- `new Marked({...})` crea una instancia con su configuración propia, en lugar de modificar el `marked` global.
- En versiones recientes, cada método del renderer recibe **un objeto token** (`{ href, title, text }`), no argumentos sueltos como en versiones antiguas. Si copias ejemplos de internet que usan `image(href, title, text)`, son de una versión anterior.

Un renderer que devuelve `false` (o `undefined`) delega en el comportamiento por defecto, útil si solo quieres tocar algunos casos.

## Dividir en secciones y generar slugs

Una página de ayuda suele querer un índice lateral o navegar por secciones. Para eso hace falta partir el Markdown por encabezados `## ` y dar a cada sección un identificador estable (un *slug*) para usarlo como ancla (`#crear-una-tarea`).

`marked` no genera ids en los encabezados (lo hacía en versiones antiguas), así que lo haces tú:

```js
export function calcularSlug(titulo) {
  return titulo
    .toLowerCase()
    .replace(/[^\p{L}\p{N}\s-]/gu, '') // quita signos; \p{L} conserva tildes y eñes
    .trim()
    .replace(/\s+/g, '-')
}

export function dividirEnSecciones(markdown) {
  return markdown
    .split(/\n(?=## )/) // corta antes de cada línea que empieza por "## "
    .filter((bloque) => bloque.startsWith('## '))
    .map((bloque) => {
      const [primeraLinea, ...resto] = bloque.split('\n')
      return {
        titulo: primeraLinea.replace(/^##\s*/, '').trim(),
        cuerpo: resto.join('\n'),
      }
    })
}
```

Con el `ayuda.md` de antes, `dividirEnSecciones` devuelve dos secciones y `calcularSlug('Crear una tarea')` da `crear-una-tarea`. La expresión `(?=## )` es un *lookahead*: corta sin consumir el `## `, de modo que cada bloque conserva su encabezado. El `filter` descarta lo que haya antes de la primera sección (el `# Ayuda` inicial).

Cada sección se convierte por separado y se emite con su id:

```js
const secciones = dividirEnSecciones(markdown).map((s) => ({
  id: calcularSlug(s.titulo),
  titulo: s.titulo,
  html: marked.parse(s.cuerpo.trim()),
}))
```

## El módulo generado: commitearlo o generarlo en cada build

El script puede escribir un módulo TypeScript con el resultado:

```ts
// GENERADO por scripts/generar-ayuda.mjs (`pnpm gen:ayuda`). NO editar a mano.
export interface SeccionAyuda {
  id: string
  titulo: string
  html: string
}

export const SECCIONES_AYUDA: SeccionAyuda[] = [
  { id: 'crear-una-tarea', titulo: 'Crear una tarea', html: '<p>Pulsa <strong>Nueva tarea</strong>…</p>' },
]
```

Para escribir cadenas con HTML (que lleva comillas dobles en los atributos) dentro de un módulo, usa un escapado correcto de `\`, comillas y saltos de línea, o simplemente `JSON.stringify`, que ya lo hace.

La cabecera «GENERADO, no editar» importa: quien abra el fichero sabrá que sus cambios se perderán.

Hay dos estrategias para que ese fichero exista:

| | Commitear el generado | Generarlo en cada build (y `.gitignore`) |
|---|---|---|
| Build reproducible | Sí, no depende del script ni de la versión de `marked` | Depende de que el script funcione en el entorno de CI |
| Revisión en PR | Se ve en el diff cómo cambia el HTML resultante | Solo se revisa el `.md` |
| Riesgo | **Desfase**: alguien edita el `.md` y olvida regenerar; la app muestra la versión vieja | Ninguno de desfase, pero un paso más en cada build |
| Ruido en el repo | Diffs largos de líneas enormes | Ninguno |

Ninguna es la correcta siempre. Si commiteas el generado (lo razonable cuando el contenido cambia poco y quieres builds simples), **cubre el riesgo de desfase**: un test o un paso de CI que regenere en memoria y compare con el fichero commiteado, de forma que falle si no coinciden.

```js
// generar-ayuda.test.mjs (con node:test)
import { test } from 'node:test'
import assert from 'node:assert/strict'
import { readFileSync } from 'node:fs'
import { construirModulo } from './generar-ayuda.mjs'

test('el módulo generado está al día con ayuda.md', () => {
  const esperado = construirModulo(readFileSync('docs/ayuda.md', 'utf-8'))
  const actual = readFileSync('src/ayuda.generated.ts', 'utf-8')
  assert.equal(actual, esperado)
})
```

Esto exige separar la función pura que **construye** el texto del módulo de la que **lo escribe a disco**, que es una buena división de todos modos.

## Que el script sirva como módulo y como comando

Conviene que el mismo fichero exporte sus funciones (para testearlas) y, además, haga su trabajo cuando se ejecuta con `node scripts/generar-ayuda.mjs`. Para eso hay que detectar «me han lanzado como script». La versión que se ve en muchos sitios **no funciona en Windows**:

```js
// ❌ nunca coincide en Windows
if (import.meta.url === `file://${process.argv[1]}`) { /* ... */ }
```

`import.meta.url` es una URL (`file:///C:/proyecto/scripts/generar-ayuda.mjs`, con barras normales y triple barra), mientras que `process.argv[1]` es una ruta del sistema (`C:\proyecto\scripts\generar-ayuda.mjs`). Concatenar `file://` no las iguala. Conviértela con `pathToFileURL`:

```js
import { pathToFileURL } from 'node:url'

// ✅ funciona en Windows, macOS y Linux
if (import.meta.url === pathToFileURL(process.argv[1]).href) {
  generar()
}
```

Importar el módulo desde un test no dispara `generar()`, porque `process.argv[1]` es entonces el ejecutor de tests, no este fichero.

## Errores frecuentes

- **Editar el fichero generado a mano.** Se pierde en la siguiente regeneración. La cabecera «GENERADO» y la revisión en PR lo evitan.
- **Olvidar regenerar tras editar el Markdown.** Es el desfase de la sección anterior; se detecta con el test de comparación.
- **Ejemplos con la firma antigua del renderer.** `image(href, title, text)` en lugar de `image({ href, title, text })` produce `undefined` en las rutas. Comprueba la versión instalada.
- **Reemplazos de texto sobre el HTML final** en vez de un renderer: tocan lo que no debían.
- **Esperar que `marked` limpie el HTML.** No lo hace: si el Markdown lleva HTML incrustado (`<script>`), sale tal cual. Con contenido propio no es un problema; con contenido ajeno, sí (ver [dangerouslySetInnerHTML y XSS](dangerouslySetInnerHTML-y-XSS.md)).
- **Rutas relativas al lanzar el script desde otra carpeta.** `readFileSync('docs/ayuda.md')` depende del directorio de trabajo. Resuélvelas a partir de `import.meta.dirname` (Node 20.11 o superior).

## Buenas prácticas avanzadas

- **Separa construir de escribir.** Una función pura `(markdown) => textoDelMódulo` se testea sin tocar disco, y el test de «generado al día» sale casi gratis.
- **Usa una instancia `new Marked({...})`, no el `marked` global.** Evita que otro código del mismo proceso cambie la configuración que esperas.
- **Escapa lo que metes en atributos desde el renderer.** Si en `image` interpolas `alt` o `title` sin escapar, un texto con comillas rompe el HTML. Con Markdown propio es poco probable, pero es el tipo de fallo que aparece el día que alguien escribe `"cita"` en un alt.
- **Falla ruidosamente.** Si el script espera una cabecera de versión o un `## Índice` y no los encuentra, lanza un `Error` con un mensaje claro en lugar de generar un módulo vacío que nadie nota.
- **Mantén el slug estable.** Las anclas (`#crear-una-tarea`) acaban en enlaces guardados y en otras páginas; renombrar un título cambia su slug. Decide si te importa antes de renombrar.
- **Fija la versión de `marked`.** Dos versiones pueden producir HTML ligeramente distinto; con el generado commiteado, una actualización se ve como un diff en el PR, y eso es una ventaja si la esperas.

## Documentación oficial

- [marked — documentación](https://marked.js.org/) — la sección de uso (`marked.parse`) y la de «Using advanced» para opciones y renderers; consúltala al personalizar la salida.
- [marked — repositorio](https://github.com/markedjs/marked) — los *releases* listan los cambios incompatibles entre versiones mayores, imprescindibles si un ejemplo antiguo no funciona.
- [Node.js — módulos ESM](https://nodejs.org/api/esm.html) — referencia de `import.meta.url` e `import.meta.dirname`.

---

*En resumen: convierte el Markdown a HTML una sola vez en un script, toca solo lo necesario desde el renderer de `marked`, y si commiteas el resultado, añade un test que avise cuando se desfase de la fuente.*
