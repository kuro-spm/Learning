# Playwright para documentación visual

## ¿Qué es?

Es el mismo Playwright de siempre ([ver la ficha general](Playwright.md)), pero puesto a un uso distinto: en vez de comprobar que la aplicación funciona, se usa para **recorrerla de verdad y dejar constancia visual** de cómo se usa — las capturas de pantalla de un manual de usuario, de una guía de onboarding o de la documentación de una API con interfaz web.

## ¿Por qué existe?

Las capturas de un manual casi siempre se hacen a mano: alguien abre la aplicación, navega paso a paso y va haciendo pantallazos. Funciona la primera vez, pero en cuanto la interfaz cambia un poco — un botón que se mueve, un texto que se retoca — las capturas viejas mienten, y nadie se acuerda de rehacerlas todas. Además, repetir ese recorrido a mano para cada nueva versión es trabajo manual puro, sin ninguna garantía de que el camino que se siguió la vez anterior fuera exactamente el mismo.

Un script de Playwright que hace el mismo recorrido es reproducible: cada vez que se lanza, pasa por los mismos pasos, en el mismo orden, y deja las capturas con los nombres que tú decidas. Si la interfaz cambia, el script falla señalando exactamente dónde — no se entera nadie seis meses después de que una captura del manual mostraba una pantalla que ya no existe.

> Si ya conoces los tests E2E, piensa en esto como un test al que no le importa si algo "pasa" o "falla" en sentido estricto: su entregable no es un ✅ verde, son los ficheros de imagen que deja por el camino.

## Locators que no se rompen con un cambio de idioma

Un *locator* escrito contra el **texto visible** de un botón ("Entrar", "Añadir al carrito") es frágil de una forma concreta que conviene conocer: si la aplicación soporta varios idiomas y el *Page Object* se escribió mirando la interfaz en uno de ellos, ese locator **deja de encontrar el botón** en cuanto la interfaz cambia de idioma — y no lo avisa con un error claro de inmediato: Playwright sigue intentando localizar el elemento durante todo el timeout de la aserción, así que el fallo tarda en aparecer y el mensaje de error solo dice "no se encontró", sin pista de que la causa real es el idioma.

Imagina una aplicación de tareas con selector ES/CA/EN. Un *Page Object* de la pantalla de login, escrito mirando la interfaz en catalán:

```ts
// LoginPage.ts — escrito contra la interfaz en CATALÁN
export class LoginPage {
  constructor(private readonly page: Page) {}

  async iniciarSesion(email: string, password: string) {
    await this.page.getByLabel('Correu electrònic').fill(email);
    await this.page.getByLabel('Contrasenya').fill(password);
    await this.page.getByRole('button', { name: 'Entra' }).click(); // 'Entra', no 'Entrar'
  }
}
```

Si un segundo script fija la interfaz en castellano (`localStorage.setItem('idioma', 'es')`) y reutiliza esta misma clase, el botón se llama ahora "Iniciar sesión", no "Entra" — el locator no encuentra nada, y el fallo llega varios pasos después, en una aserción que a simple vista no tiene nada que ver con el idioma.

La lección no es "no uses Page Objects": es que un Page Object **hereda el idioma con el que se escribió**, y si vas a automatizar la misma pantalla en un idioma distinto del que ya tienes cubierto, necesitas o bien una variante del Page Object en ese idioma, o bien locators que no dependan del texto (un `id` o un `data-testid` estable, que no cambian con el idioma):

```ts
// Esto sobrevive al cambio de idioma: el id no se traduce
await page.locator('#login-email').fill(email);

// Esto no: "Iniciar sesión" solo existe en castellano
await page.getByRole('button', { name: 'Iniciar sesión' }).click();
```

Antes de reutilizar un Page Object ajeno en un recorrido con el idioma cambiado, comprueba sus locators: si dependen de texto visible, vas a tener que reescribirlos para el idioma nuevo.

## Esperar hechos, no plazos

Una aserción web-first (`expect(locator).toBeVisible()`) reintenta hasta que algo aparece en pantalla, pero no dice **por qué** tarda. Cuando el paso que esperas depende de una petición de red concreta — por ejemplo, confirmar que un alta se guardó antes de seguir — es mejor anclar la espera a esa petición y a su código de estado, no solo a que algo cambie visualmente:

```ts
const guardado = page.waitForResponse(
  (r) => r.url().endsWith('/api/tasks') && r.request().method() === 'POST',
);
await page.getByRole('button', { name: 'Guardar tarea' }).click();
const respuesta = await guardado;
expect(respuesta.status()).toBe(201); // si el servidor rechazó el alta, el test falla AQUÍ
await expect(page.getByText('Tarea creada')).toBeVisible();
```

