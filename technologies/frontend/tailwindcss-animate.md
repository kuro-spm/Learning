# tailwindcss-animate

**¿Qué es?**
Plugin oficial para Tailwind CSS que añade clases de utilidad para animar elementos HTML directamente desde el marcado, sin escribir CSS personalizado.

---

## ¿Por qué existe?

Las animaciones CSS tienen una sintaxis verbosa y repetitiva. Para animar un elemento necesitas definir `@keyframes`, luego asignar propiedades como `animation-duration`, `animation-timing-function`, `animation-fill-mode`... todo en archivos CSS separados.

Analogía backend: es como tener que escribir a mano cada consulta SQL con `SqlCommand` y parámetros, cuando podrías usar Entity Framework y resolverlo en una línea. `tailwindcss-animate` hace lo mismo con las animaciones: encapsula la complejidad en clases reutilizables que aplicas directamente en el componente.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, este plugin se usa en el frontend React para dar feedback visual al usuario: transiciones de entrada y salida en modales, indicadores de carga, aparición de tarjetas de proyectos y notificaciones del sistema.

Al ser un plugin de Tailwind, ya está integrado en el pipeline de estilos del proyecto — no requiere imports adicionales en los componentes. Se activa añadiendo clases en el JSX/TSX.

---

## Lo mínimo que necesitas saber

**1. Clases de entrada (`animate-in`) y salida (`animate-out`)**
```tsx
<div className="animate-in fade-in duration-300">
  Contenido que aparece con fundido
</div>
```

**2. Tipos de animación combinables**
```tsx
// Fundido + desplazamiento hacia arriba al entrar
<div className="animate-in fade-in slide-in-from-bottom-4 duration-500">
  Modal de confirmación
</div>
```

**3. Control de duración y retardo**
```tsx
<div className="animate-in zoom-in duration-200 delay-150">
  Tarjeta de proyecto
</div>
```

**4. Animaciones de salida en componentes condicionales**
```tsx
{visible && (
  <div className="animate-in fade-in data-[state=closed]:animate-out data-[state=closed]:fade-out">
    Notificación
  </div>
)}
```

**5. El plugin se registra en `tailwind.config.ts`**
```ts
import tailwindcssAnimate from "tailwindcss-animate";

export default {
  plugins: [tailwindcssAnimate],
};
```

---

## Lo que NO hace

- **No es una librería de componentes animados** — no te da modales ni tooltips prediseñados. Solo te da las clases; tú construyes los componentes.
- **No gestiona el estado de visibilidad** — si un elemento debe aparecer o desaparecer según lógica de negocio, eso sigue siendo responsabilidad de React (`useState`, condicionales).
- **No reemplaza `framer-motion`** — para animaciones complejas con física, secuencias o gestos, se necesita una librería más potente. Este plugin cubre el 80% de los casos con animaciones simples.
- **No añade animaciones automáticamente** — debes aplicar las clases explícitamente en cada elemento.

---

*En resumen: `tailwindcss-animate` es la forma más rápida de añadir animaciones estándar en el frontend de EcoWaveProjectManagement, manteniendo la coherencia con el sistema de diseño de Tailwind y sin salir del JSX.*
