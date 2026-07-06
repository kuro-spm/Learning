# Vitest

**¿Qué es?** Vitest es un framework de tests unitarios diseñado específicamente para proyectos que usan Vite. Ejecuta tests en el mismo entorno que el bundler, lo que lo hace extremadamente rápido y sin configuración extra.

---

## ¿Por qué existe?

Jest era el estándar para tests en JavaScript, pero con la llegada de Vite surgió un problema: Jest usa su propio sistema de transformación de módulos, incompatible con la configuración de Vite. Cada test requería configurar un pipeline paralelo, con transformadores duplicados y constantes conflictos.

**Analogía backend:** imagina que en tu proyecto .NET tienes xUnit funcionando bien, pero de repente introduces un nuevo sistema de build que no entiende los assemblies que xUnit necesita. Tendrías que mantener dos pipelines de compilación distintos. Eso era Jest con Vite. Vitest resuelve esto reutilizando directamente la configuración de Vite, sin duplicaciones.

---

## ¿Cuándo y para qué se usa?

Aparece en la capa frontend (React) de cualquier proyecto para testear:

- Lógica de transformación de datos antes de enviarlos al backend (en C#/.NET o cualquier otro lenguaje).
- Hooks personalizados que gestionan estado local.
- Funciones utilitarias (formateo de fechas, cálculos, validaciones).

Los archivos de test viven junto al código que prueban, con la convención `*.test.ts` o `*.spec.tsx`.

---

## Lo mínimo que necesitas saber

**1. Estructura de un test**

```ts
import { describe, it, expect } from 'vitest'
import { calcularProgreso } from './proyecto.utils'

describe('calcularProgreso', () => {
  it('devuelve 50 cuando la mitad de tareas están completadas', () => {
    expect(calcularProgreso(4, 2)).toBe(50)
  })
})
```

**2. Matchers comunes** — funcionan igual que en xUnit pero con sintaxis de cadena:

```ts
expect(valor).toBe(42)           // igualdad estricta (como Assert.Equal)
expect(lista).toHaveLength(3)    // longitud
expect(fn).toThrow('error')      // lanza excepción
```

**3. Ejecutar los tests**

```bash
npx vitest          # modo watch (re-ejecuta al guardar)
npx vitest run      # una sola pasada (útil en CI)
```

**4. Mocks** — sustituye dependencias externas igual que Moq en .NET:

```ts
import { vi } from 'vitest'

const mockFetch = vi.fn().mockResolvedValue({ data: [] })
```

---

## Lo que NO hace

- **No testea componentes renderizados visualmente** — para eso se usa React Testing Library junto a Vitest, pero son herramientas distintas.
- **No cubre tests end-to-end** — eso es responsabilidad de Playwright o Cypress.
- **No reemplaza los tests del backend** — los tests de la API en C#/.NET siguen siendo xUnit; Vitest solo cubre el frontend.

---

*En resumen: Vitest es el xUnit del frontend Vite — misma filosofía de tests unitarios que ya conoces, pero adaptada al ecosistema JavaScript.*
