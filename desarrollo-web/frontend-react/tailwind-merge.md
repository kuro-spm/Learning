# tailwind-merge

**¿Qué es?** Una utilidad de TypeScript que fusiona clases de Tailwind CSS resolviendo conflictos automáticamente, asegurando que la última clase relevante siempre gane.

---

## ¿Por qué existe?

Tailwind genera estilos mediante clases CSS atómicas: `p-4`, `text-red-500`, `bg-blue-200`. El problema surge cuando combinas clases de forma dinámica y dos clases afectan la misma propiedad CSS. Por ejemplo:

```tsx
// Sin tailwind-merge: ambas clases llegan al DOM, pero solo una se aplica
// según el orden en la hoja de estilos de Tailwind (no el orden en el string)
const clase = "p-4 p-8"; // ¿cuál gana? No es predecible
```

En backend C#/.NET esto sería análogo a tener dos llamadas a `builder.Services.AddSingleton<IServicio>()` con implementaciones distintas: solo una se registra, pero no está claro cuál sin leer el orden de arranque. `tailwind-merge` actúa como el equivalente a un sistema de registro explícito que garantiza qué implementación prevalece.

---

## ¿Cuándo y para qué se usa?

Aparece en cualquier componente React que construya clases CSS dinámicas de forma segura: un botón con variantes, una tarjeta de producto que cambia de estilo si está en oferta, o cualquier elemento cuya apariencia dependa de props. Se usa habitualmente en:

- Componentes reutilizables (`Button`, `Card`, `Badge`) que aceptan props de estilo personalizadas.
- Una función helper `cn()` (convention de facto) que combina `tailwind-merge` con `clsx` para gestionar clases condicionales.

```ts
// src/lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```tsx
// Uso en un componente
<Button className={cn("bg-blue-500", isActive && "bg-green-500")} />
// Resultado: "bg-green-500" — sin duplicados ni conflictos
```

---

## Lo mínimo que necesitas saber

**1. `twMerge` resuelve conflictos por propiedad CSS, no por orden de string.**

```ts
twMerge("px-4 px-8")        // => "px-8"
twMerge("text-sm text-lg")  // => "text-lg"
```

**2. Las clases sin conflicto se conservan todas.**

```ts
twMerge("p-4 font-bold text-red-500") // => "p-4 font-bold text-red-500"
```

**3. Se combina con `clsx` para manejar condicionales.**

```ts
cn("base-class", condicion && "extra-class", { "otro": activo })
```

**4. Versión en uso: `3.6.0`** — compatible con Tailwind v4.

---

## Lo que NO hace

- **No genera CSS**: solo manipula strings de clases. El CSS lo sigue generando Tailwind.
- **No valida** que las clases existan en tu configuración de Tailwind.
- **No reemplaza a `clsx`**: son complementarios. `clsx` gestiona condiciones; `twMerge` resuelve conflictos.
- **No afecta al backend**: es exclusivamente una herramienta de capa de presentación en el cliente.

---

*En resumen: `tailwind-merge` es el árbitro que decide qué clase de Tailwind prevalece cuando dos clases pelean por la misma propiedad CSS, haciendo que los componentes reutilizables sean predecibles sin importar qué estilos les pasen desde fuera.*