Si en vez de esto te limitas a esperar `toBeVisible()` sobre el mensaje de confirmación y el servidor responde con un error, el fallo aparece como "no encuentro el texto 'Tarea creada'" — cierto, pero no dice que el problema real fue un 500 del backend. Anclar la espera a la petición y comprobar su `status()` hace que el fallo señale la causa, no el síntoma.

Esto importa el doble cuando el paso que esperas es **lento o caro** — una llamada a un proveedor externo que tarda segundos en responder de verdad, por ejemplo. Un timeout corto pensado para una interacción instantánea hace que el test falle por impaciencia, no porque nada vaya mal; ahí la espera debe ser generosa (decenas de segundos, no los 5 segundos por defecto) y, otra vez, anclada a la respuesta real:

```ts
await boton.click();
// Una llamada real a un proveedor externo puede tardar bastante más que una local:
await expect(resultado).toBeVisible({ timeout: 120_000 });
```

## Tráfico simulado o real: una decisión consciente

Playwright puede interceptar cualquier petición con `page.route()` y devolver una respuesta inventada, sin que llegue a tocar el backend de verdad:

```ts
await page.route('**/api/images/generate', (route) =>
  route.fulfill({ status: 202, json: { id: 'img-1', estado: 'EN_CURSO' } }),
);
```

Para la mayoría de los tests esto es lo correcto: son rápidos, deterministas y no dependen de que un servicio externo esté disponible. Pero cuando el objetivo **es** comprobar la integración real — o, como en la documentación visual, enseñar de verdad lo que la aplicación hace — hay que dejar pasar el tráfico real, sin ningún `page.route()` de por medio.

La decisión tiene un matiz importante si el servicio real del otro lado **cobra por petición** (un proveedor de generación de imágenes por IA, un envío de SMS, una pasarela de pago en modo real): cada vez que el script corre sin mocks, genera un gasto de verdad. Antes de lanzar un recorrido así conviene tenerlo clarísimo y, si el script falla a mitad, **no** relanzarlo entero sin pensar — es dinero repetido por el mismo tramo que ya funcionó. La sección de pasos nombrados, más abajo, es la forma de poder reanudar solo lo que falta.

## Automatizar botones con estado sin perder el control

Un botón de tipo interruptor (`aria-pressed="true"/"false"`, un favorito, un "modo oscuro") cambia de estado cada vez que se pulsa. Un script que lo pulsa **sin comprobar antes en qué estado está** puede acabar haciendo justo lo contrario de lo que pretendía — si el interruptor ya estaba activado, pulsarlo lo **desactiva**:

```ts
// MAL: asume que está apagado, sin comprobarlo
await page.getByRole('button', { name: 'Modo oscuro' }).click();

// BIEN: comprueba el estado real antes de decidir si hace falta pulsar
const modoOscuro = page.getByRole('button', { name: 'Modo oscuro' });
const activado = await modoOscuro.getAttribute('aria-pressed');
if (activado !== 'true') await modoOscuro.click();
```

Esto se vuelve crítico en dos situaciones concretas. La primera: cuando un interruptor viene **activado por defecto** (no todos arrancan en "apagado" — comprueba el valor inicial real en vez de asumirlo, por ejemplo abriendo la aplicación a mano una vez antes de automatizarla). La segunda: cuando el script se **reanuda a mitad de camino** — si una ejecución anterior ya dejó el interruptor en el estado que quieres y la reanudación vuelve a pulsarlo "para asegurarse", lo deja en el estado contrario sin que nada avise del error hasta un paso bastante más adelante, cuando algo que dependía de ese estado deja de estar disponible.

## Capturar y organizar las pantallas

`page.screenshot()` captura toda la página visible; un *locator* puede capturarse solo a él, útil para un detalle concreto (un icono, un formulario pequeño) sin el ruido del resto de la pantalla:

```ts
import { mkdirSync } from 'node:fs';
import path from 'node:path';

const CARPETA_CAPTURAS = path.join('capturas-manual');

async function capturar(page: Page, nombre: string) {
  mkdirSync(CARPETA_CAPTURAS, { recursive: true });
  await page.screenshot({ path: path.join(CARPETA_CAPTURAS, `${nombre}.jpg`), type: 'jpeg', quality: 90 });
}

// Solo el icono de notificaciones, no la página entera:
await page.getByRole('button', { name: 'Notificaciones' }).screenshot({ path: 'icono-notificaciones.png' });
```

`type: 'jpeg'` con una `quality` moderada da ficheros bastante más ligeros que el PNG por defecto, buena opción cuando las capturas van a vivir dentro de un repositorio o de un documento.

