# i18n con claves en React

## ¿Qué es?

La internacionalización (*i18n*, por las 18 letras entre la «i» y la «n») consiste en que la interfaz no lleve los textos escritos a fuego en los componentes, sino **claves** (`'auth.submit'`) que se resuelven en el texto del idioma activo a través de una función `t`.

## ¿Por qué existe?

Una pantalla con textos incrustados solo habla un idioma:

```tsx
// ❌ MAL — para soportar otro idioma hay que tocar cada componente
<button>Iniciar sesión</button>
```

Con claves, el componente no sabe en qué idioma está; solo pide un texto por su identificador y un diccionario por idioma lo resuelve:

```tsx
// ✅ BIEN — el componente es el mismo en todos los idiomas
<button>{t('auth.submit')}</button>
```

> Si ya conoces los ficheros de recursos (`.resx` en .NET, `messages.properties` en Java), es la misma idea: un fichero por idioma con las mismas claves y distinto texto.

## ¿Cuándo y para qué se usa?

En cualquier aplicación que vaya a mostrarse en más de un idioma (una tienda online para varios países, una app interna en castellano y catalán). Aunque hoy solo haya un idioma, tener los textos en un diccionario centraliza el copy y permite revisarlo o corregirlo sin recorrer los componentes.

---

## Los diccionarios

Cada idioma es un objeto con las mismas claves. Agrupar por zona de la aplicación mantiene las claves legibles:

```ts
// locales/es.ts
export const es = {
  common: { loading: 'Cargando…', language: 'Idioma' },
  auth: {
    title: 'Inicia sesión',
    submit: 'Entrar',
    invalidCredentials: 'Credenciales no válidas.',
  },
}
```

```ts
// locales/ca.ts
import type { es } from './es'

export const ca: typeof es = {
  common: { loading: 'Carregant…', language: 'Idioma' },
  auth: {
    title: 'Inicia sessió',
    submit: 'Entra',
    invalidCredentials: 'Credencials no vàlides.',
  },
}
```

Las claves anidadas se escriben con puntos: `t('auth.submit')`. La anotación `typeof es` en el segundo diccionario es el truco para **mantenerlos sincronizados**: si añades una clave en un idioma y olvidas el otro, el compilador falla en lugar de descubrirlo un usuario en producción.

Conviene nombrar las claves por **significado** (`auth.submit`), no por el texto: el texto cambia con el idioma y con las revisiones de redacción, la clave no.

## Puesta en marcha con react-i18next

La librería de referencia en React es `i18next` con su binding `react-i18next`:

```bash
pnpm add i18next react-i18next
```

Se configura una sola vez, antes de renderizar la aplicación:

```ts
// i18n.ts
import i18n from 'i18next'
import { initReactI18next } from 'react-i18next'
import { es } from './locales/es'
import { ca } from './locales/ca'

i18n.use(initReactI18next).init({
  resources: {
    es: { translation: es },
    ca: { translation: ca },
  },
  lng: 'es',            // idioma inicial
  fallbackLng: 'es',    // idioma al que se recurre si falta una clave
  interpolation: { escapeValue: false }, // React ya escapa el contenido
})

export default i18n
```

Se importa una vez en el punto de entrada (`import './i18n'` en `main.tsx`). En los componentes se usa el hook:

```tsx
import { useTranslation } from 'react-i18next'

function LoginButton() {
  const { t } = useTranslation()
  return <button>{t('auth.submit')}</button>
}
```

Devuelve «Entrar» o «Entra» según el idioma activo, y el componente se vuelve a renderizar cuando este cambia.

## Textos con variables y plurales

Los textos con datos dinámicos llevan **marcadores** en el diccionario y los valores se pasan como segundo argumento:

```ts
// es.ts
cart: {
  greeting: 'Hola, {{name}}',
  items_one: '{{count}} artículo',
  items_other: '{{count}} artículos',
}
```

```tsx
t('cart.greeting', { name: user.name })  // "Hola, Ana"
t('cart.items', { count: 3 })            // "3 artículos"
```

