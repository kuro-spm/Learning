# Playwright

## ¿Qué es?

Playwright es una herramienta de tests **end-to-end (E2E)**: automatiza un navegador real (Chromium, Firefox o WebKit) para probar tu aplicación web de punta a punta, haciendo lo que haría una persona — abrir una página, escribir en un formulario, pulsar un botón y comprobar lo que aparece en pantalla.

## ¿Por qué existe?

Un test unitario prueba una función aislada; un test de integración prueba varias piezas del backend juntas. Pero ninguno de los dos abre el navegador, así que ninguno detecta que el botón de "Entrar" está tapado por un banner, que la ruta de "home" no carga tras el login, o que el frontend y el backend se han puesto de acuerdo en un formato de fecha distinto. Esos fallos solo aparecen cuando **todo el sistema corre junto y se maneja como lo haría un usuario**.

Antes existían herramientas como Selenium, pero eran frágiles: el test hacía clic *antes* de que el elemento estuviera listo y fallaba de forma intermitente (los temidos tests *flaky*). Playwright nace para resolver eso con **auto-waiting**: espera automáticamente a que el elemento sea visible, esté habilitado y sea interactuable antes de actuar, sin que tú escribas ni una sola pausa.

> Si vienes de los tests de API con `WebApplicationFactory`, que atacan la aplicación por HTTP, piensa en Playwright como el escalón de arriba: ataca la aplicación por el navegador, viendo también el HTML, el CSS y el JavaScript que el usuario tiene delante.

## ¿Cuándo y para qué se usa?

Playwright ocupa la **cima de la pirámide de tests** (ver [Tipos de tests](../testing-dotnet/Tipos-de-tests.md)): por encima de los unitarios (rápidos y numerosos, con [Vitest](../../desarrollo-web/frontend-react/Vitest.md) en el frontend) y de los de integración (piezas reales juntas, con [Testcontainers](../testing-dotnet/Testcontainers.md) dando una base de datos de verdad). Son pocos, lentos y caros de mantener, pero son los únicos que prueban el sistema **entero**.

Se reservan para los **flujos críticos** (los *golden paths*) de una aplicación: en una tienda online, el recorrido "buscar producto → añadir al carrito → pagar"; en cualquier aplicación con usuarios, la **autenticación** — iniciar sesión, llegar a la home y cerrar sesión, más los casos de error (credenciales inválidas, sesión caducada o token invalidado). Es un escenario ideal para el primer test E2E porque cruza todo el sistema: el formulario en un frontend React + Vite + TypeScript, la validación de credenciales en un backend .NET y la persistencia de la sesión.

No pruebes *todo* con E2E: es lento y frágil. Cubre el puñado de caminos que, si se rompen, dejan la aplicación inservible.

## Lo mínimo que necesitas saber

**1. Un test es un flujo de usuario**

Un test abre una página, interactúa y comprueba el resultado. `page` es la pestaña del navegador:

```ts
import { test, expect } from '@playwright/test';

test('el login lleva a la home', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('ana@example.com');
  await page.getByLabel('Contraseña').fill('secreto123');
  await page.getByRole('button', { name: 'Entrar' }).click();

  await expect(page).toHaveURL('/home');
  await expect(page.getByRole('heading', { name: 'Bienvenida' })).toBeVisible();
});
```

**2. Locators: apunta a lo que ve el usuario, no al HTML**

Un *locator* describe cómo encontrar un elemento. Los recomendados imitan cómo una persona (o un lector de pantalla) percibe la página, no su estructura interna:

```ts
page.getByRole('button', { name: 'Entrar' }); // por su rol accesible y su texto
page.getByLabel('Email');                      // el input asociado a esa etiqueta
page.getByText('Credenciales inválidas');      // por el texto visible
page.getByTestId('user-menu');                 // último recurso: un data-testid explícito
```

Evita selectores CSS frágiles como `.btn-primary > span:nth-child(2)`: cualquier retoque de maquetación los rompe.

**3. Auto-waiting y aserciones web-first**

No escribas esperas manuales. Las aserciones de Playwright (`expect`) **reintentan automáticamente** hasta que la condición se cumple o salta el timeout, absorbiendo la asincronía de la red y del renderizado:

```ts
// Espera (reintentando) a que el mensaje aparezca; no hace falta ningún sleep
await expect(page.getByText('Credenciales inválidas')).toBeVisible();
await expect(page.getByRole('button', { name: 'Entrar' })).toBeEnabled();
```

**4. Headless vs headed**

Por defecto el navegador corre en modo **headless** (sin ventana): más rápido, y es como se ejecuta en CI. En modo **headed** (`--headed`) ves la ventana abrirse y actuar sola, útil para depurar. El modo interactivo `--ui` da un panel con la línea de tiempo de cada paso:

```bash
npx playwright test                 # headless, todos los navegadores
npx playwright test --headed        # ves el navegador actuar
npx playwright test --ui            # modo interactivo para depurar
npx playwright codegen localhost:5173  # graba tus clics y genera el test
```

**5. Ejecución multi-navegador**

En `playwright.config.ts` declaras contra qué navegadores correr. Cada `project` ejecuta toda la suite en un motor distinto, así cazas el bug que solo se da en Safari (WebKit):

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  use: {
    baseURL: 'http://localhost:5173', // hace que page.goto('/login') sea relativo
    trace: 'on-first-retry',          // guarda traza si un test falla y se reintenta
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit',   use: { ...devices['Desktop Safari'] } },
  ],
});
```

**6. Page Objects: encapsular una pantalla**

En lugar de repetir los mismos locators en cada test, un *Page Object* agrupa los elementos y acciones de una pantalla en una clase. Si cambia el formulario de login, lo arreglas en un solo sitio:

```ts
export class LoginPage {
  constructor(private readonly page: Page) {}

