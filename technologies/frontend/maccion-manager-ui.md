# @maccion/manager-ui

**¿Qué es?**
Una librería de componentes React de uso interno en Maccion, diseñada específicamente para construir interfaces de tipo escritorio/manager: tablas complejas, paneles, formularios de gestión y layouts administrativos.

---

## ¿Por qué existe?

En un proyecto backend, si necesitas enviar emails no reinventas el cliente SMTP: usas una librería como MailKit. El mismo principio aplica aquí. Sin `@maccion/manager-ui`, cada desarrolladora frontend tendría que construir desde cero componentes como tablas con filtros, paginación, modales de confirmación o sidebars colapsables — y cada una los implementaría de forma distinta.

Esta librería centraliza esos patrones visuales y de interacción para que el equipo frontend no duplique trabajo y todas las interfaces de tipo manager tengan coherencia visual y de comportamiento.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, la parte frontend (React) usa `@maccion/manager-ui` para construir las pantallas de administración: gestión de proyectos, visualización de tareas, dashboards de estado y formularios de configuración.

Es el equivalente frontend a lo que en el backend de C#/.NET serían los servicios de infraestructura compartida: algo que ya existe, está probado, y el equipo consume sin necesidad de implementarlo.

---

## Lo mínimo que necesitas saber

**1. Se importa como cualquier paquete npm**
```bash
# Ya está instalado en el proyecto; no necesitas hacer nada
npm install @maccion/manager-ui
```

**2. Los componentes se usan directamente en JSX/TSX**
```tsx
import { DataTable, PageLayout } from '@maccion/manager-ui';

function ProjectList() {
  return (
    <PageLayout title="Proyectos">
      <DataTable columns={columns} data={projects} />
    </PageLayout>
  );
}
```

**3. Los componentes aceptan `props` para configurarse**
```tsx
<DataTable
  columns={columns}
  data={projects}
  paginated
  onRowClick={(row) => navigate(`/projects/${row.id}`)}
/>
```
Piensa en las `props` como los parámetros de un método: definen el comportamiento del componente sin que tengas que tocar su implementación interna.

**4. Tiene su propio sistema de estilos**
Los componentes ya vienen con estilos aplicados. No necesitas escribir CSS para usarlos; si necesitas ajustes visuales, el proyecto puede pasar `className` o usar las variables de tema que expone la librería.

---

## Lo que NO hace

- No es un framework completo: no gestiona rutas, estado global ni llamadas a la API. Solo es UI.
- No reemplaza a React: es una colección de componentes que viven dentro de una aplicación React, no por encima de ella.
- No define la logica de negocio: los datos que muestran los componentes vienen de fuera (del estado de la app o de las llamadas al backend .NET).
- No es una librería de estilos genérica como Tailwind o Bootstrap: está pensada exclusivamente para interfaces de tipo manager/escritorio en el contexto de Maccion.

---

*En resumen: `@maccion/manager-ui` es la caja de herramientas de componentes visuales que el frontend de EcoWaveProjectManagement usa para construir sus pantallas de administración, de la misma manera que el backend usa librerías NuGet para no reimplementar funcionalidad común.*
