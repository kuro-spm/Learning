# Zustand

**¿Qué es?** Una librería minimalista para gestionar estado global en aplicaciones React. Permite que distintos componentes lean y modifiquen datos compartidos sin pasárselos manualmente de uno a otro.

---

## ¿Por qué existe?

En React, los datos fluyen de componentes padre a hijos mediante `props`. Cuando varios componentes alejados en el árbol necesitan el mismo dato, tienes que "perforar" ese dato por múltiples niveles intermedios — un antipatrón conocido como *prop drilling*.

La analogía en .NET: imagina que para leer un valor de configuración desde un servicio profundo, en lugar de inyectarlo con `IOptions<T>`, tuvieras que pasarlo a mano por cada constructor del camino. Zustand es el equivalente frontend del contenedor de dependencias — un lugar centralizado donde cualquier componente puede leer o escribir estado sin intermediarios.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, Zustand gestiona el estado global del cliente: datos de sesión del usuario autenticado, filtros activos en las vistas de proyectos, notificaciones en tiempo real y cualquier dato que múltiples partes de la UI necesiten compartir. Los stores de Zustand se definen en `frontend/src/stores/` y son consumidos directamente por los componentes que los necesitan.

---

## Lo mínimo que necesitas saber

**1. Crear un store**

```ts
// stores/userStore.ts
import { create } from 'zustand'

interface UserStore {
  name: string
  setName: (name: string) => void
}

export const useUserStore = create<UserStore>((set) => ({
  name: '',
  setName: (name) => set({ name }),
}))
```

**2. Leer estado en un componente**

```tsx
// components/Header.tsx
import { useUserStore } from '@/stores/userStore'

export function Header() {
  const name = useUserStore((state) => state.name)
  return <h1>Hola, {name}</h1>
}
```

**3. Modificar estado desde otro componente**

```tsx
const setName = useUserStore((state) => state.setName)
setName('Ana')
```

**4. Selectores para evitar re-renders innecesarios**

Selecciona solo lo que necesitas — el componente solo se re-renderiza si ese dato cambia, no si cambia cualquier cosa del store.

```tsx
const name = useUserStore((state) => state.name) // correcto
const store = useUserStore()                      // evitar: suscribe a todo
```

---

## Lo que NO hace

- **No persiste datos entre sesiones** por defecto (necesitas un middleware adicional para eso).
- **No reemplaza la cache del servidor** — no gestiona datos que vienen de la API. Para eso existe React Query en este proyecto.
- **No es un sistema de autenticación** — almacena los datos de sesión, pero la lógica de login vive en otro lugar.
- **No necesita `Provider`** a diferencia de Redux o Context — el store es global por diseño.

---

*En resumen: Zustand es el lugar donde vive el estado del cliente que varios componentes necesitan compartir — sencillo de crear, sencillo de leer, y sin la ceremonia de otras soluciones más pesadas.*
