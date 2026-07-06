# eslint-plugin-react-hooks

Plugin de ESLint que detecta automáticamente el uso incorrecto de los Hooks de React en tiempo de escritura, antes de que el error llegue a ejecución.

---

## ¿Por qué existe?

Los Hooks de React (como `useState` o `useEffect`) tienen reglas estrictas: solo pueden llamarse en el nivel raíz del componente, nunca dentro de condicionales, bucles o funciones anidadas. Si las rompes, React pierde el rastro del estado interno y el comportamiento es impredecible.

**Analogía .NET:** imagina que tienes un método que solo puede llamarse desde el constructor de una clase, nunca desde un método privado ni dentro de un `if`. El compilador de C# te lo diría en tiempo de compilación. En JavaScript no hay compilador, así que este plugin hace ese trabajo: actúa como un analizador estático — similar a lo que hace Roslyn con sus `Diagnostic Analyzers` — pero para las reglas de Hooks.

---

## ¿Cuándo y para qué se usa?

Aparece en cualquier proyecto React que use Hooks, desde un formulario de registro sencillo hasta un panel de administración complejo. Se configura como parte de ESLint y se ejecuta automáticamente durante el desarrollo (en el editor) y en el pipeline de CI, impidiendo que código con Hooks mal usados llegue a la rama principal.

No requiere ninguna acción manual: si tu editor tiene ESLint activo, verás los errores subrayados directamente en el archivo `.tsx`.

---

## Lo mínimo que necesitas saber

**Regla 1 — `rules-of-hooks`:** los Hooks solo van en el nivel raíz del componente.

```tsx
// MAL — Hook dentro de un condicional
function MyComponent({ isReady }: { isReady: boolean }) {
  if (isReady) {
    const [count, setCount] = useState(0); // Error: Hook condicional
  }
}

// BIEN
function MyComponent({ isReady }: { isReady: boolean }) {
  const [count, setCount] = useState(0);
  if (!isReady) return null;
}
```

**Regla 2 — `exhaustive-deps`:** el array de dependencias de `useEffect` debe incluir todas las variables externas que usa.

```tsx
// MAL — falta userId en dependencias
useEffect(() => {
  fetchUser(userId);
}, []); // Advertencia: userId no declarado como dependencia

// BIEN
useEffect(() => {
  fetchUser(userId);
}, [userId]);
```

**Configuración habitual:** muchos proyectos activan estas reglas como `error` (no como `warn`), lo que bloquea la compilación si se incumplen.

---

## Lo que NO hace

- No corrige los errores automáticamente (no es un formatter como Prettier).
- No valida la lógica de negocio dentro de los Hooks, solo su estructura.
- No afecta al código C#/.NET del backend en absoluto.
- No es un sustituto de entender cómo funcionan los Hooks; solo avisa cuando las reglas se rompen.

---

*En resumen: `eslint-plugin-react-hooks` es el guardián estático que evita los errores más comunes con Hooks de React, actuando como un analizador en tiempo de desarrollo para que los problemas aparezcan en tu editor y no en producción.*
