# Modal con zoom y arrastre

## ¿Qué es?

Un modal que muestra una imagen ampliada y permite **acercarla con la rueda del ratón, moverla arrastrando y reiniciarla** con doble clic o con botones. Es el visor que se abre al pulsar una captura de una página de documentación o la foto de un producto en una tienda online.

## ¿Por qué existe?

Una captura de pantalla incrustada en un texto suele ser demasiado pequeña para leer los detalles, y obligar a abrirla en otra pestaña rompe el hilo. Un modal con zoom resuelve eso sin salir de la página, pero tiene dos partes que, hechas a medias, dan un resultado frustrante:

- **El modal**: si no gestiona el foco, la tecla Esc y los lectores de pantalla, solo funciona con ratón.
- **El zoom**: si acerca hacia el centro en vez de hacia el cursor, hay que acercar, arrastrar, acercar, arrastrar... para llegar al detalle. El zoom cómodo mantiene **bajo el cursor el mismo punto de la imagen**, como en los mapas.

Esta guía construye ambas piezas con React, sin librerías de gestos.

## Piezas del visor

| Pieza | Responsabilidad |
|---|---|
| `Dialog` accesible | Abrir y cerrar, atrapar el foco, Esc, `aria-modal`. |
| Hook `useZoomPan` | Estado `{ escala, x, y }` y las operaciones sobre él. Sin DOM. |
| Componente visor | Traduce eventos (rueda, puntero) a llamadas al hook y aplica la transformación CSS. |
| Botones | Alternativa de teclado y táctil a la rueda y al arrastre. |

---

## El modal accesible

Construir un modal desde cero es más difícil de lo que parece (foco, Esc, `aria-*`, portal), así que se parte de uno ya hecho: el `Dialog` de [Radix UI](../frontend-react/RadixUI.md), que es el que usa [shadcn/ui](../frontend-react/shadcn-ui.md) por debajo.

```tsx
import { useState } from 'react'
import { Dialog, DialogContent, DialogTitle } from '@/components/ui/dialog'

function Galeria({ fotos }: { fotos: { src: string; alt: string }[] }) {
  const [ampliada, setAmpliada] = useState<{ src: string; alt: string } | null>(null)

  return (
    <>
      {fotos.map((foto) => (
        <button key={foto.src} type="button" onClick={() => setAmpliada(foto)}>
          <img src={foto.src} alt={foto.alt} className="cursor-zoom-in" />
        </button>
      ))}

      <Dialog open={ampliada != null} onOpenChange={(abierto) => !abierto && setAmpliada(null)}>
        <DialogContent className="max-w-[95vw] border-0 bg-transparent p-2 shadow-none">
          <DialogTitle className="sr-only">{ampliada?.alt}</DialogTitle>
          {ampliada && <ImagenAmpliada src={ampliada.src} alt={ampliada.alt} />}
        </DialogContent>
      </Dialog>
    </>
  )
}
```

Qué resuelve el `Dialog` sin más código:

- **Foco**: al abrir, el foco entra en el modal y queda atrapado; al cerrar, vuelve al elemento que lo abrió (si lo abrió un botón, como arriba).
- **Esc** y clic en el fondo cierran el modal (llaman a `onOpenChange(false)`).
- **Lectores de pantalla**: el contenido de detrás se marca como inerte y el modal se anuncia como diálogo.

El detalle que se olvida es el **título**: un diálogo accesible necesita un nombre. Como el visor ya muestra una imagen y un título visible sobraría, se usa `DialogTitle` con la clase `sr-only` (visible solo para lectores de pantalla) y se reutiliza el `alt` de la imagen. Radix avisa por consola si falta el título.

El estado se guarda en el componente padre (qué imagen está ampliada o `null`), y `ImagenAmpliada` solo se monta mientras el modal está abierto. Eso da un reinicio gratis: cada apertura crea un visor nuevo con zoom 1.

## El estado: `{ escala, x, y }`

Toda la vista cabe en tres números:

- `escala`: 1 es el tamaño inicial, 2 es el doble...
- `x`, `y`: desplazamiento en píxeles de la imagen respecto a su posición inicial.

