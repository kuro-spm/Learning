# Manual de usuario dentro de una app web — Guía de tecnologías

Conjunto de guías para construir una sección de ayuda con el manual de usuario integrado en una aplicación React: del Markdown fuente a una página navegable, con capturas generadas de forma automática y una versión en Word del mismo contenido. Pensado para quien programa a diario en backend o frontend y quiere ver cómo encajan las piezas.

---

## Orden de lectura recomendado

### 1. De Markdown a página

El contenido se escribe una vez en Markdown y se convierte en HTML antes de que la app lo necesite.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Markdown-a-HTML-en-build-time](Markdown-a-HTML-en-build-time.md) | Punto de partida: convertir el Markdown con un script y guardar el resultado como módulo. |
| 2 | [dangerouslySetInnerHTML-y-XSS](dangerouslySetInnerHTML-y-XSS.md) | Cómo mostrar ese HTML en React y cuándo es seguro hacerlo. |
| 3 | [i18n-con-claves-en-React](i18n-con-claves-en-React.md) | Los textos de la propia interfaz (menú, botones) en varios idiomas. |

### 2. La página de ayuda

Una vez el contenido está en pantalla, estas dos guías lo hacen cómodo de usar.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 4 | [Scroll-spy-con-IntersectionObserver](Scroll-spy-con-IntersectionObserver.md) | Índice lateral que resalta la sección visible y se queda fijo al hacer scroll. |
| 5 | [Modal-con-zoom-y-arrastre](Modal-con-zoom-y-arrastre.md) | Ampliar las capturas en un modal con zoom con la rueda y arrastre. |

### 3. Capturas y versión en Word

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 6 | [Capturas-de-pantalla-con-Playwright](Capturas-de-pantalla-con-Playwright.md) | Generar las imágenes del manual de forma repetible en lugar de a mano. |
| 7 | [Documentos-Word-con-python-docx](Documentos-Word-con-python-docx.md) | Producir el `.docx` entregable a partir del mismo Markdown. |

---

> Las rutas de la aplicación (por ejemplo, una página `/ayuda`) se definen con [TanStack Router](../frontend-react/TanStackRouter.md), y los fundamentos de Playwright están en [Playwright](../../testing/e2e/Playwright.md).
