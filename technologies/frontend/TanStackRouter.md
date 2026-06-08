# TanStack Router

**Versión usada:** 1.170.10 | **Categoría:** Estado y Datos

## ¿Qué es?

Una librería de enrutamiento client-side para React con tipado completo de extremo a extremo. Gestiona qué componente se muestra según la URL del navegador, con TypeScript integrado desde el diseño.

---

## ¿Por qué existe?

En una aplicación de página única (SPA), el navegador no recarga la página al navegar. Alguien tiene que interceptar los cambios de URL y decidir qué renderizar. TanStack Router hace exactamente eso, pero con una ventaja clave respecto a alternativas como React Router: **los parámetros de ruta y query strings son completamente tipados**.

Analogía .NET: imagina que en ASP.NET Core, cuando accedes a `[HttpGet("{id}")]`, el parámetro `id` llegara siempre como `object` y tuvieras que castearlo manualmente. TanStack Router es como tener `[FromRoute] int id` con inferencia automática, pero para el frontend.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, TanStack Router define la estructura de navegación de la SPA React. Cada ruta del frontend corresponde a una vista de la aplicación (dashboard, detalle de proyecto, gestión de tareas, etc.). El router también se encarga de cargar datos antes de renderizar una vista (`loaders`), lo que evita estados de carga inconsistentes.

La configuración central vive en `src/routeTree.gen.ts` (generado automáticamente) y en los archivos de ruta bajo `src/routes/`.

---

## Lo mínimo que necesitas saber

**1. Las rutas son archivos**

Cada archivo en `src/routes/` define una ruta. El nombre del archivo determina la URL.

```
src/routes/
  index.tsx          →  /
  projects/
    index.tsx        →  /projects
    $projectId.tsx   →  /projects/123  (parámetro dinámico)
```

**2. Parámetros de ruta tipados**

```tsx
// src/routes/projects/$projectId.tsx
export const Route = createFileRoute('/projects/$projectId')({
  component: ProjectDetail,
})

function ProjectDetail() {
  const { projectId } = Route.useParams() // projectId: string, inferido automáticamente
  return <h1>Proyecto {projectId}</h1>
}
```

**3. Loaders — datos antes de renderizar**

```tsx
export const Route = createFileRoute('/projects/$projectId')({
  loader: ({ params }) => fetchProject(params.projectId),
  component: ProjectDetail,
})

function ProjectDetail() {
  const project = Route.useLoaderData() // tipado según lo que retorna fetchProject
}
```

**4. Navegación tipada**

```tsx
import { Link } from '@tanstack/react-router'

// TypeScript avisa si la ruta no existe o le faltan params
<Link to="/projects/$projectId" params={{ projectId: '42' }}>
  Ver proyecto
</Link>
```

**5. Query strings con validación**

```tsx
export const Route = createFileRoute('/projects')({
  validateSearch: (search) => ({
    page: Number(search.page ?? 1),
    status: (search.status as string) ?? 'active',
  }),
})
```

---

## Lo que NO hace

- **No gestiona estado global** — para eso existe Zustand o TanStack Query en el proyecto.
- **No reemplaza a TanStack Query** — los `loaders` del router son para datos iniciales; TanStack Query gestiona caché y sincronización continua.
- **No es un servidor** — solo controla la navegación en el cliente. Las llamadas a la API siguen yendo al backend C#/.NET.
- **No genera rutas automáticamente** — el archivo `routeTree.gen.ts` se regenera ejecutando el CLI del proyecto, no es magia silenciosa.

---

*En resumen: TanStack Router es el "controlador de URLs" del frontend, con la misma seguridad de tipos que ya esperas del backend en C#, lo que elimina errores de navegación enteros en tiempo de compilación.*
