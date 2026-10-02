# Scroll-spy con IntersectionObserver

## ¿Qué es?

Un *scroll-spy* es un índice de navegación que se **resalta solo** según la sección que se está viendo mientras se hace scroll. `IntersectionObserver` es la API del navegador que avisa cuando un elemento entra o sale de la zona visible, y es la herramienta moderna para construirlo.

## ¿Por qué existe?

Una página larga de documentación, un manual o una ficha de producto con varias secciones suele llevar un índice lateral con enlaces a cada sección. Que el índice indique «aquí estás» exige saber qué sección ocupa la pantalla en cada momento.

La solución clásica era escuchar el evento `scroll` y, en cada disparo, medir con `getBoundingClientRect()` la posición de todas las secciones. Funciona, pero `scroll` se dispara decenas de veces por segundo, se ejecuta en el hilo principal y cada medición puede forzar al navegador a recalcular el layout. Es el equivalente a consultar una tabla en bucle (*polling*) para saber si algo ha cambiado.

`IntersectionObserver` invierte el modelo: se le dice qué elementos vigilar y el navegador **avisa solo cuando cambia su visibilidad**, sin que el código tenga que medir nada.

> Si ya conoces los *triggers* de una base de datos, piensa en el observer como uno: en vez de preguntar continuamente «¿ha cambiado algo?», registras el interés una vez y te notifican.

## ¿Cuándo y para qué se usa?

- Índice lateral de una página de documentación o de un manual de usuario.
- Barra de pestañas de una página de producto de una tienda online («Descripción», «Opiniones», «Envíos») que marca la sección en pantalla.
- Cabecera que cambia de estilo al dejar de verse un bloque concreto.

Para otros usos del mismo observer (carga diferida de imágenes, scroll infinito), la mecánica es idéntica; solo cambia lo que se hace en el callback.

---

## Cómo funciona `IntersectionObserver`

Se crea con un *callback* y unas opciones, y después se le pide observar elementos:

```ts
const observer = new IntersectionObserver(
  (entries) => {
    for (const entry of entries) {
      console.log(entry.target.id, entry.isIntersecting, entry.intersectionRatio)
    }
  },
  { threshold: [0, 0.5, 1] },
)

observer.observe(document.getElementById('envios')!)
// cuando ya no haga falta:
observer.disconnect()
```

Cada `entry` describe un cambio e incluye tres datos útiles:

| Propiedad | Significado |
|---|---|
| `target` | El elemento observado (de ahí se saca su `id`). |
| `isIntersecting` | `true` si ahora se solapa con la zona de observación. |
| `intersectionRatio` | Qué fracción del elemento es visible, de 0 a 1. |

Dos detalles que sorprenden la primera vez:

- **El callback solo recibe los elementos que han cambiado**, no todos. Para saber el estado de conjunto hay que guardarlo aparte (un `Map` de id → ratio, como se verá abajo).
- **Al llamar a `observe()` el callback se ejecuta una vez** con el estado inicial de cada elemento. Esto es la raíz de uno de los errores frecuentes de más abajo.

### Las tres opciones

| Opción | Qué controla | Valor por defecto |
|---|---|---|
| `root` | El elemento respecto al que se mide la visibilidad. | `null`: el viewport del navegador. |
| `rootMargin` | Margen que **agranda o encoge** la zona de observación, con sintaxis de margin CSS (`"-20% 0px"`). | `"0px"` |
| `threshold` | Umbrales de ratio que disparan el callback: un número o un array. | `0`: basta con que se vea un píxel. |

Con `threshold: [0, 0.25, 0.5, 0.75, 1]` el callback se dispara cada vez que el ratio cruza uno de esos cinco valores: da una resolución suficiente para comparar secciones sin disparar el callback en cada píxel. Si solo importa «entra o sale», basta `0`.

`root` solo hace falta si el scroll ocurre dentro de un contenedor con `overflow: auto` en lugar de en la página entera. `rootMargin` se usa sobre todo para definir una **banda de activación**; se retoma en la sección de errores frecuentes.

## El patrón: función pura + hook

Hay un truco que simplifica mucho el testeo: separar **la decisión** («de las secciones visibles, ¿cuál cuenta como activa?») del **pegamento** con el navegador.

Primero, la decisión como función pura, que no sabe nada de observers ni de DOM:

