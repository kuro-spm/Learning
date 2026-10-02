# Capturas de pantalla con Playwright

## ¿Qué es?

Usar Playwright como **generador de imágenes**: un script que abre la aplicación, la recorre como lo haría una persona y guarda una captura de cada pantalla relevante para un manual de usuario o una documentación. Su entregable no es un test en verde, son los ficheros de imagen.

## ¿Por qué existe?

Las capturas de un manual se hacen casi siempre a mano, y tienen un problema de fondo: **caducan en silencio**. Basta con que un botón cambie de sitio o de texto para que el manual muestre una pantalla que ya no existe, y nadie se entera hasta que alguien se queja. Rehacer treinta capturas a mano en cada versión es además un trabajo tedioso y poco reproducible: cada vez se hacen con otros datos, otro tamaño de ventana y otro orden.

Un script hace siempre el mismo recorrido, con los mismos datos y el mismo tamaño de ventana. Si la interfaz cambia y un paso deja de ser posible, el script **falla señalando el paso**, que es justo el aviso que el manual necesitaba.

> Si ya conoces los tests E2E, piensa en esto como un test cuyo `expect` sirve para asegurarse de que la pantalla está lista, y cuyo resultado es la imagen que deja por el camino.

Esta ficha se centra en ese caso. Los fundamentos de Playwright (locators, auto-waiting, fixtures, ejecución) están en [Playwright](../../testing/e2e/Playwright.md), y hay una guía complementaria con otros matices de este mismo uso (locators frente al cambio de idioma, esperar respuestas de red, botones con estado, recorridos con pasos caros) en [Playwright para documentación visual](../../testing/e2e/Playwright-Documentacion-Visual.md). Aquí no se repiten.

## Un recorrido natural, no capturas sueltas

Hay dos formas de organizar el script. La primera es una captura por pantalla, cada una en su test, con sus propios datos de partida. La segunda es **un único recorrido de principio a fin**, en el mismo orden en que alguien usaría la aplicación, capturando en cada paso.

Para un manual conviene la segunda, por tres razones:

- **Coherencia visual.** El producto que se ve en el paso 5 es el mismo que se creó en el paso 3: mismo nombre, mismos datos. Con capturas sueltas, cada imagen muestra un ejemplo distinto y el manual parece un collage.
- **Menos preparación.** Cada captura suelta necesita crear sus propios datos de partida; en un recorrido, el paso anterior ya los dejó listos.
- **El orden documenta.** Si el recorrido «crear producto → publicarlo → ver el pedido» se puede hacer de punta a punta, es la prueba de que el manual cuenta algo realizable.

La contrapartida es que un fallo a mitad impide generar las capturas posteriores; por eso se deja alguna forma de relanzar solo una parte (ver «Recapturar solo una parte»).

```ts
test('recorrido del manual de la tienda', async ({ page }) => {
  // 1. Acceso
  await page.goto('/login');
  await captura(page, '01-login');
  await page.getByLabel('Email').fill(EMAIL);
  await page.getByLabel('Contraseña').fill(PASSWORD);
  await page.getByRole('button', { name: 'Entrar' }).click();

  // 2. Catálogo
  await page.goto('/productos');
  await captura(page, '02-catalogo');

  // 3. Alta de un producto, y su resultado
  // ...
});
```

## Un estado de partida determinista

Una captura es reproducible solo si la aplicación está **en el mismo estado** cada vez. Dos factores se controlan explícitamente.

### El idioma, antes de navegar

Si la aplicación recuerda el idioma en `localStorage`, hay que fijarlo **antes** de que cargue la primera página. `context.addInitScript` ejecuta código en cada página del contexto antes de que corra ningún script de la aplicación:

```ts
import { test as base } from '@playwright/test';

const test = base.extend({
  context: async ({ context }, use) => {
    await context.addInitScript(() => {
      window.localStorage.setItem('idioma', 'es');
    });
    await use(context);
  },
});
```

Hacerlo con `page.evaluate` tras `goto` llega tarde: la página ya se pintó en el idioma por defecto y la primera captura saldría en otro idioma. La opción `locale` de Playwright (`test.use({ locale: 'es-ES' })`) cambia el idioma que el navegador anuncia, que solo sirve si la aplicación lo lee de ahí en lugar de guardarlo ella misma.

