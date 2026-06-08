# clsx

**¿Qué es?** Una función de utilidad minimalista que combina nombres de clases CSS de forma condicional. Recibe cualquier combinación de strings, objetos y arrays, y devuelve un único string limpio con las clases resultantes.

---

## ¿Por qué existe?

En React, las clases CSS se asignan como strings en el atributo `className`. El problema aparece cuando esas clases son dinámicas: dependiendo del estado del componente, necesitas incluir unas clases y excluir otras.

Sin `clsx`, el código acaba así:

```tsx
className={"btn" + (isActive ? " btn--active" : "") + (isDisabled ? " btn--disabled" : "")}
```

Esto es el equivalente frontend al encadenamiento manual de strings en C# para construir una query SQL en lugar de usar un query builder. Funciona, pero es ilegible y propenso a errores (espacios de más, clases vacías, etc.).

`clsx` actúa como ese query builder: tú declaras las condiciones, él construye el string final.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, `clsx` se usa en los componentes React del frontend para gestionar estilos condicionales. Por ejemplo, en componentes de estado de tareas, botones con variantes, o tarjetas que cambian apariencia según la prioridad del proyecto. Cualquier componente que tenga más de una variante visual probablemente usa `clsx` internamente.

---

## Lo mínimo que necesitas saber

**1. Uso básico: combinar clases fijas y condicionales**

```tsx
import clsx from 'clsx';

const clase = clsx('btn', isActive && 'btn--active', isDisabled && 'btn--disabled');
// Resultado si isActive=true, isDisabled=false: "btn btn--active"
```

**2. Sintaxis con objeto (más legible para múltiples condiciones)**

```tsx
const clase = clsx({
  'btn--active': isActive,
  'btn--disabled': isDisabled,
  'btn--loading': isLoading,
});
```

**3. Ignorar valores falsy automáticamente**

`clsx` ignora `false`, `null`, `undefined` y `0`. No necesitas guardar nada de eso.

```tsx
clsx('base', false, null, undefined, 'visible');
// Resultado: "base visible"
```

**4. Combinación libre de formatos**

```tsx
clsx('fixed', ['top-0', 'w-full'], { 'bg-red-500': hasError });
```

---

## Lo que NO hace

- **No aplica estilos**: solo genera strings de clases. El trabajo real lo hace CSS o Tailwind.
- **No valida** que las clases existan en tu hoja de estilos.
- **No reemplaza a CSS-in-JS** (como styled-components): son herramientas con propósitos distintos.
- **No tiene lógica de theming**: si necesitas tokens de diseño o variables de tema, eso es responsabilidad de otra capa.

---

*En resumen: `clsx` es una pequeña función de una sola responsabilidad que mantiene el código de clases CSS legible cuando la UI tiene estados dinámicos — el principio de responsabilidad única aplicado a la capa de presentación.*