  async iniciarSesion(email: string, password: string) {
    await this.page.goto('/login');
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Contraseña').fill(password);
    await this.page.getByRole('button', { name: 'Entrar' }).click();
  }
}
```

**7. Fixtures: preparar el contexto de cada test**

Una *fixture* es un recurso que Playwright construye antes del test y limpia después — como los [fixtures de xUnit](../testing-dotnet/Fixtures-y-ciclo-de-vida.md). `page`, `browser` y `context` son fixtures integradas; puedes crear las tuyas con `test.extend` (por ejemplo, un usuario ya autenticado):

```ts
export const test = base.extend<{ loginPage: LoginPage }>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page)); // se inyecta en cada test que la pida
  },
});
```

**8. Trazas y artefactos de depuración**

Cuando un test falla en CI y "en tu máquina va bien", la **traza** es tu mejor amiga: una grabación paso a paso con capturas, snapshots del DOM, peticiones de red y consola. Se abre en un visor local:

```bash
npx playwright show-trace trace.zip   # traza de una ejecución concreta
npx playwright show-report            # informe HTML con vídeos y capturas de los fallos
```

## Lo que NO hace

- **No es un test unitario** — arrancar un navegador y un servidor cuesta segundos; para probar una función pura, Vitest o xUnit son órdenes de magnitud más rápidos.
- **No sustituye a los tests de integración** — un test E2E te dice *que* algo falla, pero no siempre *dónde*; los tests de integración de backend siguen siendo los que localizan el fallo en el SQL o el repositorio.
- **No corre sin la aplicación levantada** — Playwright no arranca tu backend ni tu base de datos por ti; necesita un stack en marcha al que apuntar (en local, en CI, o vía la opción `webServer` para el front).
- **No es una herramienta de tests de carga** — mide si un flujo *funciona*, no cuántos usuarios simultáneos aguanta.

## Buenas prácticas avanzadas

- **Reutiliza la autenticación con `storageState`, no repitas el login en cada test** — hacer login por UI en los 50 tests los vuelve lentos y frágiles. Autentícate una vez en un *setup project*, guarda las cookies y el `localStorage` en un fichero con `context.storageState({ path })` y cárgalo en el resto: los demás tests arrancan ya dentro de la aplicación. El propio flujo de login sí se prueba a mano, pero solo ahí.
- **Cada test debe crear sus propios datos y ser independiente** — si el test B depende de que el test A haya dejado un usuario creado, en cuanto se ejecuten en paralelo o cambie el orden, todo revienta. Playwright paraleliza por defecto: diseña datos que no colisionen (emails únicos, por ejemplo) y limpia lo que ensucies.
- **Prohíbe `waitForTimeout` en tus reglas de linter** — un `page.waitForTimeout(2000)` es la fuente número uno de tests *flaky*: o esperas de más (suite lenta) o de menos (fallo intermitente). Si sientes la tentación de poner un sleep, casi siempre existe una aserción web-first (`toBeVisible`, `toHaveURL`) que expresa de verdad qué estás esperando.
- **Activa `trace: 'on-first-retry'` y deja que CI reintente una vez** — así no pagas el coste de grabar trazas en cada ejecución verde, pero cuando algo falla tienes la grabación completa del intento fallido. Es la diferencia entre depurar un fallo de CI en cinco minutos o en una tarde.
- **Prueba el caso de error, no solo el camino feliz** — en autenticación, el test valioso no es solo "login correcto → home"; es también "credenciales inválidas → mensaje de error y sigo en /login" y "token invalidado → me redirige al login". Esos son los que protegen contra regresiones de seguridad silenciosas.
- **Prefiere locators por rol accesible** — usar `getByRole` en vez de clases CSS no solo hace el test más robusto ante cambios de maquetación, sino que además falla si rompes la accesibilidad (un botón sin rol o sin nombre no se encuentra), convirtiendo tu suite E2E en un chequeo de accesibilidad de propina.

## Playwright en CI

El sitio natural de la suite E2E es el pipeline, como **puerta de validación antes de producción**. El patrón habitual: en una rama de integración previa a producción (típicamente `staging`), un workflow **levanta un stack efímero** — la API, el frontend y la base de datos en contenedores dentro del propio runner (ver [Entornos efímeros en CI](../../devops/ci-cd/Entornos-Efimeros-en-CI.md)) — y lanza Playwright contra ese `localhost`. Si un *golden path* como el login se rompe, el merge hacia producción se bloquea.

```yaml
- name: Levantar el stack y esperar a que responda
  run: docker compose up -d --wait

- name: Instalar navegadores de Playwright
  run: npx playwright install --with-deps

- name: Ejecutar los tests E2E
  run: npx playwright test
  env:
    BASE_URL: http://localhost:5173
```

> Nota: `npx playwright install --with-deps` descarga los navegadores y sus dependencias del sistema; en local se ejecuta una sola vez tras instalar el paquete.

## Recursos didácticos divertidos

- [try.playwright.dev](https://try.playwright.dev/) — un playground en el navegador para escribir y ejecutar tests sin instalar nada.
- `npx playwright codegen <url>` — abre un navegador, graba tus clics y va escribiendo el test por ti; la mejor forma de aprender qué locator usar para cada elemento.
- [Trace Viewer](https://trace.playwright.dev/) — sube una traza y explora la línea de tiempo, las capturas y el DOM de cada paso; entender una traza fallida es media depuración.

---

*En resumen: Playwright maneja un navegador real como lo haría un usuario para probar tu aplicación de punta a punta — pocos tests, en la cima de la pirámide, que protegen los flujos que no te puedes permitir romper.*
