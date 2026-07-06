# typescript-eslint

**¿Qué es?** Un conjunto de reglas y herramientas que permiten a ESLint entender y analizar código TypeScript, detectando errores y malos patrones que el compilador de TypeScript por sí solo no reporta.

---

## ¿Por qué existe?

ESLint es un linter para JavaScript, pero JavaScript no tiene tipos. Cuando el proyecto usa TypeScript, ESLint necesita un "traductor" que entienda la sintaxis de tipos, genéricos, decoradores y demás construcciones propias de TS.

**Analogía backend:** imagina que tienes Roslyn Analyzers en C# para detectar código problemático más allá de lo que el compilador reporta — como `CA2007` que te avisa de `await` sin `ConfigureAwait`. `typescript-eslint` hace exactamente eso en el mundo TypeScript: añade reglas de análisis estático que van más allá de la compilación.

Sin él, ESLint simplemente fallaría al leer archivos `.ts` o ignoraría construcciones como `interface`, `type` o `as`.

---

## ¿Cuándo y para qué se usa?

Se usa en cualquier frontend React escrito en TypeScript. `typescript-eslint` se configura en el archivo `eslint.config.js` (o `.eslintrc`) del proyecto y se ejecuta:

- Durante el desarrollo, integrado con el editor (VS Code muestra los errores en tiempo real).
- En el pipeline de CI, para bloquear merges con código que viola las reglas definidas.
- Al ejecutar `npm run lint` manualmente.

---

## Lo mínimo que necesitas saber

**1. Reglas que van más allá del compilador**

```ts
// typescript-eslint puede detectar esto como error:
// @typescript-eslint/no-floating-promises
async function cargarProductos() {
  fetchProductos(); // Promise ignorada — el compilador no se queja, el linter sí
}
```

**2. Reglas de tipado estricto**

```ts
// @typescript-eslint/no-explicit-any
function procesar(datos: any) { ... } // Advertencia: evita usar `any`
```

**3. Integración con el parser**

`typescript-eslint` incluye su propio parser (`@typescript-eslint/parser`) que reemplaza al parser por defecto de ESLint. Esto le permite leer la sintaxis TS correctamente.

**4. Reglas que requieren información de tipos**

Algunas reglas necesitan acceso al programa TypeScript completo (el `tsconfig.json`) para hacer análisis más profundo, como detectar promesas no manejadas o comparaciones innecesarias.

---

## Lo que NO hace

- **No compila TypeScript** — eso lo hace `tsc`. Este linter solo analiza el código estáticamente.
- **No formatea el código** — eso es responsabilidad de Prettier.
- **No reemplaza los errores del compilador** — los complementa. Si `tsc` falla, el linter es irrelevante.
- **No afecta al backend** — es una herramienta exclusiva del directorio `frontend/`.

---

*En resumen: `typescript-eslint` es el Roslyn Analyzer del mundo TypeScript — no compila ni formatea, pero detecta patrones problemáticos que el compilador deja pasar, manteniendo el código del frontend más seguro y consistente.*