La misma lógica vale para el tamaño de ventana y el tema: fija `viewport` (y `colorScheme` si hay modo oscuro) en la configuración, para que todas las capturas tengan las mismas proporciones.

### Datos de ejemplo, de forma idempotente

Un manual debería mostrar datos reconocibles («Camiseta básica», «Pedido 1042»), no los restos de pruebas anteriores. El script crea los datos de ejemplo que necesita, pero tiene que poder ejecutarse **más de una vez** sin romperse: la segunda ejecución no debe chocar con lo que dejó la primera (por ejemplo, un índice único sobre el nombre rechazaría un duplicado). La receta es *comprobar antes de crear*:

```ts
async function asegurarProducto(page: Page, nombre: string) {
  await page.goto('/productos');
  const fila = page.getByRole('cell', { name: nombre, exact: true });
  if ((await fila.count()) > 0) return; // ya existe: no se vuelve a crear

  await page.getByRole('button', { name: 'Nuevo producto' }).click();
  await page.getByLabel('Nombre').fill(nombre);
  await page.getByRole('button', { name: 'Guardar' }).click();
  await expect(fila).toBeVisible();
}
```

Dos precisiones: `count()` no espera, así que antes hay que asegurarse de que la lista ya cargó (siguiente sección); y si los datos de ejemplo no se borran al terminar, decide qué hacer con ellos (dejarlos como datos de muestra permanentes es una opción válida, pero debe ser una decisión consciente y documentada en el propio script).

## Esperar antes de capturar

`page.screenshot()` toma la foto **en el instante en que se ejecuta**. No espera a que los datos hayan llegado ni a que una animación termine. Por eso la captura se ejecuta siempre después de comprobar que la pantalla está en su estado final. El fallo típico es una imagen del manual con un «Cargando…» o un esqueleto gris donde debía haber contenido.

La pauta es esperar a **dos cosas**: que lo que quieres mostrar sea visible, y que lo transitorio haya desaparecido.

```ts
await page.goto('/pedidos');
await expect(page.getByRole('heading', { name: 'Pedidos' })).toBeVisible();
await expect(page.getByText('Cargando…')).toBeHidden(); // el spinner se ha ido
await expect(page.getByRole('row')).not.toHaveCount(1);  // hay filas además de la cabecera
await captura(page, '05-pedidos');
```

Esperar solo al título no basta: el título suele pintarse al instante y los datos llegan después. Esperar solo a que desaparezca el spinner tampoco, porque puede no haber aparecido todavía. Y un `waitForTimeout(2000)` «por si acaso» es la peor opción: unas veces sobra y otras no alcanza (las razones de esperar hechos y no plazos están en [Playwright para documentación visual](../../testing/e2e/Playwright-Documentacion-Visual.md)).

Dos ayudas más de la propia captura: `animations: 'disabled'` detiene las animaciones CSS y las lleva a su estado final, y `caret: 'hide'` oculta el cursor parpadeante de los campos de texto (es el comportamiento por defecto).

## Hacer la captura

`page.screenshot` captura la ventana visible; `locator.screenshot` captura solo un elemento. Encapsula la llamada en una función para tener en un único sitio la carpeta, el formato y la calidad:

```ts
import { mkdirSync } from 'node:fs';
import path from 'node:path';
import type { Locator, Page } from '@playwright/test';

const CARPETA = path.join('..', 'docs', 'manual', 'images');

async function captura(page: Page, nombre: string) {
  mkdirSync(CARPETA, { recursive: true });
  await page.screenshot({
    path: path.join(CARPETA, `${nombre}.jpg`),
    type: 'jpeg',
    quality: 90,
    animations: 'disabled',
  });
}

async function capturaDe(elemento: Locator, nombre: string) {
  await elemento.screenshot({ path: path.join(CARPETA, `${nombre}.png`) });
}
```

Las opciones que se usan de verdad:

| Opción | Para qué sirve |
|---|---|
| `fullPage: true` | Captura la página entera, no solo lo visible (cuidado con páginas muy largas). |
| `locator.screenshot()` | Solo un panel, un formulario o un icono, sin el ruido del resto. |
| `clip: { x, y, width, height }` | Un rectángulo concreto de la página. |
| `mask: [locator, ...]` | Tapa con un recuadro de color elementos que no deben salir (emails reales, datos personales). |
| `type` y `quality` | Formato `png` (por defecto) o `jpeg`; `quality` (0-100) solo existe en `jpeg`. |

