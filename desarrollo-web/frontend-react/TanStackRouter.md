# TanStack Router

**Versión usada:** 1.170.10 | **Categoría:** Estado y Datos

## ¿Qué es?

Una librería de enrutamiento client-side para React con tipado completo de extremo a extremo. Gestiona qué componente se muestra según la URL del navegador, con TypeScript integrado desde el diseño.

---

## ¿Por qué existe?

En una aplicación de página única (SPA), el navegador no recarga la página al navegar. Alguien tiene que interceptar los cambios de URL y decidir qué renderizar. TanStack Router hace exactamente eso, pero con una ventaja clave respecto a alternativas como React Router: **los parámetros de ruta y query strings son completamente tipados**.

Analogía .NET: imagina que en ASP.NET Core, cuando accedes a `[HttpGet("{id}")]`, el parámetro `id` llegara siempre como `object` y tuvieras que castearlo manualmente. TanStack Router es como tener `[FromRoute] int id` con inferencia automática, pero para el frontend.

---

## ¿Cuándo y para qué se usa?

Aparece en cualquier SPA React con varias vistas: un dashboard, la ficha de detalle de un producto, un listado con filtros, un formulario de registro. TanStack Router define la estructura de navegación completa, y además se encarga de cargar datos antes de renderizar una vista (`loaders`), lo que evita estados de carga inconsistentes.

La configuración central suele vivir en un archivo `routeTree.gen.ts` (generado automáticamente) y en los archivos de ruta bajo `src/routes/`.

---

## Lo mínimo que necesitas saber

**1. Las rutas son archivos**

Cada archivo en `src/routes/` define una ruta. El nombre del archivo determina la URL.

```
src/routes/
  index.tsx          →  /
  products/
    index.tsx        →  /products
    $productId.tsx   →  /products/123  (parámetro dinámico)
```

**2. Parámetros de ruta tipados**

```tsx
// src/routes/products/$productId.tsx
export const Route = createFileRoute('/products/$productId')({
  component: ProductDetail,
})

function ProductDetail() {
  const { productId } = Route.useParams() // productId: string, inferido automáticamente
  return <h1>Producto {productId}</h1>
}
```

**3. Loaders — datos antes de renderizar**

```tsx
export const Route = createFileRoute('/products/$productId')({
  loader: ({ params }) => fetchProduct(params.productId),
  component: ProductDetail,
})

function ProductDetail() {
  const product = Route.useLoaderData() // tipado según lo que retorna fetchProduct
}
```

**4. Navegación tipada**

```tsx
import { Link } from '@tanstack/react-router'

// TypeScript avisa si la ruta no existe o le faltan params
<Link to="/products/$productId" params={{ productId: '42' }}>
  Ver producto
</Link>
```

**5. Query strings con validación**

```tsx
export const Route = createFileRoute('/products')({
  validateSearch: (search) => ({
    page: Number(search.page ?? 1),
    status: (search.status as string) ?? 'active',
  }),
})
```

---

## Lo que NO hace

- **No gestiona estado global** — para eso existen librerías como Zustand o TanStack Query.
- **No reemplaza a TanStack Query** — los `loaders` del router son para datos iniciales; TanStack Query gestiona caché y sincronización continua.
- **No es un servidor** — solo controla la navegación en el cliente. Las llamadas a la API siguen yendo al backend C#/.NET.
- **No genera rutas automáticamente** — el archivo `routeTree.gen.ts` se regenera ejecutando el CLI de la herramienta, no es magia silenciosa.

---

*En resumen: TanStack Router es el "controlador de URLs" del frontend, con la misma seguridad de tipos que ya esperas del backend en C#, lo que elimina errores de navegación enteros en tiempo de compilación.*
