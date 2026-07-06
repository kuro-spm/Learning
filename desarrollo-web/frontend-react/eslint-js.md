# @eslint/js

**¿Qué es?** Es el paquete oficial que provee las reglas base de ESLint para JavaScript. Actúa como punto de partida para configurar qué patrones de código son válidos y cuáles deben marcarse como error o advertencia.

---

## ¿Por qué existe?

JavaScript no tiene un compilador estricto que rechace código mal escrito antes de ejecutarlo. A diferencia de C#, donde el compilador de Roslyn te impide compilar si usas una variable no declarada o llamas a un método inexistente, en JavaScript esos errores solo aparecen en tiempo de ejecución.

ESLint es el equivalente a tener un analizador estático similar a lo que hace el compilador de C# o herramientas como SonarQube en el mundo .NET: revisa el código antes de que se ejecute y señala problemas. `@eslint/js` es el paquete que aporta el conjunto de reglas estándar sobre las que se construye esa validación.

---

## ¿Cuándo y para qué se usa?

Aparece en cualquier proyecto frontend en React o JavaScript que quiera detectar errores antes de ejecutar el código: una tienda online, un panel de administración o una app de gestión de tareas. Se referencia en el archivo `eslint.config.js` (o `eslint.config.mjs`) en la raíz del proyecto y define las reglas mínimas que se aplican a todos los archivos `.js` y `.jsx`.

```js
// eslint.config.js
import js from "@eslint/js";

export default [
  js.configs.recommended, // activa las reglas recomendadas de JS base
  {
    rules: {
      "no-unused-vars": "warn",
      "no-console": "warn",
    },
  },
];
```

ESLint se ejecuta en dos momentos: durante el desarrollo (el editor muestra los errores en tiempo real) y como paso de validación antes del build o en el pipeline de CI.

---

## Lo mínimo que necesitas saber

**1. `js.configs.recommended`**
Es un preset con las reglas más útiles ya activadas. Equivale a heredar de una clase base con comportamientos sensatos.

**2. Las reglas tienen tres niveles de severidad**

```js
"no-unused-vars": "off"   // desactivada
"no-unused-vars": "warn"  // aviso, no bloquea
"no-unused-vars": "error" // bloquea el build / falla en CI
```

**3. Puedes sobreescribir reglas por archivo o carpeta**

```js
{
  files: ["**/*.test.js"],
  rules: {
    "no-console": "off", // permitido en tests
  },
}
```

**4. Se ejecuta desde la terminal**

```bash
npx eslint src/
```

---

## Lo que NO hace

- No formatea el código (eso lo hace **Prettier**).
- No valida tipos de TypeScript (eso requiere `typescript-eslint`).
- No es un reemplazo del compilador: no verifica que las importaciones resuelvan a archivos reales.
- No corrige errores de logica de negocio, solo de forma y patrones sintácticos.

---

*En resumen: `@eslint/js` es el guardián de calidad de código JavaScript del frontend — el equivalente al compilador de C# en cuanto a detección temprana de problemas, pero configurable y ejecutado como herramienta externa.*
