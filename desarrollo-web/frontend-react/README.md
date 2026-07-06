# React y su ecosistema — Guía de tecnologías

Documentación introductoria del stack frontend más habitual hoy en día: **React + Vite + TypeScript**, pensada para desarrolladoras con experiencia en backend (C#/.NET) que se acercan al frontend por primera vez.

Cada archivo explica qué es la tecnología, por qué existe, cuándo se usa y lo mínimo que necesitas saber para no perderte, apoyándose en analogías con el backend cuando ayudan a fijar el concepto.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Está pensado para que cada concepto apoye al siguiente.

### 1. Fundamentos del lenguaje y entorno

Antes de entrar en frameworks, entiende sobre qué se construye todo.

| # | Archivo | Por qué leerlo primero |
|---|---|---|
| 1 | [TypeScript](TypeScript.md) | El lenguaje base. Sin esto, el código fuente no tiene sentido. |
| 2 | [pnpm](pnpm.md) | El gestor de paquetes — equivalente a `dotnet restore` / NuGet. |

### 2. El framework de UI y el entorno de desarrollo

El núcleo del frontend: cómo se construye la interfaz y cómo se ejecuta en local.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [React](React.md) | El framework principal. Todo lo demás orbita alrededor de esto. |
| 4 | [Vite](Vite.md) | El servidor de desarrollo y bundler. Equivale al `dotnet watch`. |
| 5 | [@vitejs/plugin-react](vitejs-plugin-react.md) | Conecta Vite con React. Rara vez lo tocarás, pero conviene saber qué hace. |

### 3. Estilos

Cómo se aplica el diseño visual. Tailwind es la pieza central; el resto son complementos.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 6 | [Tailwind CSS](TailwindCSS.md) | El sistema de estilos. Leer antes que cualquier otro de esta sección. |
| 7 | [@tailwindcss/vite](tailwindcss-vite.md) | Integración de Tailwind con Vite. Configuración, no lógica. |
| 8 | [class-variance-authority](class-variance-authority.md) | Cómo se definen variantes de componentes (ej. botón primario / secundario). |
| 9 | [tailwind-merge](tailwind-merge.md) | Evita conflictos cuando se combinan clases Tailwind dinámicamente. |
| 10 | [tailwindcss-animate](tailwindcss-animate.md) | Animaciones CSS listas para usar. Complemento menor. |

### 4. Componentes de interfaz

Las piezas visuales reutilizables que componen la UI.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 11 | [Radix UI](RadixUI.md) | La base de accesibilidad sobre la que se construye shadcn/ui. Leer primero. |
| 12 | [shadcn/ui](shadcn-ui.md) | Una biblioteca de componentes muy extendida, construida sobre Radix UI. |
| 13 | [Lucide React](LucideReact.md) | Los iconos. Sencillo, pero está en todos lados. |
| 14 | [sonner](sonner.md) | Las notificaciones toast (mensajes de éxito, error, etc.). |

### 5. Estado y datos

Cómo la aplicación guarda información y se comunica con el backend.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 15 | [Zustand](Zustand.md) | Estado global del cliente (sesión, filtros, UI compartida). Leer antes que React Query. |
| 16 | [TanStack Query](TanStackQuery.md) | Fetching y caché de datos del servidor. El puente con tu API. |
| 17 | [TanStack Router](TanStackRouter.md) | Las rutas de la aplicación. Equivale a los controllers de ASP.NET, pero en el cliente. |

### 6. Utilidades

Pequeñas herramientas que aparecen en el código con frecuencia.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 18 | [clsx](clsx.md) | Combinar clases CSS condicionalmente. Dos minutos de lectura. |

### 7. Calidad de código

Herramientas que mantienen el código limpio y coherente. No necesitas configurarlas, pero conviene entender qué hace cada una cuando el linter se queja.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 19 | [ESLint](ESLint.md) | El linter general. Leer antes que los plugins. |
| 20 | [@eslint/js](eslint-js.md) | Configuración base de ESLint para JavaScript. |
| 21 | [typescript-eslint](typescript-eslint.md) | Añade reglas específicas de TypeScript al linter. |
| 22 | [eslint-plugin-react-hooks](eslint-plugin-react-hooks.md) | Valida el uso correcto de los Hooks de React. |
| 23 | [eslint-plugin-react-refresh](eslint-plugin-react-refresh.md) | Garantiza que el hot reload funcione correctamente en desarrollo. |

### 8. Testing

Cómo se prueban los componentes y la lógica frontend.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 24 | [Vitest](Vitest.md) | El framework de tests. Equivale a xUnit en .NET. Leer primero. |
| 25 | [jsdom](jsdom.md) | Simula el navegador durante los tests. Necesario para entender el entorno. |
| 26 | [@testing-library/react](testing-library-react.md) | Cómo se renderizan y testean componentes React. |
| 27 | [@testing-library/jest-dom](testing-library-jest-dom.md) | Matchers adicionales para hacer assertions sobre el DOM. |
| 28 | [@testing-library/user-event](testing-library-user-event.md) | Simula clics, escritura y otras interacciones reales del usuario. |

---

## Índice completo por categoría

<details>
<summary>Ver todos los archivos</summary>

**Core Framework & Build**
- [React](React.md)
- [Vite](Vite.md)
- [@vitejs/plugin-react](vitejs-plugin-react.md)
- [TypeScript](TypeScript.md)

**Estilos**
- [Tailwind CSS](TailwindCSS.md)
- [@tailwindcss/vite](tailwindcss-vite.md)
- [tailwind-merge](tailwind-merge.md)
- [tailwindcss-animate](tailwindcss-animate.md)
- [class-variance-authority](class-variance-authority.md)

**Componentes UI**
- [shadcn/ui](shadcn-ui.md)
- [Radix UI](RadixUI.md)
- [Lucide React](LucideReact.md)
- [sonner](sonner.md)

**Estado y Datos**
- [Zustand](Zustand.md)
- [TanStack Query](TanStackQuery.md)
- [TanStack Router](TanStackRouter.md)

**Utilidades**
- [clsx](clsx.md)

**Herramientas de Desarrollo**
- [pnpm](pnpm.md)
- [ESLint](ESLint.md)
- [@eslint/js](eslint-js.md)
- [typescript-eslint](typescript-eslint.md)
- [eslint-plugin-react-hooks](eslint-plugin-react-hooks.md)
- [eslint-plugin-react-refresh](eslint-plugin-react-refresh.md)

**Testing**
- [Vitest](Vitest.md)
- [@testing-library/react](testing-library-react.md)
- [@testing-library/jest-dom](testing-library-jest-dom.md)
- [@testing-library/user-event](testing-library-user-event.md)
- [jsdom](jsdom.md)

</details>

---

> Si el backend es .NET, echa un vistazo a la guía de [De C# WPF a C# para web](../de-wpf-a-web/README.md) y, en concreto, a [Comunicación entre frontend y backend](../de-wpf-a-web/Comunicacion-Frontend-Backend.md) para ver cómo se conectan los dos mundos.
