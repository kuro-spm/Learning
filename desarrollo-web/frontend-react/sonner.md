# Sonner

**¿Qué es?** Sonner es una librería de React que muestra notificaciones tipo "toast" (mensajes emergentes temporales) con una API mínima y un diseño moderno listo para usar.

---

## ¿Por qué existe?

En backend C#/.NET estás acostumbrada a lanzar excepciones o retornar `Result<T>` para comunicar el estado de una operación. En el frontend se necesita el equivalente visual: informar al usuario de que algo ocurrió (éxito, error, advertencia) sin bloquear la interfaz con un modal o una alerta nativa del navegador.

Antes de librerías como Sonner, cada equipo montaba su propia solución: componentes de alerta en el estado global, combinaciones de `useState` + `useEffect` para auto-cerrar mensajes, estilos manuales. Sonner elimina todo ese boilerplate con una llamada de una sola línea.

---

## ¿Cuándo y para qué se usa?

Aparece en cualquier interfaz que necesite dar feedback inmediato al usuario tras una acción sin interrumpir su flujo: una tienda online que confirma que un pedido se creó, una app de tareas que avisa de que una tarea cambió de estado, o cualquier pantalla que deba informar de que una petición a la API ha fallado. El `<Toaster />` se monta una sola vez en el componente raíz de la aplicación, y desde cualquier punto de la app se disparan toasts con la función `toast()`.

---

## Lo mínimo que necesitas saber

**1. Montar el componente una vez en la raíz**

```tsx
// App.tsx
import { Toaster } from 'sonner';

export default function App() {
  return (
    <>
      <Toaster position="top-right" />
      <RouterOutlet />
    </>
  );
}
```

**2. Disparar un toast desde cualquier lugar**

```tsx
import { toast } from 'sonner';

// Éxito
toast.success('Pedido creado correctamente');

// Error
toast.error('No se pudo conectar con el servidor');

// Informativo
toast('Cambios guardados');
```

**3. Toast con promesa (ideal para llamadas a la API)**

```tsx
toast.promise(crearPedido(datos), {
  loading: 'Creando pedido...',
  success: 'Pedido creado',
  error: 'Error al crear el pedido',
});
```

**4. Versión usada:** `1.5.0`. No requiere configuración de store ni contexto adicional.

---

## Lo que NO hace

- **No gestiona estado global** — no es un reemplazo de Redux ni de Zustand; solo muestra mensajes visuales.
- **No valida datos** — la lógica de negocio sigue viviendo en tu servicio o en el backend.
- **No persiste mensajes** — si el usuario navega, el toast desaparece. Para errores críticos usa un componente de alerta permanente.
- **No reemplaza los modales de confirmación** — para acciones destructivas (borrar un pedido), sigue usando un diálogo explícito.

---

*En resumen: Sonner es el `Console.WriteLine` del frontend, pero bonito, temporal y sin interrumpir el flujo del usuario.*