Un matiz de entorno si el proyecto está configurado como **módulos ES** (`"type": "module"` en el `package.json`, o un `tsconfig` con `module` en `ESNext`/`NodeNext`): la variable `__dirname`, disponible siempre en CommonJS para saber en qué carpeta vive el fichero actual, **no existe** en ESM y lanza `ReferenceError: __dirname is not defined`. La forma portable de recuperar ese dato en ESM es a partir de `import.meta.url`:

```ts
import { fileURLToPath } from 'node:url';
import path from 'node:path';

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const CARPETA_CAPTURAS = path.resolve(__dirname, '..', 'capturas-manual');
```

Si el proyecto ya tiene la convención de invocar sus scripts siempre desde el mismo directorio (por ejemplo, la raíz del paquete), una alternativa más simple es usar una ruta **relativa al directorio desde el que se lanza el comando** en vez de resolverla contra la ubicación del propio fichero — evita el problema de raíz sin necesitar `import.meta.url`, a costa de depender de que el comando siempre se invoque desde el mismo sitio.

## Recorridos largos: pasos nombrados y sin reintentos automáticos

Un recorrido de documentación visual suele encadenar muchas acciones en un único test. Envolver cada tramo en `test.step()` no cambia el comportamiento, pero sí lo que cuenta el informe cuando algo falla: en vez de un rastro de líneas de código, el informe dice en qué **paso con nombre** se rompió.

```ts
test('recorrido del manual de usuario', async ({ page }) => {
  await test.step('Iniciar sesión', async () => {
    await page.goto('/login');
    // …
  });

  await test.step('Crear una tarea nueva', async () => {
    await page.getByRole('button', { name: 'Nueva tarea' }).click();
    // …
  });

  await test.step('Generar la miniatura con el proveedor de IA (llamada real)', async () => {
    await page.getByRole('button', { name: 'Generar' }).click();
    await expect(page.getByRole('img', { name: 'Miniatura generada' })).toBeVisible({ timeout: 120_000 });
  });
});
```

Cuando alguno de esos pasos dispara una acción **cara o no-idempotente** (una llamada real de pago, un alta que no se puede repetir sin duplicar datos), hay que desactivar los reintentos automáticos para ese test (`retries: 0` en la configuración, o a nivel de proyecto) en vez de confiar en el comportamiento por defecto de la suite. Un reintento automático que repite sin preguntar el paso que ya costó dinero la primera vez es exactamente el tipo de sorpresa que conviene evitar por diseño, no descubrir en la factura.

## Buenas prácticas avanzadas

- **Nunca asumas el estado inicial de un control con memoria** (un interruptor, una preferencia persistida en `localStorage` o `sessionStorage`): compruébalo en vivo antes de decidir si hace falta tocarlo. Un valor por defecto que cambia entre versiones de la aplicación rompe en silencio cualquier script que lo diera por sentado.
- **Separa claramente, en el propio código, qué pasos usan tráfico real y cuáles están mockeados** — un comentario visible encima de cada `page.route()` (o de su ausencia deliberada) evita que alguien reactive sin querer una llamada de pago al copiar un bloque de otro test.
- **Diseña la reanudación antes de necesitarla** — si un recorrido largo con pasos caros puede fallar a mitad, vale la pena dejar preparado desde el principio un punto de entrada que salte directamente a mitad del recorrido (una variable de entorno que indique "parte de aquí"), en vez de improvisarlo con el disgusto ya encima.
- **Verifica el estado real contra la base de datos o la API cuando un paso falle a mitad**, no solo contra lo que muestra la pantalla — un botón con estado optimista puede pintar un cambio antes de que el servidor lo confirme, y confiar solo en el DOM puede hacer que dos ejecuciones sucesivas se pisen sin que lo notes hasta mucho después.

## Documentación oficial

- [Locators de Playwright](https://playwright.dev/docs/locators) — la referencia completa de `getByRole`, `getByLabel` y el resto de estrategias de localización, con la guía oficial de cuál preferir en cada caso.
- [Test Fixtures](https://playwright.dev/docs/test-fixtures) y [`test.step()`](https://playwright.dev/docs/api/class-test#test-step) — la API completa para organizar fixtures propias y pasos con nombre.

## Recursos didácticos

- [ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/) — el catálogo de roles y nombres accesibles (`aria-pressed`, `aria-label`...) que hace posible localizar elementos "como los vería un lector de pantalla"; entender esto de verdad hace mejores locators.

---

*En resumen: el mismo Playwright que prueba que una aplicación funciona puede recorrerla de verdad y dejar constancia — pero un recorrido así vive y muere por los mismos detalles que un test de corrección: locators que sobreviven al idioma, esperas ancladas a hechos, y mucho cuidado con repetir lo que ya costó caro.*
