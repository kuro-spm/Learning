# @maccion/operativa-ui

Librería de componentes React desarrollada internamente por Maccion para construir interfaces de planta e interfaces táctiles. Proporciona los bloques visuales reutilizables que comparten todos los productos de operativa.

## ¿Por qué existe?

En .NET tenemos NuGet packages internos para encapsular lógica de negocio compartida entre proyectos: validadores, servicios comunes, modelos de dominio. El problema equivalente en el frontend es que cada equipo acaba creando sus propios botones, tablas y formularios, y con el tiempo divergen en estilo y comportamiento.

`@maccion/operativa-ui` resuelve exactamente eso: centraliza los componentes visuales específicos del contexto de planta (interfaces táctiles, pantallas industriales, dashboards de operativa) para que no se reimplementen en cada aplicación. Es el "paquete NuGet interno" del frontend de Maccion, pero para la UI.

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement** el frontend está construido en React. Esta librería provee los componentes de interfaz que se usan en las pantallas orientadas a operadores de planta: paneles de control, vistas táctiles de gestión de proyectos y elementos de navegación adaptados a entornos industriales.

Los componentes se importan directamente desde el paquete y se componen dentro de las páginas React del proyecto, igual que cualquier componente externo.

```tsx
import { OperativaPanel, TouchButton } from '@maccion/operativa-ui';

function PlantaDashboard() {
  return (
    <OperativaPanel title="Estado del proyecto">
      <TouchButton size="large" onClick={handleConfirm}>
        Confirmar recepción
      </TouchButton>
    </OperativaPanel>
  );
}
```

## Lo mínimo que necesitas saber

**1. Son componentes React.** Se usan como cualquier elemento JSX dentro de los archivos `.tsx` del proyecto.

**2. Están pensados para pantallas táctiles.** Los tamaños, espaciados y áreas de interacción están optimizados para dedos, no para ratón. No los uses en interfaces de escritorio estándar sin verificar que el diseño lo contempla.

**3. Reciben props para configurarse.** No hay que editar la librería; se parametriza desde fuera:

```tsx
<StatusBadge status="en-progreso" variant="compact" />
```

**4. El tipado está incluido.** Al ser TypeScript, el editor autocompleta las props disponibles. Si una prop no aparece en el autocompletado, no existe en ese componente.

**5. No gestiona estado global.** Solo pinta y emite eventos. La lógica de negocio y el estado siguen siendo responsabilidad del proyecto consumidor.

## Lo que NO hace

- No contiene lógica de negocio de EcoWave: no sabe nada de proyectos, usuarios ni flujos de trabajo.
- No hace llamadas a la API del backend; eso lo maneja el propio proyecto.
- No reemplaza a librerías generalistas como MUI o Tailwind; es un complemento especializado en operativa.
- No incluye enrutamiento ni manejo de autenticación.

---

*En resumen: `@maccion/operativa-ui` es la caja de piezas visuales estándar para interfaces de planta en React — úsala para componer pantallas táctiles sin reinventar los componentes desde cero.*
