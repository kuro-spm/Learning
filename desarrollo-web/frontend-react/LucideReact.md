# Lucide React

**¿Qué es?** Una biblioteca de iconos SVG listos para usar como componentes React. Instalas el paquete, importas el icono por nombre y lo usas como cualquier componente JSX.

---

## ¿Por qué existe?

En el frontend necesitas iconos constantemente: botones de acción, estados de alerta, indicadores visuales. Sin una biblioteca como esta, las opciones son incómodas: descargar archivos SVG manualmente, incrustar HTML crudo, o depender de fuentes de iconos (como Font Awesome en modo fuente tipográfica) que añaden peso y complejidad.

La analogía en .NET sería tener que escribir a mano cada helper de formateo de fechas en lugar de usar `DateTimeOffset` o NodaTime — técnicamente posible, pero innecesario cuando existe una solución estándar y mantenida.

Lucide React resuelve esto: cada icono es un componente React independiente, tipado, escalable y sin dependencias externas de red.

---

## ¿Cuándo y para qué se usa?

Aparece en la capa de componentes UI de cualquier aplicación React para representar acciones y estados de forma visual: iconos en botones de creación de elementos, indicadores de estado en una lista de tareas, acciones de edición/eliminación en la tabla de productos de una tienda online, o elementos de navegación lateral en un panel de administración.

Cuando una interfaz tiene muchas acciones repetidas (editar, borrar, ver detalle, filtrar), Lucide proporciona un vocabulario visual coherente para todas ellas.

---

## Lo mínimo que necesitas saber

**1. Importación por nombre de icono**

```tsx
import { Trash2, PencilLine, CheckCircle } from "lucide-react";

function TaskActions() {
  return (
    <div>
      <PencilLine size={16} />
      <Trash2 size={16} color="red" />
      <CheckCircle size={16} />
    </div>
  );
}
```

**2. Props principales**

| Prop | Tipo | Descripción |
|---|---|---|
| `size` | `number` | Tamaño en píxeles (default: 24) |
| `color` | `string` | Color CSS o variable |
| `strokeWidth` | `number` | Grosor del trazo (default: 2) |
| `className` | `string` | Clases CSS / Tailwind |

**3. Compatibilidad con Tailwind**

```tsx
<CheckCircle className="text-green-500 w-5 h-5" />
```

Si el proyecto usa Tailwind, puedes controlar tamaño y color directamente con clases en lugar de props.

**4. Los nombres son descriptivos en inglés**

Busca iconos en [lucide.dev](https://lucide.dev) por concepto visual. El nombre del icono en la web es exactamente el nombre del componente en PascalCase.

---

## Lo que NO hace

- No es un sistema de componentes UI completo — solo proporciona iconos, no botones, modales ni layouts.
- No incluye iconos de marcas (logos de GitHub, Twitter, etc.) — para eso existe `react-icons` u otras bibliotecas.
- No gestiona estado ni lógica — un icono de "papelera" no borra nada por si solo; esa lógica vive en tu componente padre.
- No requiere configuración global — cada icono se importa y usa directamente, sin providers ni contextos.

---

*En resumen: Lucide React es la forma más directa de añadir iconos consistentes y tipados a cualquier componente React del proyecto, sin fricción y con control total sobre tamaño y estilo.*
