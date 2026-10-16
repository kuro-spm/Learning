# SEO en páginas web

## ¿Qué es?

El SEO (*Search Engine Optimization*) es el conjunto de prácticas que hacen que un buscador pueda **encontrar, entender y mostrar** las páginas de una web en sus resultados. Se reparte entre lo técnico (que el buscador pueda acceder a la página), el contenido (que la página diga con claridad de qué trata) y la reputación (que otras webs la enlacen). Esta guía cubre la parte que controla quien desarrolla la web.

## ¿Por qué existe?

Una web que nadie encuentra no existe a efectos prácticos: buena parte de las visitas llega desde un buscador. Y el buscador no ve la web como una persona. Un robot descarga el HTML, extrae el texto, los enlaces y unas pocas etiquetas, y con eso decide de qué trata la página y si merece aparecer para una búsqueda. Si la página devuelve un error, esconde su contenido tras JavaScript que el robot no ejecuta o repite el mismo título en todas las URLs, el buscador no tiene con qué trabajar, por muy buena que sea la web para una persona.

> Si ya conoces las cabeceras HTTP y los códigos de estado, piensa en el SEO como el contrato con un cliente muy particular: un robot que solo entiende estados HTTP claros, enlaces `<a href>` y HTML con significado, y que toma sus decisiones sin preguntar.

## ¿Cuándo y para qué se usa?

En cualquier web cuyo tráfico dependa de que se la encuentre: una tienda online (cada producto es una página que debe poder aparecer al buscarlo), un blog, la web de una empresa. Una aplicación tras un login (un panel interno, una app de tareas) no necesita SEO en sus pantallas privadas, pero sí en su página pública de presentación.

---

## Cómo trabaja un buscador: rastreo, indexación y posicionamiento

Son tres fases consecutivas, y un fallo en una impide las siguientes:

| Fase | Qué ocurre | Qué la rompe |
|---|---|---|
| **Rastreo** (*crawling*) | Un robot sigue enlaces y descarga páginas. | Páginas sin ningún enlace que lleve a ellas, `robots.txt` que las bloquea, errores del servidor, lentitud extrema. |
| **Indexación** (*indexing*) | El buscador analiza la página y decide guardarla en su índice. | `noindex`, contenido duplicado, contenido pobre, el texto se genera con JavaScript que no llega a ejecutarse. |
| **Posicionamiento** (*ranking*) | Para cada búsqueda, ordena las páginas indexadas. | Contenido que no responde a lo que se busca, página lenta o incómoda de usar, falta de enlaces desde otras webs. |

Por eso el diagnóstico va por orden: antes de preguntarse por qué una página sale en la posición 15, hay que comprobar que está indexada.

## Que la página se pueda rastrear e indexar

### Códigos de estado honestos

El robot interpreta el código de estado igual que cualquier cliente HTTP:

- `200` — la página existe, se indexa.
- `301` — se movió para siempre; el buscador transfiere la posición a la URL nueva. Úsalo al cambiar una URL.
- `404` / `410` — no existe; se acaba retirando del índice.
- `5xx` — error temporal; el robot reintenta, y si persiste, baja el ritmo con el que visita la web.

El fallo clásico es el ***soft 404***: una página «producto no encontrado» que devuelve `200`. Para el buscador es una página válida con poco contenido, y la indexa. La respuesta correcta para algo que no existe es un `404` real, aunque el cuerpo muestre un mensaje amable. Es especialmente frecuente en aplicaciones de una sola página, donde el servidor devuelve siempre el mismo `index.html` con `200` y es el JavaScript quien «decide» que la ruta no existe.

### `robots.txt` y `noindex`: dos cosas distintas

`robots.txt`, en la raíz del dominio, dice a los robots **qué rutas no deben rastrear**:

```text
User-agent: *
Disallow: /admin/
Disallow: /carrito

Sitemap: https://example.com/sitemap.xml
```

El error más frecuente es usarlo para **ocultar** páginas del buscador. No sirve: una página bloqueada en `robots.txt` puede aparecer igualmente en los resultados, solo con la URL y sin descripción, si otras webs la enlazan. Para pedir que **no se indexe**, la página tiene que poder rastrearse y decirlo ella misma:

```html
<meta name="robots" content="noindex">
```

