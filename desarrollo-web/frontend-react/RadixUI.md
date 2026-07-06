# Radix UI

**¿Qué es?** Una colección de componentes UI primitivos y headless para React: sin estilos propios, pero con toda la lógica de accesibilidad, teclado y ARIA incorporada.

---

## ¿Por qué existe?

Construir un desplegable, un modal o un menú accesible desde cero es sorprendentemente complicado. No basta con mostrar y ocultar un `<div>`: hay que gestionar el foco, las teclas de escape, los roles ARIA, los lectores de pantalla y el comportamiento del portal en el DOM.

**Analogía backend:** es como ASP.NET Identity. Podrías implementar autenticación desde cero, pero Identity ya resuelve las partes difíciles (hashing, tokens, claims) y tú solo configuras lo que necesitas. Radix hace lo mismo con la interacción UI: resuelve la complejidad accesible y tú pones los estilos.

---

## ¿Cuándo y para qué se usa?

Radix UI rara vez se usa directamente: lo habitual es que actúe como la capa inferior de una librería de componentes como **shadcn/ui**. Cuando usas un `<Dialog>`, un `<DropdownMenu>` o un `<Select>` de shadcn/ui —en el formulario de registro de una app, el menú de acciones de una tienda online o los filtros de un panel de administración—, por debajo hay un componente Radix UI gestionando toda la lógica.

Si en algún momento necesitas un componente que shadcn/ui no cubre, puedes acudir directamente a Radix e instalarlo como paquete independiente (cada primitiva es un paquete separado, por ejemplo `@radix-ui/react-dialog`).

---

## Lo mínimo que necesitas saber

**1. Composicion por partes (compound components)**

Radix expone cada componente como un conjunto de piezas con roles claros:

```tsx
import * as Dialog from '@radix-ui/react-dialog';

<Dialog.Root>
  <Dialog.Trigger>Abrir</Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay />
    <Dialog.Content>
      <Dialog.Title>Titulo</Dialog.Title>
      <Dialog.Close>Cerrar</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

**2. Headless: tú pones los estilos**

Radix no aplica ningún estilo visual. Puedes usar Tailwind, CSS modules o lo que uses en el proyecto:

```tsx
<Dialog.Overlay className="fixed inset-0 bg-black/50" />
<Dialog.Content className="rounded-lg p-6 shadow-xl">
  ...
</Dialog.Content>
```

**3. Accesibilidad incluida**

Cierre con `Escape`, trampa de foco dentro del modal, roles `aria-*` correctos — todo viene configurado por defecto. No tienes que añadir nada.

**4. Estado controlado o no controlado**

Como los inputs de React, puedes dejarlo en modo no controlado (Radix gestiona el estado abierto/cerrado) o controlarlo tú con `open` y `onOpenChange`:

```tsx
<Dialog.Root open={isOpen} onOpenChange={setIsOpen}>
```

---

## Lo que NO hace

- **No incluye estilos.** No esperes ninguna apariencia visual por defecto.
- **No es una libreria de diseño.** No tiene sistema de tokens, temas ni colores. Eso lo aporta Tailwind o tu CSS.
- **No reemplaza a shadcn/ui.** Si el proyecto ya usa shadcn/ui, este envuelve Radix con estilos y convenciones; úsalo como primera opcion.
- **No gestiona datos ni estado de aplicacion.** Solo gestiona el estado de interaccion UI (abierto/cerrado, seleccionado, etc.).

---

*En resumen: Radix UI es la fontaneria accesible que hay detras de los componentes de shadcn/ui — no la veras directamente en el dia a dia, pero es la razon por la que los modales, menus y selects funcionan bien para todos los usuarios.*