```ts
export function elegirSeccionActiva(
  entradas: { id: string; ratio: number }[],
): string | null {
  if (entradas.length === 0) return null
  return entradas.reduce((mejor, actual) => (actual.ratio > mejor.ratio ? actual : mejor)).id
}
```

El criterio aquí es «gana la que más se ve». Cambiar el criterio (la primera visible, la más cercana al borde superior...) solo toca esta función.

Después, el hook, que mantiene un `Map` con el último ratio conocido de cada sección y delega la elección:

```ts
import { useEffect, useRef, useState } from 'react'

export function useSeccionActiva(ids: string[]): string | null {
  const [activa, setActiva] = useState<string | null>(ids[0] ?? null)

  const idsRef = useRef(ids)
  idsRef.current = ids
  const idsKey = ids.join(',')

  useEffect(() => {
    const visibles = new Map<string, number>()
    const observer = new IntersectionObserver(
      (entries) => {
        for (const entry of entries) {
          visibles.set(entry.target.id, entry.isIntersecting ? entry.intersectionRatio : 0)
        }
        const entradas = [...visibles.entries()].map(([id, ratio]) => ({ id, ratio }))
        const elegida = elegirSeccionActiva(entradas)
        if (elegida) setActiva(elegida)
      },
      { threshold: [0, 0.25, 0.5, 0.75, 1] },
    )

    for (const id of idsRef.current) {
      const el = document.getElementById(id)
      if (el) observer.observe(el)
    }

    return () => observer.disconnect()
  }, [idsKey])

  return activa
}
```

Qué hace, paso a paso:

1. Al montar, crea el observer y observa cada elemento cuyo `id` se le pasó.
2. En cada aviso actualiza el `Map` con los elementos que cambiaron y pasa **todo el mapa** a la función pura.
3. Devuelve el `id` activo, que el componente usa para resaltar el enlace.
4. Al desmontar, `disconnect()` libera el observer. Sin esa limpieza, el observer sigue vivo tras desmontar el componente.

La razón de `idsRef` e `idsKey` se explica en «Errores frecuentes»: es el arreglo de un bug real.

## Anclas e índice

El índice es una lista de enlaces a `#id`; cada sección del contenido lleva el `id` correspondiente. El navegador hace el salto sin una línea de JavaScript:

```tsx
function IndiceManual({ secciones }: { secciones: { id: string; titulo: string }[] }) {
  const activa = useSeccionActiva(secciones.map((s) => s.id))

  return (
    <nav aria-label="Índice de la página">
      {secciones.map((s) => (
        <a
          key={s.id}
          href={`#${s.id}`}
          aria-current={activa === s.id ? 'location' : undefined}
          className={activa === s.id ? 'font-semibold' : 'text-muted'}
        >
          {s.titulo}
        </a>
      ))}
    </nav>
  )
}

// y en el contenido:
// <section id="envios"><h2>Envíos</h2>...</section>
```

- `href="#envios"` mantiene la navegación nativa: funciona con teclado, con clic central y se puede copiar el enlace.
- `aria-current="location"` (valor definido por ARIA para «la ubicación actual dentro de un conjunto») comunica la sección activa también a lectores de pantalla, no solo con color.
- `aria-label` en el `<nav>` lo distingue de otros `<nav>` de la página.

## Índice fijo con `position: sticky`

Un índice largo debe acompañar al lector mientras recorre el contenido. `position: sticky; top: <valor>` lo deja pegado al borde superior al hacer scroll. Pero `sticky` falla en silencio si no se cumplen sus condiciones, y es la fuente habitual de «no funciona y no sé por qué»:

```tsx
<div className="grid grid-cols-[14rem_1fr] gap-8">
  <div>                                  {/* envoltorio sin estilos propios */}
    <nav className="sticky top-6 max-h-[calc(100vh-3rem)] overflow-y-auto">
      {/* enlaces */}
    </nav>
  </div>
  <main>{/* contenido muy largo */}</main>
