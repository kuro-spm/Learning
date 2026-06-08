# shadcn/ui

**¿Qué es?** Una colección de componentes de interfaz de usuario listos para usar en React, construidos sobre Radix UI y estilizados con Tailwind CSS. No es una librería que instalas como dependencia: copias el código fuente directamente en tu proyecto.

---

## ¿Por qué existe?

Construir componentes de UI accesibles desde cero es costoso y propenso a errores: gestión de foco, navegación por teclado, roles ARIA... todo eso ya está resuelto aquí.

La analogía en backend: imagina tener que implementar autenticación desde cero cada vez que creas un proyecto en .NET, cuando ya existe `Microsoft.AspNetCore.Identity`. shadcn/ui hace algo similar para los componentes visuales: te da una base sólida y probada, pero sin ocultarte el código. A diferencia de un paquete NuGet que vive en `node_modules`, el código de cada componente vive en tu repositorio y puedes modificarlo libremente.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, shadcn/ui es la base visual del frontend React. Todos los elementos de interfaz reutilizables (botones, diálogos, tablas, formularios, menús) provienen de esta biblioteca. Los componentes se encuentran en `frontend/src/components/ui/` y se consumen desde las vistas y páginas de la aplicación.

---

## Lo mínimo que necesitas saber

**1. Los componentes son código tuyo.** Se generan con la CLI y quedan en tu proyecto:

```bash
npx shadcn@latest add button
```

Esto crea el archivo `components/ui/button.tsx` que puedes editar.

**2. Se usan como cualquier componente React:**

```tsx
import { Button } from "@/components/ui/button";

export function GuardarProyecto() {
  return <Button variant="default">Guardar</Button>;
}
```

**3. Las variantes controlan el estilo.** Cada componente expone una prop `variant` y otras opciones predefinidas:

```tsx
<Button variant="destructive">Eliminar</Button>
<Button variant="outline" size="sm">Cancelar</Button>
```

**4. Los componentes son accesibles por defecto.** El foco, los atajos de teclado y los atributos ARIA ya están gestionados internamente por Radix UI.

**5. La personalización va en el componente local.** Si necesitas ajustar estilos, editas directamente `components/ui/button.tsx`, no sobreescribes clases desde fuera.

---

## Lo que NO hace

- **No es un framework de estilos.** El sistema de estilos es Tailwind CSS; shadcn/ui lo usa, pero no lo reemplaza.
- **No gestiona estado ni lógica de negocio.** Solo es UI: presentación, accesibilidad y estructura visual.
- **No se actualiza como un paquete normal.** Como el código es tuyo, las actualizaciones son manuales componente a componente.
- **No incluye iconos propios.** Para iconos, el proyecto usa una librería separada (habitualmente `lucide-react`).

---

*En resumen: shadcn/ui te da componentes de interfaz accesibles y editables que viven en tu código fuente, permitiendo construir la UI de EcoWaveProjectManagement de forma consistente y sin reinventar la rueda.*