Y aquí está la trampa que lo remata: si bloqueas la página en `robots.txt` **y** le pones `noindex`, el robot nunca llega a leer el `noindex`, porque no puede descargarla. Para sacar algo del índice, `noindex` sí; `robots.txt`, no. (Este último solo se usa para ahorrar visitas inútiles del robot a rutas sin valor, como filtros infinitos o el carrito.)

### El sitemap

Un `sitemap.xml` es la lista de URLs que quieres que el buscador conozca. No garantiza que las indexe, pero acelera el descubrimiento y es la mejor vía para páginas a las que casi nada enlaza:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/productos/zapatillas-rojas</loc>
    <lastmod>2026-03-05</lastmod>
  </url>
</urlset>
```

Debe incluir solo URLs **definitivas**: las que responden `200`, son canónicas (ver más abajo) y no tienen `noindex`. Un sitemap lleno de redirecciones y errores resta confianza. Lo normal es generarlo desde la base de datos en lugar de mantenerlo a mano.

### El contenido tiene que estar en el HTML que recibe el robot

Los buscadores modernos ejecutan JavaScript, pero lo hacen en una segunda pasada, más tarde y con más coste, y otros robots (de redes sociales, de otros buscadores) no lo ejecutan en absoluto. Una página que llega vacía (`<div id="root"></div>`) y se rellena en el navegador funciona peor que una que llega con el contenido ya escrito.

Para una web que vive del buscador, conviene que el HTML inicial contenga el texto, los títulos y los enlaces importantes. Las opciones, de menos a más sencilla de mantener según el caso:

- **Renderizado en servidor (SSR)** — el servidor devuelve el HTML completo en cada petición (Next.js, Razor Pages, MVC).
- **Generación estática (SSG)** — el HTML se genera en el *build* y se sirve como fichero; ideal para contenido que cambia poco (blogs, documentación).
- **Prerenderizado** — se genera el HTML de las rutas públicas de una SPA existente.

Un cliente que solo renderiza en el navegador es una decisión válida para un panel interno. No lo es para un catálogo de productos que debe posicionar.

## Qué escribir en cada página

### El título y la descripción

Es lo que aparece en el resultado de búsqueda, y lo primero que lee quien decide si hacer clic:

```html
<head>
  <title>Zapatillas rojas de running para hombre | Tienda Ejemplo</title>
  <meta name="description" content="Zapatillas ligeras con amortiguación para correr en asfalto. Envío gratis en 24 h y devolución en 30 días.">
  <link rel="canonical" href="https://example.com/productos/zapatillas-rojas">
</head>
```

- `<title>` — **único en cada página**, con lo más específico primero y el nombre de la marca al final. El buscador lo recorta por anchura (unos 50-60 caracteres); lo que sobra no se ve. Es una de las señales más fuertes de qué trata la página.
- `meta description` — no influye directamente en la posición, pero es el texto que suele mostrarse bajo el título, y por tanto decide el clic. Una descripción distinta y que diga algo concreto por página; si falta, el buscador elige un fragmento del contenido.
- `canonical` — la URL «oficial» de esta página. Si el mismo contenido es accesible desde varias URLs (`/productos/x`, `/productos/x?utm_source=correo`, `/productos/x?color=rojo`), le dices cuál es la buena para que el buscador no las trate como duplicados. Cada página debe apuntar a **sí misma** si es la oficial.

Un patrón muy común y muy dañino es que un framework ponga el mismo `<title>` y la misma descripción en todas las páginas porque nadie los configuró por ruta. Es lo primero que conviene revisar.

### La estructura del contenido

Los títulos del texto cuentan una jerarquía, y el buscador la usa para entender qué es lo importante:

```html
<h1>Zapatillas rojas de running</h1>   <!-- un solo h1 por página, el tema principal -->
  <h2>Características</h2>
  <h2>Guía de tallas</h2>
    <h3>Cómo medir el pie</h3>
