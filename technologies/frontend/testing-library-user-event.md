# @testing-library/user-event

**¿Qué es?**
Una librería de testing para React que simula interacciones reales del usuario (clics, escritura, tabulación) de forma fiel a cómo ocurren en el navegador real.

---

## ¿Por qué existe?

El problema: cuando un usuario escribe en un input o hace clic en un botón, el navegador dispara una cadena de eventos (`keydown`, `keypress`, `input`, `keyup`, `change`, `blur`...). La alternativa más básica, `fireEvent` de Testing Library, dispara solo un evento aislado, lo que puede hacer que tu test pase aunque la UI real falle.

**Analogía backend:** es como la diferencia entre llamar directamente a un método interno de un servicio C# en un test unitario, versus hacer una petición HTTP real en un test de integración. `fireEvent` es el método directo; `userEvent` es la petición real.

`@testing-library/user-event` orquesta toda esa cadena de eventos igual que lo haría un navegador, haciendo los tests mucho más fiables.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, los tests de componentes React usan `userEvent` para validar flujos de usuario reales: rellenar formularios de proyectos, hacer clic en botones de acción, navegar entre campos con Tab, o interactuar con selects y checkboxes. Se usa junto a `@testing-library/react` y `vitest` como entorno de test.

---

## Lo mínimo que necesitas saber

**1. Siempre se inicializa con `setup()`**
```tsx
import userEvent from '@testing-library/user-event'

const user = userEvent.setup()
```

**2. Todo es `async/await`**
```tsx
it('permite escribir en el campo nombre', async () => {
  const user = userEvent.setup()
  render(<ProjectForm />)

  await user.type(screen.getByLabelText('Nombre'), 'EcoWave Q3')
  expect(screen.getByLabelText('Nombre')).toHaveValue('EcoWave Q3')
})
```

**3. `click`, `type`, `clear`, `tab` son los métodos más usados**
```tsx
await user.click(screen.getByRole('button', { name: 'Guardar' }))
await user.clear(screen.getByLabelText('Descripción'))
await user.tab() // simula navegar con teclado
```

**4. `type` simula tecla a tecla, `keyboard` es para secuencias especiales**
```tsx
await user.keyboard('{Enter}')
await user.keyboard('{Control>}a{/Control}') // Ctrl+A
```

---

## Lo que NO hace

- **No ejecuta en un navegador real** — sigue siendo un entorno simulado (jsdom). Para tests E2E reales, se usa Playwright o Cypress.
- **No reemplaza los tests de integración con el backend** — solo valida la UI en aislamiento.
- **No valida estilos visuales** — si un botón está oculto con CSS pero existe en el DOM, `userEvent` puede igualmente interactuar con él.

---

*En resumen: `@testing-library/user-event` hace que tus tests de componentes React se comporten como un usuario real, disparando la cadena completa de eventos del navegador y dando confianza real en que la UI funciona.*