</div>
```

Las trampas, una a una:

| Trampa | Por qué rompe `sticky` | Arreglo |
|---|---|---|
| **El elemento sticky mide lo mismo que su contenedor** | Un elemento sticky solo se mueve dentro de su contenedor (su *bloque contenedor*). Si es hijo directo de un grid, `align-items: stretch` (el valor por defecto) lo estira a la altura de la fila y no le queda recorrido: no hay a dónde «desplazarse». | O bien `align-self: start` en el elemento sticky (deja de estirarse), o bien, como en el ejemplo, un envoltorio sin estilos que sí se estira a la altura de la fila y dentro de él el `nav` sticky, que tiene así todo el recorrido. |
| **Falta `top`** | Sin `top` (o `bottom`, `left`, `right`) no hay umbral al que pegarse. | Siempre declarar `top`. |
| **Un ancestro con `overflow` distinto de `visible`** | `overflow: hidden`, `auto` o `scroll` en un ancestro convierte a ese ancestro en el contenedor de scroll; `sticky` se pega a él, no al viewport. | Quitar el `overflow` del ancestro o aceptar que el índice se pega a ese contenedor. |
| **Índice más alto que la pantalla** | Los últimos enlaces quedan inalcanzables. | `max-height: calc(100vh - ...)` más `overflow-y: auto` en el propio `nav`. |

La regla práctica para depurar: inspeccionar los ancestros uno a uno en las herramientas de desarrollo buscando `overflow` y comprobando la altura del contenedor.

## Probarlo

jsdom, el DOM simulado que usa la mayoría de los entornos de test (ver [jsdom](../frontend-react/jsdom.md)), **no implementa `IntersectionObserver`**: cualquier test que monte el componente falla con `IntersectionObserver is not defined`, aunque no verifique nada de scroll. Hay dos frentes:

**1. Stub global**, en el fichero de preparación de los tests (por ejemplo `vitest.setup.ts`), para que los componentes puedan montarse:

```ts
if (!('IntersectionObserver' in globalThis)) {
  globalThis.IntersectionObserver = class {
    observe() {}
    unobserve() {}
    disconnect() {}
  } as unknown as typeof IntersectionObserver
}
```

El stub se pone en el setup de tests y no en el componente, para no meter andamiaje de test en el código que se despliega.

**2. Test de la función pura**, que es donde está la lógica y no necesita ningún mock:

```ts
it('elige la entrada con mayor ratio de intersección', () => {
  const resultado = elegirSeccionActiva([
    { id: 'descripcion', ratio: 0.2 },
    { id: 'opiniones', ratio: 0.8 },
  ])
  expect(resultado).toBe('opiniones')
})

it('sin ninguna entrada visible, devuelve null', () => {
  expect(elegirSeccionActiva([])).toBeNull()
})
```

Si se necesita probar el hook entero (que el resaltado cambia al «llegar» una entrada), el stub puede guardar el callback recibido en el constructor y el test lo invoca a mano con entradas falsas (`{ target: { id: 'opiniones' }, isIntersecting: true, intersectionRatio: 0.9 }`). Es más trabajo y rinde menos que probar la función pura, así que se reserva para cuando hay lógica en el pegamento. La verificación del comportamiento real en un navegador sigue haciendo falta: el stub no dispara nunca el callback, y por eso no detecta el error de la siguiente sección.

## Errores frecuentes

### Referencias inestables en las dependencias del efecto

Es el fallo más sutil. Si el efecto depende directamente del array de ids:

```tsx
// ❌ MAL: `secciones.map(...)` crea un array nuevo en cada render
const activa = useSeccionActiva(secciones.map((s) => s.id))

useEffect(() => { /* crear observer */ }, [ids])   // cambia en CADA render
```

Esto ocurre: el componente se renderiza → el array es nuevo → el efecto se vuelve a ejecutar → se crea un observer nuevo → como algún elemento ya está visible, su callback inicial llama a `setActiva` → nuevo render → array nuevo... **Bucle infinito.** Y los tests con el stub no lo ven, porque el stub no dispara el callback.

El arreglo es que el efecto dependa del **contenido** y no de la referencia:

```tsx
// ✅ BIEN
const idsRef = useRef(ids)
idsRef.current = ids          // el efecto lee siempre la última versión...
const idsKey = ids.join(',')  // ...pero solo se reengancha si cambia el contenido

