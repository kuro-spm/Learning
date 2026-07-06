# eslint-plugin-react-refresh

**¿Qué es?** Un plugin de ESLint que añade reglas específicas para garantizar que los componentes de React sean compatibles con Fast Refresh (la recarga en caliente del navegador durante el desarrollo).

---

## ¿Por qué existe?

Fast Refresh es el mecanismo que permite que los cambios en el código se reflejen en el navegador instantáneamente sin perder el estado de la aplicación. Sin embargo, para que funcione correctamente, los archivos `.tsx`/`.jsx` deben seguir ciertas convenciones: básicamente, cada archivo debe exportar solo componentes de React, no mezclar exports de componentes con exports de utilidades o constantes arbitrarias.

El problema que resuelve es silencioso: si un archivo viola estas reglas, Fast Refresh simplemente recarga la página entera en lugar de actualizar solo el componente afectado. No hay error visible, solo una experiencia de desarrollo más lenta.

**Analogía .NET:** es similar a cuando en un proyecto C# mezclas lógica de negocio dentro de un controlador. La aplicación funciona igual, pero viola una convención que empeora la mantenibilidad. Este plugin hace que ESLint te avise antes de que el problema pase desapercibido.

---

## ¿Cuándo y para qué se usa?

Aparece en cualquier proyecto React que use Vite como bundler, que incluye Fast Refresh por defecto: una tienda online, un blog o un panel de administración. Este plugin se configura en ESLint para marcar automáticamente los archivos (`src/**`) que rompan la compatibilidad con Fast Refresh, evitando recargas completas inesperadas durante el desarrollo.

---

## Lo mínimo que necesitas saber

**1. La regla principal: `only-export-components`**

Un archivo `.tsx` solo debe exportar componentes de React:

```tsx
// CORRECTO: solo exporta un componente
export default function ProductCard() {
  return <div>...</div>;
}

// INCORRECTO: mezcla componente con una constante
export const MAX_ITEMS = 10; // esto rompe Fast Refresh
export default function ProductCard() {
  return <div>...</div>;
}
```

**2. Exports nombrados de componentes también son válidos**

```tsx
// CORRECTO: múltiples componentes en un mismo archivo
export function ProductCard() { ... }
export function ProductBadge() { ... }
```

**3. El error que verás en la consola de ESLint**

```
Fast refresh only works when a file only exports components.
Use a separate file to share constants or functions.
```

**4. La solución habitual es mover el código no-componente**

```tsx
// utils/constants.ts
export const MAX_ITEMS = 10;

// components/ProductCard.tsx
import { MAX_ITEMS } from '../utils/constants';
export default function ProductCard() { ... }
```

---

## Lo que NO hace

- No afecta al build de producción: es una herramienta exclusiva de desarrollo.
- No configura ni instala Fast Refresh; solo valida que tu código sea compatible con él.
- No detecta errores de lógica en los componentes, solo problemas estructurales de exports.
- No reemplaza a otros plugins de ESLint como `eslint-plugin-react` o `eslint-plugin-react-hooks`.

---

*En resumen: este plugin actúa como un guardián que revisa que cada archivo del frontend siga las reglas necesarias para que la recarga en caliente funcione de forma óptima, ahorrando interrupciones durante el desarrollo.*