```ts
import { useCallback, useMemo, useState } from 'react'

export const ZOOM_MIN = 1
export const ZOOM_MAX = 8

interface Vista { escala: number; x: number; y: number }
const VISTA_INICIAL: Vista = { escala: 1, x: 0, y: 0 }

const acotar = (escala: number) => Math.min(ZOOM_MAX, Math.max(ZOOM_MIN, escala))
```

Los límites van en constantes exportadas. `ZOOM_MIN = 1` impide alejarse más que el ajuste inicial; `ZOOM_MAX = 8` evita ampliar hasta un borrón de píxeles. `acotar` es la única puerta por la que pasa cualquier escala nueva.

El hook ofrece tres operaciones y no sabe nada de ratón ni de DOM:

```ts
export function useZoomPan() {
  const [vista, setVista] = useState<Vista>(VISTA_INICIAL)

  const alZoom = useCallback((factor: number, cx = 0, cy = 0) => { /* siguiente sección */ }, [])

  const alArrastrar = useCallback((dx: number, dy: number) => {
    setVista((previa) =>
      previa.escala === ZOOM_MIN ? previa : { ...previa, x: previa.x + dx, y: previa.y + dy },
    )
  }, [])

  const reiniciar = useCallback(() => setVista(VISTA_INICIAL), [])

  return useMemo(
    () => ({ ...vista, alZoom, alArrastrar, reiniciar }),
    [vista, alZoom, alArrastrar, reiniciar],
  )
}
```

- `alArrastrar` suma el movimiento al desplazamiento, y **no hace nada con escala 1**: arrastrar una imagen que cabe entera solo la descolocaría.
- Se usa la forma funcional de `setVista((previa) => ...)` para que varias llamadas seguidas (una ráfaga de eventos de rueda) no pisen el estado anterior.
- Las funciones van con `useCallback` y el objeto devuelto con `useMemo`, para que sean estables y no provoquen renders en quien las reciba.

## Zoom anclado al cursor: la fórmula

Este es el núcleo. Hay que fijar antes dos convenciones:

1. La imagen se transforma con CSS desde su **centro** (`transform-origin: center`, que es el valor por defecto).
2. Las coordenadas del cursor se expresan **respecto al centro** de la imagen: `(0, 0)` es el centro, no la esquina.

La transformación `translate(x, y) scale(s)` coloca el punto de la imagen que está a distancia `p` del centro (medida en la imagen original) en la posición `x + s·p` respecto al centro: primero se escala y después se traslada.

**Objetivo**: al cambiar la escala de `s` a `s'`, el punto de la imagen que está bajo el cursor (el ancla `a`) debe seguir bajo el cursor.

Paso a paso, para el eje horizontal (el vertical es idéntico):

1. Antes del zoom, el punto bajo el ancla cumple `a = x + s·p`. Despejando, el punto de la imagen es `p = (a − x) / s`.
2. Después del zoom, queremos `a = x' + s'·p` con **el mismo `p`**.
3. Despejando el desplazamiento nuevo: `x' = a − s'·p`.
4. Sustituyendo `p` del paso 1: `x' = a − (a − x)·s'/s`.

Esa es la fórmula: `desplazamiento' = ancla − (ancla − desplazamiento)·escala'/escala`. En código:

```ts
const alZoom = useCallback((factor: number, cx = 0, cy = 0) => {
  setVista((previa) => {
    const escala = acotar(previa.escala * factor)
    if (escala === ZOOM_MIN) return VISTA_INICIAL          // al volver al inicio, se recentra
    const ratio = escala / previa.escala
    return {
      escala,
      x: cx - (cx - previa.x) * ratio,
      y: cy - (cy - previa.y) * ratio,
    }
  })
}, [])
```

Observaciones:

- El zoom es **multiplicativo** (`factor`): 1,15 acerca un 15 % y su inverso `1/1,15` aleja lo mismo. Con pasos aditivos, el zoom se sentiría rápido de cerca y lento de lejos.
- `ratio` se calcula con la escala **ya acotada**. Si la nueva escala se ha topado con el límite máximo, el ratio refleja el cambio real, no el pedido, y el ancla sigue fija.
- Sin ancla (`cx = cy = 0`) el zoom es sobre el centro, que es lo que se quiere con los botones.
- Al llegar a `ZOOM_MIN` se devuelve la vista inicial: si la imagen vuelve a su tamaño original, también vuelve a su posición.