### Formato y calidad

PNG no pierde información y es el mejor para texto y bordes nítidos, pero pesa mucho. JPEG con `quality` entre 80 y 90 reduce el tamaño de forma notable con una pérdida casi imperceptible en capturas de interfaz, y es una buena opción cuando las imágenes van a vivir en un repositorio o dentro de un documento. Elige un formato para todo el manual: mezclar PNG y JPEG sin motivo da un resultado desigual.

### Nombres de fichero

Los nombres de imagen son parte del contrato con el manual (que los referencia en el texto). Usa nombres **estables y descriptivos** (`catalogo.jpg`, `alta-producto.jpg`), sin fechas ni contadores automáticos: si el nombre cambia en cada ejecución, el manual apunta a imágenes que ya no existen. Si necesitas un orden, prefija con número (`03-alta-producto.jpg`) y asume que insertar un paso intermedio obliga a renumerar.

## Un proyecto de Playwright para las capturas

Las capturas **no son un test de corrección** y no deben ejecutarse con la suite normal: ni en cada commit, ni en el CI. Puede tardar mucho, puede crear datos, y puede tocar servicios que cobran. Playwright permite aislarlas con un *proyecto* propio en `playwright.config.ts`:

```ts
export default defineConfig({
  testDir: './e2e',
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
      testIgnore: [/manual-screenshots\.spec\.ts/], // la suite normal no lo ejecuta
    },
    {
      name: 'manual-screenshots',
      testMatch: /manual-screenshots\.spec\.ts/,
      use: { ...devices['Desktop Chrome'], viewport: { width: 1280, height: 800 } },
      retries: 0, // una acción cara no debe repetirse sola
    },
  ],
});
```

Tres decisiones aquí:

- **`testIgnore` en el proyecto normal y `testMatch` en el de capturas**, para que cada fichero pertenezca a un solo proyecto.
- **Sin `dependencies`**, salvo que el recorrido las necesite de verdad. Si el script hace su propio login, no hay razón para arrastrar los proyectos de preparación de la suite (y con ello sus efectos).
- **`retries: 0`.** Un reintento automático repite todo el test, incluidos los pasos que ya tuvieron efectos (altas, envíos, llamadas a un servicio de pago). En una suite de tests el reintento ayuda con la inestabilidad; en un recorrido con efectos reales cuesta datos duplicados o dinero. La razón de fondo está en [Playwright para documentación visual](../../testing/e2e/Playwright-Documentacion-Visual.md).

Se lanza a mano, solo cuando hay que regenerar el manual:

```bash
npx playwright test --project=manual-screenshots
```

## Recapturar solo una parte

Si cambia una única pantalla no tiene sentido repetir un recorrido de quince pasos. Hay dos formas de acotar:

1. **Un test separado y barato para el subconjunto**, con un nombre reconocible. Es el patrón recomendado cuando hay un grupo de pantallas que se pueden alcanzar sin pasar por las costosas.
2. **`--grep` para elegir por título.** Filtra los tests cuyo nombre coincide con la expresión.

```ts
test('capturas del catálogo', async ({ page }) => {
  // login + captura de /productos, sin pasos caros
});
```

```bash
npx playwright test --project=manual-screenshots --grep "catálogo"
```

`--grep` filtra **tests enteros**, no pasos dentro de un test. Por eso, si un único recorrido largo contiene todo, no hay forma de ejecutar solo su mitad con `--grep`. Si quieres granularidad, divide el recorrido en varios tests, pensando en qué estado deja cada uno para el siguiente (y por eso la idempotencia de los datos de ejemplo es tan importante: permite que cada test prepare lo que necesita).

## Errores frecuentes

