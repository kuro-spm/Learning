# TypeScript

**¿Qué es?** TypeScript es JavaScript con tipado estático. Añade un sistema de tipos encima de JS que desaparece en tiempo de ejecución: el navegador nunca ve TypeScript, solo el JavaScript compilado.

---

## ¿Por qué existe?

JavaScript es dinámico por naturaleza: una variable puede ser un número ahora y una cadena un segundo después. Eso funciona en proyectos pequeños, pero escala muy mal.

Si vienes de C#/.NET, ya sabes cómo se siente trabajar con tipos: el compilador te avisa antes de que el error llegue a producción, el IDE te autocompleta con precisión, y refactorizar no es una apuesta. TypeScript trae exactamente esa experiencia al ecosistema JavaScript. Piénsalo como pasar de escribir Python sin anotaciones a escribir Python con `mypy` activado, pero integrado desde el primer momento.

Sin TypeScript, un error como pasar un `string` donde se espera un `number` solo aparece en tiempo de ejecución (o peor, silenciosamente). Con TypeScript, el compilador lo detecta antes de que ejecutes nada.

---

## ¿Cuándo y para qué se usa?

Aparece en la base de cualquier frontend React de cierto tamaño: cada componente, hook, servicio de API y modelo de datos se escribe en `.ts` o `.tsx`. El compilador de TypeScript (`tsc`) forma parte del proceso de build: si el código no compila, el build falla.

También actúa como contrato entre el frontend y el backend (en C#/.NET o cualquier otro lenguaje): los tipos que describes en TypeScript deben coincidir con los DTOs que expone la API, ya sea en una tienda online, un panel de administración o cualquier otra aplicación.

---

## Lo mínimo que necesitas saber

**1. Tipos básicos y anotaciones**
```ts
const productName: string = "Zapatillas running";
const taskCount: number = 42;
const isActive: boolean = true;
```

**2. Interfaces — equivalente a las clases DTO de C#**
```ts
interface Task {
  id: number;
  title: string;
  completed: boolean;
  assignedTo?: string; // opcional, como int? en C#
}
```

**3. Tipado de funciones**
```ts
function getTaskById(id: number): Task | null {
  // ...
}
```

**4. Generics — igual que en C#**
```ts
function fetchData<T>(url: string): Promise<T> {
  return fetch(url).then(res => res.json());
}
```

**5. Union types — cuando un valor puede ser más de un tipo**
```ts
type Status = "pending" | "in-progress" | "done";
```

---

## Lo que NO hace

- **No ejecuta tipos en el navegador.** En runtime no hay validación de tipos; TypeScript solo existe en desarrollo y build.
- **No reemplaza la validación de datos.** Si la API devuelve un campo inesperado, TypeScript no lo va a detectar en producción. Para eso existen librerías como `zod`.
- **No es un lenguaje diferente.** Todo JavaScript válido es TypeScript válido. No necesitas reaprender la sintaxis base.
- **No ralentiza la app.** El compilador genera JavaScript puro; el output final no incluye ningún overhead de tipos.

---

*En resumen: TypeScript te da el rigor de C# dentro del ecosistema JavaScript, detectando errores antes de ejecutar y haciendo que el IDE sea tu aliado, no un adivino.*
