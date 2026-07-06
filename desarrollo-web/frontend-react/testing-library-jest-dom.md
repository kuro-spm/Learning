# @testing-library/jest-dom

## ¿Qué es?

Una librería que añade matchers personalizados a Jest para hacer afirmaciones sobre el estado del DOM de forma legible y expresiva.

---

## ¿Por qué existe?

Jest, por defecto, sabe comparar valores: números, strings, objetos. Pero cuando testeas una interfaz web necesitas preguntar cosas como "¿este botón está desactivado?" o "¿este elemento tiene este texto visible?". Sin `jest-dom`, tendrías que escribir assertions torpes accediendo a propiedades del DOM manualmente.

**Analogía backend:** Es como la diferencia entre escribir `Assert.AreEqual(true, user.IsActive)` versus usar FluentAssertions y escribir `user.IsActive.Should().BeTrue("because the user just registered")`. Ambos funcionan, pero uno comunica la intención con mucha más claridad.

`jest-dom` hace exactamente eso: transforma assertions crípticas sobre el DOM en frases casi legibles en lenguaje natural.

---

## ¿Cuándo y para qué se usa?

Se usa junto con `@testing-library/react` en cualquier suite de tests de componentes: verificar que un campo de un formulario de registro muestra un mensaje de error, que un botón de pago está deshabilitado hasta que se completan los datos, o que un badge de estado en un panel de administración muestra el texto correcto.

Se importa una sola vez en el archivo de setup de Jest (`setupFilesAfterFramework`) y queda disponible en todos los archivos de test del frontend.

---

## Lo mínimo que necesitas saber

**1. Se importa una vez, se usa en todas partes**

```ts
// jest.setup.ts
import '@testing-library/jest-dom';
```

**2. Matchers de visibilidad y presencia**

```tsx
expect(screen.getByText('Pedido entregado')).toBeInTheDocument();
expect(screen.queryByText('Error')).not.toBeInTheDocument();
expect(screen.getByRole('alert')).toBeVisible();
```

**3. Matchers de estado de elementos de formulario**

```tsx
expect(screen.getByRole('button', { name: 'Guardar' })).toBeDisabled();
expect(screen.getByLabelText('Email')).toBeRequired();
expect(screen.getByLabelText('Nombre')).toHaveValue('Ana');
```

**4. Matchers de estilos y clases CSS**

```tsx
expect(screen.getByTestId('status-badge')).toHaveClass('badge--active');
expect(screen.getByTestId('panel')).toHaveStyle({ display: 'none' });
```

**5. Matchers de accesibilidad semántica**

```tsx
expect(screen.getByRole('checkbox', { name: 'Activo' })).toBeChecked();
```

---

## Lo que NO hace

- **No renderiza componentes** — eso lo hace `@testing-library/react`.
- **No selecciona elementos del DOM** — eso lo hacen `screen.getBy*`, `screen.queryBy*`, etc.
- **No ejecuta tests** — eso es Jest.
- **No reemplaza tests end-to-end** — solo opera sobre el DOM renderizado en memoria, no en un navegador real.

---

*En resumen: `jest-dom` es la capa de lenguaje que convierte las comprobaciones sobre el DOM en assertions claras y mantenibles, sin añadir complejidad al setup ni a la ejecución de los tests.*
