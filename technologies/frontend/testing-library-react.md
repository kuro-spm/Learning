# @testing-library/react

**¿Qué es?** Una librería de utilidades para testear componentes React simulando cómo un usuario real interactúa con la interfaz: buscando elementos por texto visible, labels y roles, no por detalles internos de implementación.

---

## ¿Por qué existe?

Antes de esta librería, los tests de componentes React solían acceder al DOM buscando clases CSS o referencias internas del componente. Esto provocaba que los tests se rompieran ante cualquier refactor visual, aunque el comportamiento fuese correcto.

**Analogía backend:** Imagina testear un endpoint de tu API en C#/.NET verificando el nombre interno de las variables del controlador en lugar del contrato HTTP (status code, cuerpo de respuesta). Sería frágil e inútil. `@testing-library/react` aplica esa misma filosofía al frontend: testea el contrato observable por el usuario, no los detalles internos.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, esta librería se usa junto con **Vitest** (o Jest) para testear los componentes React del frontend. Los tests viven en archivos `*.test.tsx` o `*.spec.tsx` junto a los componentes que prueban, dentro de la carpeta `frontend/src`.

Se usa principalmente para validar que:
- Los formularios renderizan los campos correctos y responden a la entrada del usuario.
- Los componentes de gestión de proyectos muestran los datos esperados.
- Las interacciones (clics, cambios de input) producen el comportamiento correcto en la UI.

---

## Lo mínimo que necesitas saber

**1. `render` — monta el componente en un DOM virtual**
```tsx
import { render } from '@testing-library/react';
import ProjectCard from './ProjectCard';

render(<ProjectCard name="EcoWave" status="active" />);
```

**2. `screen` — accede a los elementos renderizados**
```tsx
import { render, screen } from '@testing-library/react';

render(<ProjectCard name="EcoWave" status="active" />);
const title = screen.getByText('EcoWave');
```

**3. `userEvent` — simula interacciones reales del usuario**
```tsx
import userEvent from '@testing-library/user-event';

const input = screen.getByLabelText('Nombre del proyecto');
await userEvent.type(input, 'EcoWave');
expect(input).toHaveValue('EcoWave');
```

**4. Queries principales**

| Query | Cuándo usarla |
|---|---|
| `getByText` | Texto visible en pantalla |
| `getByLabelText` | Inputs asociados a un `<label>` |
| `getByRole` | Elementos por rol ARIA (`button`, `heading`, etc.) |
| `queryByText` | Igual que `getBy` pero devuelve `null` si no existe (útil para afirmar ausencia) |

---

## Lo que NO hace

- **No ejecuta un navegador real.** Usa un DOM simulado (jsdom). Para tests end-to-end con navegador real, existe Playwright o Cypress.
- **No testea estilos CSS.** Verifica estructura y comportamiento, no apariencia visual.
- **No reemplaza los tests de API o de lógica de negocio.** Su alcance es estrictamente el componente UI.
- **No gestiona el servidor de pruebas ni el entorno.** Eso lo hace Vitest o Jest.

---

*En resumen: `@testing-library/react` te permite escribir tests de componentes que sobreviven a los refactors porque se centran en lo que ve y hace el usuario, no en cómo está construido el código por dentro.*
