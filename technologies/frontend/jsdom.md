# jsdom

**¿Qué es?** Una implementación de las APIs del navegador (DOM, ventana, eventos) escrita en JavaScript puro, que permite ejecutar y testear código frontend sin abrir ningún navegador real.

---

## ¿Por qué existe?

El código React manipula el DOM: crea nodos, escucha eventos, actualiza elementos. Para testear esa lógica necesitas un DOM disponible. El problema: los tests corren en Node.js, que no tiene navegador.

**Analogía .NET:** es como usar una implementación en memoria de `IDbContext` (EF Core InMemory) para testear tu capa de repositorios sin levantar SQL Server. jsdom hace lo mismo pero para el navegador: te da un "navegador falso" que vive en memoria durante el test.

Sin jsdom, cada test de componente requeriría lanzar Chromium, lo que haría los tests lentos, frágiles y difíciles de correr en CI.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, jsdom actúa como entorno de ejecución para todos los tests de componentes React del frontend. Jest lo configura automáticamente cuando el entorno es `jsdom`, por lo que no se invoca directamente en el código de tests.

La configuración relevante está en `jest.config.ts` o en `package.json`:

```ts
// jest.config.ts
export default {
  testEnvironment: "jsdom", // jsdom entra aquí
};
```

Cuando Vitest o Jest arrancan un test de componente, jsdom provee `document`, `window`, `HTMLElement` y todo lo que React necesita para renderizar.

---

## Lo mínimo que necesitas saber

**1. Lo activa la configuración, no el código**
No importas jsdom en tus tests. Se configura una vez y queda disponible globalmente.

**2. `document` y `window` existen en tus tests gracias a él**
```ts
test("el título existe", () => {
  document.title = "EcoWave";
  expect(document.title).toBe("EcoWave");
});
```

**3. Se combina con Testing Library para testear componentes React**
```tsx
import { render, screen } from "@testing-library/react";
import ProjectCard from "./ProjectCard";

test("muestra el nombre del proyecto", () => {
  render(<ProjectCard name="EcoWave" />);
  expect(screen.getByText("EcoWave")).toBeInTheDocument();
});
```

**4. Simula eventos del navegador**
Clicks, inputs, submit de formularios — jsdom los maneja como lo haría un navegador real.

---

## Lo que NO hace

- **No renderiza visualmente** nada: no hay pantalla, no hay píxeles.
- **No ejecuta CSS** ni calcula estilos reales (propiedades como `getComputedStyle` devuelven valores vacíos).
- **No es un sustituto para tests end-to-end**: para flujos completos como login o navegación real, se usa Playwright o Cypress.
- **No emula comportamientos específicos de cada navegador** (diferencias entre Chrome, Firefox, Safari).

---

*En resumen: jsdom es el "navegador de mentira" que permite que tus tests de React corran rápido y sin dependencias externas, de la misma forma que una base de datos en memoria te permite testear repositorios sin SQL Server.*
