# Tailwind CSS

**¿Qué es?** Un framework CSS que te da clases de utilidad predefinidas para aplicar estilos directamente en el HTML/JSX, sin escribir hojas de estilo separadas.

---

## ¿Por qué existe?

El problema clásico del CSS tradicional: cada componente acaba con su propio archivo `.css`, lleno de nombres de clases inventados (`card-container`, `main-wrapper`, `button-blue-big`) que colisionan entre sí, son difíciles de mantener y no siguen ningún estándar.

La analogía en backend C#: imagina que en lugar de usar los tipos primitivos del lenguaje (`int`, `string`, `bool`), cada desarrolladora creara sus propios tipos para cada caso. El resultado sería caos de nombres y duplicidad. Tailwind es como el sistema de tipos de C# para el CSS: un vocabulario compartido, predecible y consistente.

---

## ¿Cuándo y para qué se usa?

Aparece en cualquier frontend React como sistema de estilos único: sin archivos `.css` por componente ni CSS-in-JS. Todo el estilo visual — layouts, colores, tipografía, espaciados — se declara mediante clases de Tailwind directamente en los componentes `.tsx`, ya sea una tienda online, un blog o un panel de administración.

La configuración principal vive en el archivo `tailwind.config.ts` en la raíz del proyecto frontend, donde se definen los colores corporativos y las fuentes propias de cada proyecto.

---

## Lo mínimo que necesitas saber

**1. Las clases son el estilo**

```tsx
// En lugar de escribir CSS, compones clases directamente
<div className="flex items-center gap-4 p-6 bg-white rounded-lg shadow">
  <h2 className="text-xl font-semibold text-gray-800">Proyecto activo</h2>
</div>
```

**2. El sistema de espaciado es numérico y predecible**

`p-4` = `padding: 1rem`, `p-8` = `padding: 2rem`. La escala base es `1 unidad = 0.25rem`.

**3. Responsive con prefijos**

```tsx
// Columna en móvil, fila en pantallas medianas y superiores
<div className="flex flex-col md:flex-row">
```

**4. Estados interactivos con prefijos**

```tsx
<button className="bg-blue-600 hover:bg-blue-700 disabled:opacity-50">
  Guardar
</button>
```

**5. Tailwind solo incluye lo que usas**

En build de producción, elimina automáticamente las clases no utilizadas. No necesitas preocuparte por el tamaño del CSS final.

---

## Lo que NO hace

- No es una librería de componentes: no hay un `<Button>` ni un `<Modal>` listos para usar. Solo estilos.
- No reemplaza la lógica de estado ni el comportamiento interactivo — eso sigue siendo React.
- No genera JavaScript: es puramente CSS en tiempo de compilación.
- No impone estructura de carpetas ni arquitectura de componentes.

---

*En resumen: Tailwind te permite construir interfaces consistentes y mantenibles escribiendo clases descriptivas directamente en el JSX, sin salir del componente ni inventar nombres de clases.*
