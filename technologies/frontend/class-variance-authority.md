# class-variance-authority

**¿Qué es?** Una librería de TypeScript que permite definir variantes de clases CSS de forma estructurada y con tipado seguro, eliminando la concatenación manual de strings de clases.

---

## ¿Por qué existe?

En React, los componentes visuales suelen tener múltiples variantes: un botón puede ser `primary`, `secondary` o `destructive`; puede ser `sm`, `md` o `lg`. Sin una herramienta como esta, el código acaba lleno de condicionales para construir strings de clases:

```tsx
// Sin CVA: frágil y difícil de mantener
const className = `btn ${variant === "primary" ? "bg-blue-500" : ""} ${size === "sm" ? "text-sm" : "text-base"}`;
```

Es el equivalente frontend a tener un método C# con una cadena de `if/else` para construir una query SQL en vez de usar un query builder tipado. CVA actúa como ese query builder: estructura, tipo y valida las variantes.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, CVA se usa en la capa de componentes UI del frontend React para definir los estilos de elementos reutilizables como botones, badges, alertas y tarjetas de proyecto. Encontrarás su uso principalmente en `src/components/ui/` junto con Tailwind CSS como motor de clases.

---

## Lo mínimo que necesitas saber

**1. La función `cva` define variantes**

```tsx
import { cva } from "class-variance-authority";

const button = cva(
  "rounded font-medium transition-colors", // clases base, siempre presentes
  {
    variants: {
      intent: {
        primary: "bg-blue-600 text-white hover:bg-blue-700",
        danger:  "bg-red-600 text-white hover:bg-red-700",
      },
      size: {
        sm: "px-3 py-1 text-sm",
        md: "px-5 py-2 text-base",
      },
    },
    defaultVariants: {
      intent: "primary",
      size: "md",
    },
  }
);
```

**2. El resultado es una función que devuelve un string de clases**

```tsx
button({ intent: "danger", size: "sm" });
// => "rounded font-medium transition-colors bg-red-600 text-white hover:bg-red-700 px-3 py-1 text-sm"
```

**3. Se integra con TypeScript para autocompletar las variantes**

```tsx
// VariantProps extrae el tipo de las variantes para usarlo en props
import { type VariantProps } from "class-variance-authority";

type ButtonProps = VariantProps<typeof button> & { label: string };

function Button({ intent, size, label }: ButtonProps) {
  return <button className={button({ intent, size })}>{label}</button>;
}
```

**4. Variantes compuestas (casos especiales)**

```tsx
compoundVariants: [
  { intent: "primary", size: "sm", class: "uppercase tracking-wide" },
]
```

Permiten aplicar clases solo cuando se da una combinación concreta de variantes.

---

## Lo que NO hace

- No genera CSS: solo combina nombres de clases existentes (de Tailwind u otro sistema).
- No reemplaza a Tailwind ni a CSS Modules; trabaja encima de ellos.
- No gestiona temas visuales globales (eso lo hace Tailwind o CSS custom properties).
- No tiene relación con la lógica de negocio ni con el estado del componente.

---

*En resumen: CVA es el contrato tipado entre las props de un componente React y sus clases CSS, evitando la concatenación manual y haciendo las variantes visuales tan predecibles como un enum en C#.*
