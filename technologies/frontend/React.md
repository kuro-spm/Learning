# React

**¿Qué es?** React es una librería de JavaScript para construir interfaces de usuario mediante componentes reutilizables. Se encarga exclusivamente de la capa de presentación (la "V" en MVC).

---

## ¿Por qué existe?

Antes de React, actualizar la interfaz cuando cambiaban los datos implicaba manipular el DOM manualmente, lo que generaba código frágil y difícil de mantener. El equivalente en backend sería construir respuestas HTTP concatenando strings en lugar de usar un ORM o un template engine.

React resuelve esto con un **DOM virtual**: calcula los cambios mínimos necesarios y solo actualiza lo que realmente cambió, de forma declarativa. Tú describes *qué* debe verse, no *cómo* actualizar el HTML paso a paso.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, React (v19.2.6) es el framework principal del frontend. El backend en C#/.NET expone una API REST, y React consume esos endpoints para renderizar la interfaz: dashboards de proyectos, formularios, vistas de tareas, etc.

La comunicación es clara: .NET gestiona datos y lógica de negocio; React gestiona lo que el usuario ve e interactúa.

---

## Lo mínimo que necesitas saber

**1. Componentes — la unidad básica**

Un componente es una función que devuelve JSX (HTML dentro de TypeScript).

```tsx
function ProjectCard({ title }: { title: string }) {
  return <div className="card">{title}</div>;
}
```

**2. Props — parámetros del componente**

Son los argumentos que se pasan al componente, como parámetros en un método C#.

```tsx
<ProjectCard title="EcoWave Q2" />
```

**3. Estado con `useState` — datos que cambian**

Cuando un dato cambia y debe reflejarse en pantalla, se usa `useState`.

```tsx
const [isOpen, setIsOpen] = useState(false);
```

**4. Efectos con `useEffect` — lógica al montar/actualizar**

Equivale a un inicializador o listener. Aquí se hacen las llamadas a la API.

```tsx
useEffect(() => {
  fetch("/api/projects").then(res => res.json()).then(setProjects);
}, []);
```

**5. JSX — no es HTML puro**

`class` se escribe `className`, los eventos van en camelCase (`onClick`), y las expresiones JS van entre `{}`.

---

## Lo que NO hace

- **No gestiona el servidor ni rutas backend** — eso sigue siendo responsabilidad de .NET.
- **No es un framework completo por sí solo** — no incluye router ni cliente HTTP (se usan librerías adicionales como React Router o Axios).
- **No reemplaza CSS** — solo genera el HTML; los estilos son aparte.
- **No persiste datos** — el estado de React vive en memoria; al recargar la página, se pierde.

---

*En resumen: React es el motor de la interfaz en EcoWaveProjectManagement; convierte los datos que devuelve tu API de .NET en pantallas interactivas, organizando todo en componentes reutilizables y actualizando la UI automáticamente cuando los datos cambian.*
