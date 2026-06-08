# ESLint

**¿Qué es?** ESLint es una herramienta de análisis estático para JavaScript y TypeScript que revisa tu código en busca de errores, malas prácticas y violaciones de estilo antes de que lo ejecutes.

---

## ¿Por qué existe?

JavaScript y TypeScript no tienen un compilador tan estricto como C#. En .NET, si escribes algo incorrecto, el compilador te lo dice en el acto. En JS/TS puro, muchos errores solo aparecen en tiempo de ejecución o directamente en producción.

ESLint actúa como ese compilador estricto que JavaScript no trae de serie. Si en C# usas Roslyn Analyzers o StyleCop para forzar convenciones de código en el equipo, ESLint cumple ese mismo rol en el mundo JS/TS: define reglas, las aplica de forma automática y avisa (o incluso corrige) cuando el código no las respeta.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, ESLint se usa en la capa frontend (React + TypeScript). Está configurado para ejecutarse:

- **Durante el desarrollo**: el editor muestra los errores en tiempo real (subrayados rojos/amarillos).
- **En el pipeline de CI**: si algún archivo viola las reglas, el build falla antes de llegar a producción.

La configuración vive en el archivo `eslint.config.js` (o `.eslintrc`) en la raíz del proyecto frontend.

---

## Lo mínimo que necesitas saber

**1. Las reglas son configurables.** ESLint no impone nada por defecto; todo viene de conjuntos de reglas (plugins) que el proyecto activa.

**2. Dos niveles de severidad: `warn` y `error`.**

```js
// eslint.config.js
rules: {
  "no-unused-vars": "warn",   // avisa pero no rompe el build
  "no-console": "error"       // rompe el build si dejas un console.log
}
```

**3. Puedes ignorar una línea puntualmente** (úsalo con criterio):

```ts
// eslint-disable-next-line no-console
console.log("debug temporal");
```

**4. Ejecutarlo desde terminal:**

```bash
npx eslint src/            # analiza toda la carpeta src
npx eslint src/ --fix      # corrige automáticamente lo que pueda
```

**5. Integración con el editor:** con la extensión ESLint de VS Code, los errores aparecen inline mientras escribes, igual que los squiggles de Intellisense en Visual Studio.

---

## Lo que NO hace

- **No formatea el código** — eso es trabajo de Prettier (otra herramienta del proyecto). ESLint detecta problemas de lógica y convenciones, no de indentación o comillas.
- **No compila TypeScript** — solo lee el AST del código; el compilador `tsc` es quien verifica los tipos en profundidad.
- **No reemplaza las pruebas** — analiza el código estáticamente, no lo ejecuta.

---

*En resumen: ESLint es el "compilador de buenas prácticas" del frontend; atrapa errores y hace cumplir las convenciones del equipo mucho antes de que el código llegue al navegador.*