Comprobación numérica: escala 1→2 con el cursor a 100 px a la derecha del centro y `x = 0`. `x' = 100 − (100 − 0)·2 = −100`. La imagen se desplaza 100 px a la izquierda; el punto que estaba a 100 px del centro se duplica de distancia (200 px), menos los 100 px desplazados, y queda otra vez a 100 px: bajo el cursor.

## El componente visor

Aquí se traducen los eventos del navegador a las operaciones del hook.

### Rueda del ratón

```tsx
const FACTOR_RUEDA = 1.15

function alRodar(e: React.WheelEvent<HTMLDivElement>) {
  const caja = e.currentTarget.getBoundingClientRect()
  const cx = e.clientX - (caja.left + caja.width / 2)
  const cy = e.clientY - (caja.top + caja.height / 2)
  vista.alZoom(e.deltaY > 0 ? 1 / FACTOR_RUEDA : FACTOR_RUEDA, cx, cy)
}
```

`getBoundingClientRect()` da la caja del contenedor en coordenadas de pantalla; restando su centro al `clientX/Y` del evento se obtiene el ancla respecto al centro, tal como pide la fórmula. `deltaY > 0` significa rueda hacia abajo, que se interpreta como alejar.

Se mide el **contenedor** (`currentTarget`, que no se transforma) y no la imagen, porque `getBoundingClientRect()` de la imagen ya incluiría `scale` y `translate`.

### Arrastre con Pointer Events

Los *Pointer Events* unifican ratón, táctil y lápiz con una sola familia de eventos (`pointerdown`, `pointermove`, `pointerup`, `pointercancel`), así que no hay que programar el arrastre dos veces. Cada puntero tiene un `pointerId`, y con `setPointerCapture` el elemento sigue recibiendo los eventos de ese puntero **aunque el cursor salga de él**: sin captura, soltar el botón fuera del visor dejaría el arrastre «enganchado».

```tsx
const arrastre = useRef<{ pointerId: number; x: number; y: number } | null>(null)

function alBajarPuntero(e: React.PointerEvent<HTMLDivElement>) {
  arrastre.current = { pointerId: e.pointerId, x: e.clientX, y: e.clientY }
  e.currentTarget.setPointerCapture(e.pointerId)
}

function alMoverPuntero(e: React.PointerEvent<HTMLDivElement>) {
  const actual = arrastre.current
  if (actual?.pointerId !== e.pointerId) return
  vista.alArrastrar(e.clientX - actual.x, e.clientY - actual.y)
  arrastre.current = { pointerId: e.pointerId, x: e.clientX, y: e.clientY }
}

function alSoltarPuntero(e: React.PointerEvent<HTMLDivElement>) {
  if (arrastre.current?.pointerId !== e.pointerId) return
  arrastre.current = null
  e.currentTarget.releasePointerCapture(e.pointerId)
}
```

- El arrastre en curso vive en un `useRef` y no en estado: cambia en cada movimiento y no necesita volver a pintar nada por sí mismo (lo que se pinta es la vista, que sí es estado).
- En cada movimiento se calcula el **delta respecto a la posición anterior** y se guarda la nueva. Así cada llamada a `alArrastrar` es un incremento pequeño y no importa cuánto se haya arrastrado en total.
- Comprobar el `pointerId` evita que un segundo dedo o puntero interfiera.
- Se gestiona también `pointercancel` (el sistema interrumpe el gesto), con el mismo manejador que `pointerup`.

Usar el delta de `clientX/clientY` es preferible a `movementX/movementY`: este último es un valor calculado por el navegador que no está presente en todos los entornos (jsdom, por ejemplo) y es menos predecible.

### Aplicar la transformación

```tsx
<div
  className="touch-none overflow-hidden"
  onWheel={alRodar}
  onPointerDown={alBajarPuntero}
  onPointerMove={alMoverPuntero}
  onPointerUp={alSoltarPuntero}
  onPointerCancel={alSoltarPuntero}
  onDoubleClick={vista.reiniciar}
>
  <img
    src={src}
    alt={alt}
    draggable={false}
    className="max-h-[80vh] w-auto select-none"
    style={{ transform: `translate(${vista.x}px, ${vista.y}px) scale(${vista.escala})` }}
  />
</div>
```