```

Se usan los elementos de título (`h1`–`h6`) por su significado, no por su tamaño: un `<div class="grande">` con aspecto de título no cuenta como tal. Lo mismo vale para listas (`ul`, `ol`), tablas (`table`) y, en general, para el HTML semántico: es la misma estructura que usan los lectores de pantalla, por lo que mejorarla sirve a las dos cosas (ver [WCAG](../accesibilidad-web/WCAG.md)).

### URLs, enlaces e imágenes

- **URLs legibles y estables.** `/productos/zapatillas-rojas` dice de qué trata; `/p?id=4821&cat=7` no. Y una URL, una vez publicada, no debería cambiar: si hay que moverla, un `301` desde la antigua.
- **Enlaces con `<a href>` y texto descriptivo.** El robot solo sigue enlaces reales. Un `<div onclick="...">` o un `<button>` que navega con JavaScript no es un enlace. Y el texto del enlace es información: «ver guía de tallas» dice algo; «haz clic aquí» no.
- **Texto alternativo en las imágenes** (`alt`). Describe lo que muestra la imagen; el buscador lo usa para entenderla y posicionarla en la búsqueda de imágenes, y los lectores de pantalla lo leen. Una imagen decorativa lleva `alt=""`.

## Datos estructurados: decirle al buscador qué es cada cosa

El texto de una página es ambiguo para un robot: `129,99 €` puede ser un precio, un número de pedido o una errata. Los **datos estructurados** lo etiquetan sin ambigüedad usando el vocabulario de [schema.org](https://schema.org/), normalmente como JSON-LD dentro de la página:

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Zapatillas rojas de running",
  "image": "https://example.com/img/zapatillas-rojas.jpg",
  "description": "Zapatillas ligeras con amortiguación para correr en asfalto.",
  "offers": {
    "@type": "Offer",
    "price": "129.99",
    "priceCurrency": "EUR",
    "availability": "https://schema.org/InStock"
  }
}
</script>
```

Con esto, el buscador puede mostrar el precio y la disponibilidad directamente en el resultado (un *rich result*). Hay tipos para artículos (`Article`), migas de pan (`BreadcrumbList`), preguntas frecuentes (`FAQPage`), eventos y muchos más. Dos reglas:

- **Los datos marcados deben coincidir con lo que ve la persona en la página.** Etiquetar un precio que no aparece, o reseñas inventadas, va contra las directrices y puede acabar en una penalización.
- Se genera desde los mismos datos que pintan la página (la fila del producto), no se escribe a mano por página: si el precio cambia en la base de datos, cambia en las dos partes.

## Rendimiento: Core Web Vitals

Una página lenta o que «salta» mientras carga es peor para quien la usa y, además, Google incluye su experiencia en el posicionamiento. Mide tres cosas, llamadas **Core Web Vitals**:

| Métrica | Qué mide | Umbral de «bueno» |
|---|---|---|
| **LCP** (*Largest Contentful Paint*) | Cuánto tarda en verse el elemento más grande (una imagen principal, un titular). | ≤ 2,5 s |
| **INP** (*Interaction to Next Paint*) | Cuánto tarda la página en responder visualmente a un clic o una pulsación. | ≤ 200 ms |
| **CLS** (*Cumulative Layout Shift*) | Cuánto se desplaza el contenido mientras carga (el botón que se mueve justo al pulsarlo). | ≤ 0,1 |

Las causas habituales y su arreglo son concretas:

```html
<!-- ❌ Sin dimensiones, el navegador no reserva sitio y el texto salta al cargarse la imagen (CLS) -->
<img src="/img/zapatillas-rojas.jpg" alt="Zapatillas rojas">

<!-- ✅ Con dimensiones y carga diferida para lo que queda fuera de pantalla -->
<img src="/img/zapatillas-rojas.jpg" alt="Zapatillas rojas" width="800" height="600" loading="lazy">
```

- La imagen principal de la parte visible (la que determina el LCP) **no** lleva `loading="lazy"`: retrasarla empeora justo la métrica que se quiere mejorar.
- Para el INP, el culpable típico es JavaScript pesado que bloquea el hilo principal: dividir el código por rutas y evitar trabajo largo en los manejadores de eventos.
- Para el LCP también influye el servidor: un tiempo de respuesta lento (consultas sin caché, sin CDN) retrasa todo lo demás.

## Comprobar el resultado: Search Console

**Google Search Console** es la herramienta gratuita donde Google cuenta qué ve de tu web. Lo que más sirve al principio:

- **Inspección de URL**: pegas una URL y te dice si está indexada, cuándo se rastreó por última vez y cómo la ve el robot (el HTML ya renderizado). Es la forma de comprobar si el contenido generado con JavaScript llega o no.
- **Páginas (indexación)**: lista las que no están indexadas y la razón («Excluida por etiqueta noindex», «Página con redirección», «Soft 404»...). Es el diagnóstico más directo de los problemas de las secciones anteriores.
- **Rendimiento**: las búsquedas por las que se aparece, las impresiones, los clics y la posición media.
- **Experiencia**: los Core Web Vitals medidos con usuarios reales.