| Síntoma | Causa habitual | Solución |
|---|---|---|
| La captura muestra «Cargando…» o un esqueleto | Se capturó sin esperar a que terminara la carga. | Esperar a que el indicador desaparezca **y** a que el contenido esté visible, antes de `screenshot`. |
| La primera captura sale en otro idioma | El idioma se fijó después de cargar la página. | `context.addInitScript` antes de la primera navegación. |
| El script falla en la segunda ejecución | Los datos de ejemplo se crean sin comprobar si existen. | Crear solo si no existen (receta idempotente). |
| Cada captura muestra datos distintos | Se usan datos reales o aleatorios. | Datos de ejemplo con nombres fijos; `mask` para lo que no deba verse. |
| Los elementos salen movidos o a medias | Una animación o transición sin terminar. | `animations: 'disabled'` y esperar al estado final. |
| Las imágenes tienen proporciones distintas entre ejecuciones | El tamaño de ventana depende del entorno. | Fijar `viewport` en el proyecto. |
| `Error: ENOENT` al guardar | La carpeta de destino no existe. | `mkdirSync(carpeta, { recursive: true })` antes de capturar. |
| El manual muestra imágenes rotas | Los nombres cambian entre ejecuciones. | Nombres estables y descriptivos; sin fechas ni aleatorios. |

## Cuándo recapturar

Regenerar las capturas debería ser un paso explícito de la **preparación de una release**, no algo que se haga cuando alguien lo recuerda:

- Cuando cambia el aspecto o el texto de una pantalla incluida en el manual.
- Cuando se añade o se retira un paso del flujo documentado.
- Al preparar una versión nueva, aunque nadie recuerde cambios: es la forma de descubrir que un paso del recorrido dejó de funcionar.

Y tras recapturar, **mira las imágenes**: el script garantiza que el recorrido se completó, no que cada captura sea buena. Un vistazo rápido a la carpeta detecta un spinner colado, un dato personal sin enmascarar o una pantalla recortada.

## Buenas prácticas avanzadas

- **Haz que el script falle antes de capturar algo equivocado.** Cada captura va precedida de un `expect` sobre lo que debe verse (un título, una fila concreta, el número de elementos). Sin esa aserción, un recorrido «exitoso» puede producir una imagen de la pantalla de error y nadie lo nota hasta imprimir el manual.
- **Mantén los textos visibles del script en un solo lugar.** Si el recorrido localiza botones por su texto, cada cambio de redacción en la interfaz rompe el script. Un `data-testid` o un `id` estable, o un único módulo con los locators, reduce el mantenimiento al tocar solo un fichero.
- **Marca los pasos que gastan recursos reales** (llamadas de pago, correos enviados, cuota de una API) con un comentario visible, y no los incluyas en el subconjunto que se recaptura a menudo.
- **Enmascara en vez de confiar en los datos.** Aunque trabajes con datos de ejemplo, un día alguien ejecuta el script contra un entorno con datos reales; `mask` sobre emails, nombres y tokens evita publicarlos por accidente.
- **Guarda las capturas junto al texto que las usa**, y no en una carpeta temporal. Si el manual y sus imágenes viven en el mismo repositorio, una release cambia ambos en el mismo commit y siempre se pueden comparar.
- **Documenta en la cabecera del script cómo lanzarlo, qué crea y qué cuesta.** Quien lo ejecute dentro de seis meses no recordará que deja datos de muestra ni que gasta cuota.

## Documentación oficial

- [Screenshots (Playwright)](https://playwright.dev/docs/screenshots) — guía corta con los casos habituales: página entera, elemento, buffer en memoria. Empieza por aquí.
- [`page.screenshot`](https://playwright.dev/docs/api/class-page#page-screenshot) — la lista completa de opciones (`mask`, `clip`, `animations`, `quality`...) con su comportamiento exacto.
- [Test projects](https://playwright.dev/docs/test-projects) — cómo definir proyectos, dependencias y filtros, necesario para aislar el generador de capturas de la suite normal.

## Recursos didácticos

- [Playwright para documentación visual](../../testing/e2e/Playwright-Documentacion-Visual.md) — la ficha hermana de este repositorio, con los matices de idioma, esperas y pasos caros que aquí no se repiten.
- Para integrar las imágenes generadas en un manual, ver [Markdown a HTML en build time](Markdown-a-HTML-en-build-time.md), [Modal con zoom y arrastre](Modal-con-zoom-y-arrastre.md) y [Documentos Word con python-docx](Documentos-Word-con-python-docx.md).

---

*En resumen: un recorrido natural de Playwright, con estado determinista y esperas antes de cada captura, convierte las imágenes de un manual en un artefacto que se regenera en vez de en una tarea manual que caduca.*