- `overflow-hidden` en el contenedor recorta la parte de la imagen que se sale al ampliar.
- `touch-action: none` (`touch-none` en Tailwind) impide que el navegador táctil interprete el gesto como scroll de la página o pellizco: sin ello, los `pointermove` se cancelan a los pocos píxeles.
- `draggable={false}` y `select-none` evitan que el navegador inicie su propio «arrastrar imagen» y robe el gesto.
- El orden en `transform` importa: `translate(...) scale(...)` aplica primero la escala y después el desplazamiento, que es lo que asume la fórmula. Invertirlo (`scale` y luego `translate`) escalaría también los píxeles del desplazamiento y rompería el anclaje.
- `onDoubleClick` reinicia la vista.
- Usar `transform` es más fluido que cambiar `width`/`left`/`top`: el navegador lo resuelve en la GPU sin recalcular el layout.

## Controles accesibles

La rueda y el arrastre no sirven a quien navega con teclado, a quien usa un dispositivo sin rueda ni a quien necesita ayuda motriz. Los botones reutilizan las mismas operaciones del hook:

```tsx
const FACTOR_BOTON = 1.25

<div>
  <button type="button" onClick={() => vista.alZoom(1 / FACTOR_BOTON)} aria-label="Alejar">−</button>
  <span aria-live="polite">{Math.round(vista.escala * 100)} %</span>
  <button type="button" onClick={() => vista.alZoom(FACTOR_BOTON)} aria-label="Acercar">+</button>
  <button type="button" onClick={vista.reiniciar}>Reiniciar</button>
</div>
```

- Botones reales (`<button type="button">`) y no `div` con `onClick`: reciben foco y se activan con Intro y Espacio.
- Los botones que solo llevan un símbolo (`−`, `+`) necesitan `aria-label`.
- `aria-live="polite"` en el porcentaje anuncia el nivel de zoom a los lectores de pantalla sin interrumpir.
- Sin ancla, los botones hacen zoom sobre el centro; es lo esperable.
- Una línea de ayuda visible («Rueda: zoom · Arrastra: mover · Doble clic: reiniciar») descubre los gestos.

## La rueda y el scroll de la página

Una rueda sobre un visor puede hacer dos cosas: ampliar la imagen y desplazar la página. Hay dos casos:

- **Dentro de un modal**, la página de detrás normalmente ya no se desplaza: Radix bloquea el scroll del fondo mientras el diálogo está abierto. No hace falta nada más.
- **En un visor incrustado en la página** (sin modal), hay que impedir que la rueda la desplace. Y aquí aparece la trampa: React registra el `onWheel` como listener **pasivo**, y en un listener pasivo `e.preventDefault()` se ignora y la consola avisa. La solución es registrar el listener a mano con `passive: false`:

```tsx
useEffect(() => {
  const el = contenedorRef.current
  if (!el) return
  const alRodarSinScroll = (e: WheelEvent) => {
    e.preventDefault()
    // ...misma lógica de zoom
  }
  el.addEventListener('wheel', alRodarSinScroll, { passive: false })
  return () => el.removeEventListener('wheel', alRodarSinScroll)
}, [])
```

Si el visor es un contenedor con su propio scroll, `overscroll-behavior: contain` evita que el scroll «se encadene» a la página al llegar al final, y a menudo basta sin necesidad de `preventDefault`.

## Probarlo

El hook `useZoomPan` no toca el DOM, así que se prueba como lógica pura con `renderHook` de [Testing Library](../frontend-react/testing-library-react.md):

```ts
it('el punto bajo el ancla se mantiene al hacer zoom', () => {
  const { result } = renderHook(() => useZoomPan())

  act(() => result.current.alZoom(2, 100, 0))

  expect(result.current.escala).toBe(2)
  expect(result.current.x).toBe(-100)   // 100 − (100 − 0)·2
})

it('no supera el zoom máximo', () => {
  const { result } = renderHook(() => useZoomPan())
  act(() => result.current.alZoom(1000))
  expect(result.current.escala).toBe(ZOOM_MAX)
})

it('arrastrar sin zoom no mueve la imagen', () => {
  const { result } = renderHook(() => useZoomPan())
  act(() => result.current.alArrastrar(30, 30))
  expect(result.current.x).toBe(0)
})
```