Con la variable `count`, i18next elige la forma plural (`_one`, `_other`...) que corresponde al idioma.

**Nunca concatenes frases** (`t('hello') + ' ' + name + '!'`): el orden de las palabras cambia entre idiomas y la concatenación lo rompe. Una sola clave por frase completa, con el marcador dentro.

## Cambiar de idioma y recordarlo

El cambio se hace con `i18n.changeLanguage`, y la preferencia se guarda en `localStorage` para que sobreviva a recargar la página:

```ts
const STORAGE_KEY = 'app-language'
const LANGUAGES = ['es', 'ca']

function initialLanguage() {
  const saved = localStorage.getItem(STORAGE_KEY)
  return saved && LANGUAGES.includes(saved) ? saved : 'es'
}

// en init: lng: initialLanguage()
i18n.on('languageChanged', (lng) => localStorage.setItem(STORAGE_KEY, lng))
```

```tsx
<select value={i18n.language} onChange={(e) => i18n.changeLanguage(e.target.value)}>
  <option value="es">Español</option>
  <option value="ca">Català</option>
</select>
```

Se valida el valor leído porque `localStorage` puede contener cualquier cosa (un idioma que ya no existe, un valor manipulado).

## Qué pasa cuando falta una clave

Si la clave no existe en el idioma activo, i18next prueba con `fallbackLng`; si tampoco está ahí, **devuelve la propia clave** (`'auth.submit'`). Es una señal útil: un texto como `auth.submit` en pantalla es feo pero se detecta al instante, mientras que un texto vacío pasaría desapercibido. Con los diccionarios tipados como en el ejemplo, además, el caso se detecta antes de ejecutar nada.

## Tests

En los tests no hace falta un idioma real: lo habitual es sustituir `t` por una función que devuelve la clave cruda, así las aserciones no dependen de la redacción de los textos:

```tsx
// vitest
vi.mock('react-i18next', () => ({
  useTranslation: () => ({ t: (key: string) => key }),
}))

it('muestra el botón de envío', () => {
  render(<LoginButton />)
  expect(screen.getByRole('button', { name: 'auth.submit' })).toBeInTheDocument()
})
```

Si el test necesita comprobar el texto real, se inicializa i18next con el diccionario de un idioma concreto en lugar de mockear.

## Buenas prácticas avanzadas

- **Tipar las claves** — i18next permite declarar los recursos en `CustomTypeOptions` para que `t('auth.subit')` falle al compilar y el editor autocomplete las claves. Es la mejora con mejor relación esfuerzo/beneficio.
- **Un idioma como fuente de verdad** — define el tipo del diccionario a partir de uno (`typeof es`) y obliga al resto a ajustarse; evita que los diccionarios diverjan en silencio.
- **No muestres mensajes de error del servidor tal cual** — mapea el código o estado del error a una clave de la interfaz. Así el idioma del mensaje depende de la UI y no de lo que devuelva el backend.
- **No guardes texto traducido en estado o constantes de módulo** — se evalúa una vez y no reacciona al cambiar de idioma. Guarda la clave y llama a `t` al renderizar.
- **Formatea fechas, números y monedas con `Intl`** (`Intl.DateTimeFormat`, `Intl.NumberFormat`) usando el idioma activo; no los metas en los diccionarios.
- **Cuida el espacio** — un texto en catalán o alemán puede ocupar bastante más que en inglés; diseña botones y menús sin anchos fijos.

## Documentación oficial

- [i18next](https://www.i18next.com/) — el núcleo: interpolación, plurales, `fallbackLng` y tipado con TypeScript. Empieza por la sección «Essentials».
- [react-i18next](https://react.i18next.com/) — el binding para React: `useTranslation`, `Trans` (para textos con etiquetas dentro) y su guía de inicio.
- [MDN: `Intl`](https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/Intl) — formato de fechas, números y monedas según el idioma, sin dependencias.

---

*En resumen: los componentes piden textos por clave, un diccionario por idioma los resuelve, y el compilador se encarga de que todos los diccionarios tengan las mismas claves.*