useEffect(() => { /* usa idsRef.current */ }, [idsKey])
```

La alternativa es exigir a quien llama que memorice el array con `useMemo`, pero el hook queda a merced de un descuido ajeno; arreglarlo dentro del hook es más robusto.

### Secciones más altas que el viewport

Con el criterio «gana la que tiene más ratio», una sección más alta que la pantalla **nunca llega a ratio 1** y su ratio puede ser pequeño aunque ocupe toda la pantalla. Una sección corta vecina, completamente visible, ganaría con ratio 1 y el índice resaltaría la equivocada. Dos salidas:

- Usar una **banda de activación** con `rootMargin`: encoger la zona de observación a una franja horizontal delgada, por ejemplo `rootMargin: '-40% 0px -55% 0px'` con `threshold: 0`. Una sección está activa cuando cruza esa franja, sea del tamaño que sea, y en cada momento cruza una sola.
- Elegir por posición (la última sección cuyo borde superior ya ha pasado cierta línea) en vez de por ratio. Es la misma función pura con otra regla.

### Scroll suave y cabeceras fijas

Al pulsar un enlace del índice, dos detalles de CSS que la guía del observer no cubre:

- Con una **cabecera fija** (`position: fixed`/`sticky`) la sección destino queda tapada bajo ella. Se arregla con `scroll-margin-top` en cada sección, del alto de la cabecera.
- `scroll-behavior: smooth` en el `html` anima el salto, pero conviene respetar a quien ha pedido menos animación:

```css
html { scroll-behavior: smooth; }
section[id] { scroll-margin-top: 4rem; }

@media (prefers-reduced-motion: reduce) {
  html { scroll-behavior: auto; }
}
```

Durante un salto animado, el observer va marcando como activas todas las secciones intermedias. Si el efecto de «parpadeo» molesta, se puede fijar el estado activo en el clic y suspender el observer hasta que termine el scroll; es un refinamiento que solo merece la pena si se nota.

### Elementos que aún no existen

`document.getElementById` devuelve `null` si la sección todavía no está en el DOM (contenido que se carga después). El hook del ejemplo ignora los que no encuentra; si el contenido llega tarde, el efecto debe reejecutarse cuando aparezca, por ejemplo incluyendo en `idsKey` la lista de secciones ya cargadas.

## Buenas prácticas avanzadas

- **Separa la decisión del observer.** Una función pura que recibe `{id, ratio}[]` se prueba en una línea y sin mocks; el hook queda como pegamento fino que apenas necesita test. Es el patrón que más complejidad quita.
- **Guarda el estado de conjunto tú mismo.** El callback solo trae los elementos que cambiaron. Si se decide con `entries` directamente, el índice «olvida» las secciones que no cambiaron en este aviso y la elección es incorrecta.
- **Haz depender el efecto del contenido, no de la referencia.** Un array o un objeto creado en el render es una dependencia distinta cada vez. Usa una clave serializada (`ids.join(',')`) y un ref para leer el valor fresco.
- **Define una banda de activación con `rootMargin` si hay secciones desiguales.** Con ratios, las muy altas o muy bajas distorsionan la elección; una franja fija de observación se comporta igual para todas.
- **Limpia siempre con `disconnect()`.** Un observer no desconectado mantiene referencias a elementos del DOM y sigue llamando a un callback que actualiza estado de un componente ya desmontado.
- **No dependas solo del color.** Añade `aria-current` al enlace activo: el resaltado visual no llega a quien usa lector de pantalla.

## Documentación oficial

- [Intersection Observer API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) — visión general con el modelo de «raíz, umbrales y márgenes» y diagramas; el mejor punto de partida para entender `rootMargin`.
- [Constructor `IntersectionObserver` (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver/IntersectionObserver) — referencia exacta de `root`, `rootMargin` y `threshold`.
- [`position: sticky` en la página de `position` (MDN)](https://developer.mozilla.org/en-US/docs/Web/CSS/position) — la sección sobre `sticky` describe el bloque contenedor y el papel del ancestro con scroll.
- [`scroll-margin-top` (MDN)](https://developer.mozilla.org/en-US/docs/Web/CSS/scroll-margin-top) — para que las anclas no queden tapadas por una cabecera fija.

---

*En resumen: un scroll-spy es un `IntersectionObserver` que alimenta una función pura que elige la sección activa; hazlo depender de ids por contenido, define bien la zona de activación y recuerda que un `sticky` solo funciona si su contenedor es más alto que él.*