Para el componente, las limitaciones de [jsdom](../frontend-react/jsdom.md) condicionan los tests:

| Limitación | Efecto | Solución |
|---|---|---|
| `setPointerCapture` / `releasePointerCapture` no están implementados en los elementos | `TypeError` al simular `pointerdown`. | Stubearlos en el setup: `HTMLElement.prototype.setPointerCapture = vi.fn()` (y `releasePointerCapture`). |
| `movementX/Y` no es fiable en los eventos simulados | Un arrastre que dependa de él no se mueve en el test. | Calcular el delta con `clientX/clientY` (como arriba) y pasarlos en el evento: `fireEvent.pointerMove(visor, { clientX: 130, clientY: 80, pointerId: 1 })`. |
| `getBoundingClientRect()` devuelve todo ceros | El ancla de la rueda sale mal. | Stubearlo en el elemento con un rectángulo conocido, o probar la fórmula directamente en el hook, que es lo que ya cubre el test de arriba. |

Un test de componente razonable comprueba lo observable: que al disparar `wheel` el `style.transform` de la imagen cambia, que el doble clic lo reinicia y que los botones cambian el porcentaje. El anclaje exacto y el arrastre real se verifican a mano en un navegador, donde sí hay geometría real.

## Buenas prácticas avanzadas

- **Separa el estado de la vista del manejo de eventos.** Un hook de `{ escala, x, y }` sin DOM se prueba sin simular un solo evento y permite reutilizarlo en otros visores (un mapa, un plano). Los eventos solo traducen coordenadas.
- **Deriva el ancla del contenedor, no de la imagen.** La imagen ya está transformada; su `getBoundingClientRect()` mezcla el zoom en la medida y el cálculo se desvía cada vez más. Mide el padre fijo.
- **Zoom multiplicativo con límite aplicado antes del ratio.** Calcular `ratio = nuevaEscala / escalaPrevia` con la escala ya acotada mantiene el anclaje correcto aunque se pida un factor que se salga del rango.
- **Captura el puntero siempre que arrastres.** Sin `setPointerCapture`, soltar fuera del elemento pierde el `pointerup` y el arrastre queda activo. Acompáñalo de `touch-action: none` o los gestos táctiles se cancelan.
- **Todo gesto de ratón necesita equivalente de teclado.** Los botones de acercar, alejar y reiniciar no son un extra: son el único camino para parte de las personas usuarias.
- **Limita también el desplazamiento si el diseño lo pide.** El ejemplo permite arrastrar la imagen fuera del recuadro. Para una galería suele convenir acotar `x` e `y` a `±(ancho·(escala − 1)/2)`, de modo que nunca se pueda perder la imagen de vista. Es un refinamiento cómodo de añadir porque la lógica vive en el hook.

## Documentación oficial

- [Pointer events (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Pointer_events) — visión general de la familia de eventos y del modelo unificado ratón/táctil/lápiz, con la sección sobre captura del puntero.
- [`Element.setPointerCapture()` (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Element/setPointerCapture) — qué garantiza la captura y cuándo se libera.
- [`touch-action` (MDN)](https://developer.mozilla.org/en-US/docs/Web/CSS/touch-action) — por qué `none` es necesario para gestos propios en pantallas táctiles.
- [Dialog (Radix UI Primitives)](https://www.radix-ui.com/primitives/docs/components/dialog) — comportamiento de foco, teclado y accesibilidad del modal, y las partes (`Title`, `Description`, `Content`) que hay que proporcionar.

---

*En resumen: un visor con zoom es un `Dialog` accesible más un estado `{ escala, x, y }`; para que el zoom siga al cursor basta mantener fijo el punto bajo el ancla (`x' = ancla − (ancla − x)·escala'/escala`), arrastrar con Pointer Events capturados y ofrecer botones para quien no use ratón.*