Los cambios tardan días o semanas en reflejarse; no se mide el SEO de una modificación al día siguiente.

---

## Buenas prácticas avanzadas

- **Controla el contenido duplicado que generan las URLs con parámetros.** Filtros, ordenaciones, paginación y etiquetas de seguimiento (`?utm_source=`, `?sort=price`) multiplican las URLs de una misma página. Decide cuáles tienen valor propio (una categoría filtrada que la gente busca) y cuáles no, y para las segundas usa `canonical` hacia la versión base. Bloquearlas en `robots.txt` evita el rastreo, pero no soluciona la duplicación si ya están enlazadas.
- **Una redirección, no una cadena.** Si `/a` redirige a `/b` y esta a `/c`, cada salto cuesta tiempo y reparte mal la señal. Al reestructurar URLs, actualiza las redirecciones antiguas para que vayan directamente al destino final, y mantén las `301` mientras haya enlaces externos hacia la URL vieja (en la práctica, años).
- **Verifica la paginación y el contenido «infinito».** Un listado que carga más productos al hacer scroll solo funciona para el buscador si cada tanda también existe como URL con enlace rastreable (`?page=2`). Si los productos de la página 2 solo aparecen por una llamada de JavaScript, el robot no los descubre.
- **Revisa qué ve el robot, no qué ves tú.** Compara el HTML crudo (`curl -s https://example.com/producto | head`) y el HTML renderizado de la Inspección de URL de Search Console. Lo que está en el segundo pero no en el primero depende de que se ejecute JavaScript.
- **Mantén los datos estructurados dentro de las directrices y vuelve a validarlos tras cada cambio de plantilla.** Es muy fácil romperlos sin que nada falle visualmente: un campo `price` vacío o con coma decimal (`129,99` en lugar de `129.99`) invalida el *rich result* y solo lo notas semanas después. Valida con la herramienta de resultados enriquecidos.
- **Mide con datos de campo, no solo con pruebas de laboratorio.** Lighthouse mide un móvil simulado y una carga; los Core Web Vitals que cuentan son los de usuarios reales (informe de Search Console, CrUX). Una página puede sacar 100 en laboratorio y fallar en campo por culpa de dispositivos más lentos o de scripts de terceros (publicidad, analítica).
- **Las webs multiidioma necesitan además su señalización propia** (`hreflang`, una URL por idioma): se explica en [Webs multiidioma](../webs-multiidioma/Webs-multiidioma.md).

## Documentación oficial

- [Guía de inicio de SEO (Google Search Central)](https://developers.google.com/search/docs/fundamentals/seo-starter-guide) — el recorrido completo desde cero, con las recomendaciones de Google sobre títulos, URLs, enlaces e imágenes. La primera lectura.
- [Conceptos básicos de SEO con JavaScript](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics) — cómo renderiza Google el JavaScript y qué patrones de SPA dan problemas; imprescindible si la web se pinta en el navegador.
- [Introducción a los datos estructurados](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data) — qué tipos generan resultados enriquecidos y los requisitos de cada uno.
- [RFC 9309: Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309) — la fuente normativa de `robots.txt`, para cuando la duda es cómo se interpreta una regla.

## Recursos didácticos

- [Search Console](https://search.google.com/search-console) — gratuita; añadir la web y mirar el informe de indexación es la forma más rápida de ver sus problemas reales.
- [Prueba de resultados enriquecidos](https://search.google.com/test/rich-results) — pega una URL o un fragmento de HTML y dice qué datos estructurados detecta y qué errores tienen.
- [PageSpeed Insights](https://pagespeed.web.dev/) — mide una URL con datos de laboratorio y, si los hay, de usuarios reales, y enumera las mejoras por impacto.
- [web.dev: Core Web Vitals](https://web.dev/articles/vitals) — explica cada métrica con ejemplos de las causas más frecuentes y sus arreglos.

---

*En resumen: el SEO técnico consiste en que el robot pueda llegar a cada página (rastreo), entienda de qué trata (HTML, títulos, datos estructurados) y no tenga motivos para descartarla (errores, duplicados, lentitud); antes de preguntar por qué una página no sube, hay que comprobar que está indexada.*
