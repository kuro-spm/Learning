# @vitejs/plugin-react

**¿Qué es?**
Es el plugin oficial que conecta Vite con React, habilitando la transformación de JSX y el sistema de recarga en caliente llamado Fast Refresh.

---

## ¿Por qué existe?

El navegador no entiende JSX (`<MiComponente />`) ni TypeScript directamente. Alguien tiene que transformar ese código a JavaScript puro antes de enviarlo al browser. En el mundo .NET, ese rol lo cumple el compilador de C# junto con MSBuild: tú escribes C# y la toolchain lo convierte en IL ejecutable. Aquí, Vite actúa como la toolchain de frontend, y este plugin le dice exactamente cómo procesar los archivos `.jsx` y `.tsx` de React.

Además, durante el desarrollo resuelve un problema clásico: cada vez que cambias una línea de código, no quieres que la página se recargue entera y pierda el estado de la aplicación. **Fast Refresh** actualiza solo el componente modificado, manteniendo el estado del resto de la UI intacto.

---

## ¿Cuándo y para qué se usa?

Se usa en cualquier proyecto que combine Vite con React: una tienda online, un panel de administración o una app de gestión de tareas. El plugin se registra en el archivo de configuración de Vite y actúa de forma transparente durante el desarrollo y el build de producción del frontend. No requiere intervención directa una vez configurado.

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

Cada vez que ejecutas `npm run dev`, Vite usa este plugin para compilar los componentes React del proyecto en tiempo real.

---

## Lo mínimo que necesitas saber

**1. Transforma JSX automáticamente**
No necesitas importar React en cada archivo (desde React 17+). El plugin configura el nuevo JSX transform por defecto.

```tsx
// Esto funciona sin escribir: import React from 'react'
export function Tarjeta({ titulo }: { titulo: string }) {
  return <h2>{titulo}</h2>
}
```

**2. Fast Refresh preserva el estado**
Si editas el estilo o la lógica de un componente, la página no se recarga entera. El estado (como un formulario a medias) se conserva.

**3. Solo actúa en desarrollo y build**
No genera ningún código que llegue al usuario final más allá del JavaScript compilado. No es una librería de runtime.

**4. Configuración mínima**
La configuración por defecto cubre el 99% de los casos. Solo necesitas pasarle opciones avanzadas si usas Babel plugins personalizados.

```ts
// Ejemplo con opción avanzada (poco frecuente)
react({ babel: { plugins: ['@emotion/babel-plugin'] } })
```

---

## Lo que NO hace

- No es un framework de UI ni reemplaza a React.
- No gestiona rutas, estado global ni llamadas a la API del backend .NET.
- No funciona de forma independiente: necesita Vite como base.
- No interviene en el servidor de C#/.NET; su alcance es exclusivamente el frontend.

---

*En resumen: `@vitejs/plugin-react` es la pieza que hace que Vite entienda React — sin él, Vite no sabría qué hacer con un archivo `.tsx`.*
