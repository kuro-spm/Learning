# Webs multiidioma

## ¿Qué es?

Una web multiidioma es la que ofrece el mismo contenido en varios idiomas (y, a menudo, con las costumbres de cada país: formato de fechas, moneda, unidades). Hacerla bien se reparte entre dos tareas: **internacionalizar** (preparar el código para que no dependa de un idioma concreto) y **localizar** (aportar cada idioma concreto).

## ¿Por qué existe?

Traducir una web parece cosa de copiar los textos a otro idioma. En cuanto se intenta, aparecen los problemas que no son de traducción:

- ¿Cómo sabe el servidor qué idioma quiere quien visita la página?
- ¿Cómo encuentra Google la versión en inglés sin tratarla como contenido duplicado de la española?
- ¿Qué pasa con `"Tienes " + n + " mensajes"` cuando el otro idioma pone el número detrás, o necesita otra forma para el plural?
- ¿Y con `05/03/2026`, que es 5 de marzo en España y 3 de mayo en Estados Unidos?

Cada una de esas preguntas tiene una respuesta establecida. Aplicarlas desde el principio cuesta poco; arreglarlas con la web ya en producción significa tocar cada página, cada URL y buena parte de la base de datos.

> Si ya conoces los ficheros de recursos (`.resx` en .NET, `messages.properties` en Java), la parte de los textos es la misma idea. Lo nuevo es todo lo que rodea a los textos: URLs, SEO, formatos y datos.

## ¿Cuándo y para qué se usa?

En una tienda online que vende en varios países, en un blog con versión en castellano y en catalán, o en una app interna que usan equipos de distintos países. Aunque hoy solo haya un idioma, merece la pena tomar tres decisiones desde el primer día porque son las más caras de cambiar después: **cómo se representa el idioma en la URL**, **que los textos no estén escritos a fuego en el código** y **que las fechas y los números pasen por una función de formato**.

---

## Vocabulario: idioma, región y *locale*

Un **locale** (o configuración regional) es la combinación de idioma y, opcionalmente, región. Se escribe como una etiqueta **BCP 47**: el código de idioma de dos letras, y la región si hace falta, unidos por guion.

| Etiqueta | Significado |
|---|---|
| `es` | Español, sin concretar país |
| `es-ES` | Español de España |
| `es-MX` | Español de México |
| `ca` | Catalán |
| `en-GB` / `en-US` | Inglés británico / estadounidense |
| `pt-BR` | Portugués de Brasil |

Dos reglas de uso:

- **El código de idioma va en minúscula y la región en mayúscula**, con guion (`es-ES`), no con guion bajo. El guion bajo (`es_ES`) es el formato de algunos sistemas (Linux, Java antiguo) y no vale en HTML ni en cabeceras HTTP.
- **Empieza solo con el idioma (`es`, `en`) y añade región únicamente cuando haya diferencias que importen** (precios en otra moneda, vocabulario distinto). Mantener `es-ES`, `es-MX` y `es-AR` como tres traducciones completas triplica el trabajo.

Un matiz que se confunde a menudo: **el idioma no es el país**. Hay países con varios idiomas (Suiza, Bélgica, Canadá) e idiomas repartidos entre muchos países. Por eso no se debe escoger el idioma por la IP de quien visita: la IP dice dónde está, no qué lee con comodidad.

## Elegir el idioma: una URL por idioma

La decisión de diseño más importante es esta: **cada idioma tiene sus propias URLs**. Hay cuatro formas de cargar el idioma en la petición:

| Estrategia | Ejemplo | Comentario |
|---|---|---|
| Prefijo en la ruta | `example.com/es/productos` | La más sencilla de mantener y de posicionar. **Recomendada por defecto.** |
| Subdominio | `es.example.com` | Permite separar infraestructura, pero cada subdominio se trata como un sitio distinto. |
| Dominio por país | `example.es`, `example.fr` | La señal más fuerte para buscadores, pero cuesta un dominio y un certificado por país. |
| Parámetro o cookie | `example.com/productos?lang=es` | **Evitar como única señal**: la misma URL devuelve contenido distinto según quién la pida. |

Con parámetro o cookie, la URL deja de identificar el contenido. Eso rompe tres cosas a la vez: quien comparte un enlace en español lo ve en inglés si la otra persona tiene otra cookie, la caché (de navegador o CDN) puede servir la página equivocada, y los buscadores solo ven una versión. Con el idioma en la URL, un enlace significa lo mismo para todo el mundo.

### El idioma del navegador sirve para proponer, no para imponer

El navegador envía la cabecera `Accept-Language` con los idiomas que prefiere quien navega:

```http
Accept-Language: ca,es;q=0.8,en;q=0.5
```

Se lee así: catalán preferido; castellano con peso 0,8; inglés con 0,5. El uso correcto es **solo en la primera visita a la raíz** (`/`): se redirige a la versión del idioma que mejor encaje. Después manda la URL, y un selector de idioma visible permite cambiarlo. Si cada petición se redirigiera según `Accept-Language`, quien eligiera inglés a mano volvería a ser devuelto al castellano.

El síntoma típico de equivocarse aquí es una web que solo se ve en un idioma desde ciertos buscadores o ciertos países: suele ser una redirección automática por idioma o por IP que también atiende a los rastreadores.

### Un ejemplo en el servidor

En ASP.NET Core, el middleware de localización fija la cultura de cada petición a partir de proveedores. Primero se declara qué idiomas se soportan:

```csharp
var supported = new[] { "es", "ca", "en" };

app.UseRequestLocalization(new RequestLocalizationOptions()
    .SetDefaultCulture("es")
    .AddSupportedCultures(supported)
    .AddSupportedUICultures(supported));
```

- `SetDefaultCulture("es")` — idioma cuando ninguna señal encaja o el idioma pedido no está soportado.
- `AddSupportedCultures` — cultura para **formatear** números y fechas.
- `AddSupportedUICultures` — cultura para **buscar los textos** traducidos (los ficheros de recursos).

De serie, esos proveedores leen primero el parámetro `?culture=`, luego una cookie y por último `Accept-Language`. Para el idioma en la ruta (`/es/productos`) hay que añadir un proveedor propio que lea el primer segmento de la URL y lo ponga en primer lugar de la lista; la documentación oficial lo describe en su sección de proveedores de cultura personalizados.

## Decirle a navegadores y buscadores qué idioma es cada página

Con una URL por idioma, falta declarar la relación entre ellas. Hay tres piezas, y las tres van en el `<head>` de cada página.

**1. El idioma de la propia página**, en la etiqueta raíz:

```html
<html lang="es">
```

Lo usan los lectores de pantalla para escoger la pronunciación, el navegador para ofrecer traducir (o no) y el corrector ortográfico. Una web en catalán con `lang="en"` la leerá un lector de pantalla con acento inglés. Si un fragmento está en otro idioma, se marca ese elemento: `<span lang="en">checkout</span>`.

**2. Las alternativas, con `hreflang`**:

```html
<link rel="alternate" hreflang="es" href="https://example.com/es/productos">
<link rel="alternate" hreflang="ca" href="https://example.com/ca/productes">
<link rel="alternate" hreflang="en" href="https://example.com/en/products">
<link rel="alternate" hreflang="x-default" href="https://example.com/">
```

Esto le dice a Google que las cuatro URLs son **la misma página en otro idioma**, no contenido duplicado, y que sirva a cada persona la que le corresponde. `x-default` indica qué mostrar cuando ningún idioma encaja (suele ser la página que propone el idioma, o la versión principal). Dos reglas que se incumplen mucho:

- **Debe ser recíproco.** Cada página lista a todas las demás **y a sí misma**. Si la versión en inglés no apunta de vuelta a la española, el buscador ignora la anotación.
- Las URLs son **absolutas**, con protocolo y dominio.

**3. La URL canónica de cada versión apunta a sí misma**:

```html
<link rel="canonical" href="https://example.com/es/productos">
```

Error habitual: poner como canónica la versión en el idioma principal en todas las páginas. Es decirle al buscador «las versiones en otros idiomas son copias; indexa solo esta», y las traducciones desaparecen de los resultados.

Las mismas relaciones pueden declararse en el `sitemap.xml` en lugar de en cada página, lo que resulta más cómodo con muchas URLs.

## Textos: claves, variables y plurales

Los textos viven fuera del código, en un fichero por idioma, y el código pide cada uno por una **clave**. En [i18n con claves en React](../manual-de-usuario-en-la-app/i18n-con-claves-en-React.md) está el mecanismo completo; aquí interesa lo que hay que cuidar al escribir cada texto.

### No construyas frases pegando trozos

```tsx
// ❌ MAL — el orden de las palabras queda fijado en el código
<p>{t('cart.youHave')} {count} {t('cart.items')}</p>
```

Cada idioma ordena la frase a su manera, y la parte «items» puede necesitar plural. La traducción se hace por **frases completas**, con marcadores para lo variable:

```json
{
  "es": { "cart.summary": "Tienes {count} productos en el carrito" },
  "en": { "cart.summary": "There are {count} items in your cart" }
}
```

```tsx
// ✅ BIEN — la frase entera es la unidad de traducción
<p>{t('cart.summary', { count })}</p>
```

### Los plurales no son «añadir una s»

En español o inglés hay dos formas (singular y plural), pero otros idiomas tienen una, tres o seis, y la regla de cuándo usar cada una varía (el polaco o el árabe, por ejemplo). Las reglas están recogidas en el estándar **CLDR**, y los formatos de mensajes tipo **ICU** las expresan así:

```
{count, plural,
  one   {# producto en el carrito}
  other {# productos en el carrito}
}
```

La librería de textos escoge `one` u `other` (o las categorías que necesite cada idioma) según el número. Quien traduce solo escribe las formas que su idioma necesita. El navegador expone esas reglas con `Intl.PluralRules`, y la mayoría de librerías de i18n (i18next, FormatJS) las usan por debajo. Lo que no hay que hacer es escribir `count === 1 ? 'producto' : 'productos'` en el componente: funciona en español y falla en el siguiente idioma.

### Qué hacer cuando falta una traducción

Decide el comportamiento de antemano. Lo razonable es **caer al idioma por defecto** (que se vea un texto en castellano en medio de la página en catalán) y **registrar la clave que falta**, en lugar de mostrar la clave en crudo (`cart.summary`) o dejar un hueco vacío. Un test o un paso de CI que compare las claves de todos los idiomas con las del idioma base detecta los huecos antes de que lleguen a producción.

## Estructura de carpetas para los textos

Los ficheros de traducción se organizan por **idioma** en el primer nivel y por **área funcional** en el segundo:

```
src/
└── locales/
    ├── es/
    │   ├── common.json     ← botones, menú, errores genéricos
    │   ├── auth.json
    │   ├── cart.json
    │   └── products.json
    ├── ca/
    │   ├── common.json
    │   ├── auth.json
    │   ├── cart.json
    │   └── products.json
    └── en/
        └── (los mismos ficheros)
```

Cada elección tiene su razón:

- **El idioma como carpeta de primer nivel** permite entregar `ca/` entera a quien traduce, y añadir un idioma consiste en copiar una carpeta.
- **Los mismos nombres de fichero y las mismas claves en todos los idiomas** permiten comparar cada idioma con el base y detectar claves que faltan (el paso de CI de la sección anterior).
- **Un fichero por área, no uno gigante**, evita conflictos de git cuando varias personas tocan textos a la vez y permite cargar solo los textos de la pantalla que se abre (*lazy loading*), en lugar de todos los de la web.
- **Las claves llevan el área como prefijo** (`cart.summary`): son únicas y se sabe dónde se usan. `common.json` queda para lo que aparece en todas partes.

Lo mismo en ASP.NET Core, donde los textos son ficheros de recursos (`.resx`) y el nombre del idioma va en el fichero, junto a la clase o área a la que pertenecen:

```
Resources/
├── Cart.resx           ← idioma por defecto (es)
├── Cart.ca.resx
├── Cart.en.resx
├── Auth.resx
├── Auth.ca.resx
└── Auth.en.resx
```

Dos estructuras que conviene evitar:

- **Duplicar el árbol de páginas por idioma** (`pages/es/...`, `pages/en/...`). Cada cambio de diseño se repite en cada idioma y las versiones se desincronizan. La página es una sola; lo que cambia es el diccionario y el prefijo `/es/` de la ruta, que se resuelve como un segmento de ruta y no como una carpeta.
- **Mezclar en el mismo fichero los textos de interfaz y los contenidos editables.** Si quien edita los contenidos no es parte del equipo de desarrollo, esos datos pertenecen a la base de datos (sección «Contenido que vive en la base de datos»), no a un fichero del repositorio que exige despliegue para cambiarse.

## Formatos y maquetación

### Fechas, números y monedas se formatean, no se escriben

Cada idioma y región pone distinto el separador decimal, el de miles, el orden de día y mes y la posición del símbolo de moneda. No se resuelve con una tabla propia: el navegador (`Intl`) y .NET (la cultura actual) ya incluyen los datos.

```ts
const price = 12345.5;
const date = new Date('2026-03-05T00:00:00Z');

new Intl.NumberFormat('es-ES', { style: 'currency', currency: 'EUR' }).format(price);
// "12.345,50 €"
new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(price);
// "$12,345.50"

new Intl.DateTimeFormat('es-ES', { dateStyle: 'long', timeZone: 'UTC' }).format(date);
// "5 de marzo de 2026"
new Intl.DateTimeFormat('en-US', { dateStyle: 'long', timeZone: 'UTC' }).format(date);
// "March 5, 2026"
```

Tres detalles que se confunden:

- **La moneda no depende del idioma.** Quien lee en inglés desde España puede pagar en euros: la moneda sale del país o del catálogo de precios, y el formato (`en-US` o `es-ES`) solo decide cómo se escribe. En el ejemplo, `currency` es un dato aparte del *locale*.
- **Guarda las fechas en UTC y conviértelas al mostrarlas.** La zona horaria es una decisión de presentación, igual que el idioma, y la de quien visita no tiene por qué ser la del servidor.
- **No hagas aritmética con el texto ya formateado.** El formato es solo para mostrar; los cálculos se hacen con los valores originales.

### Un texto cambia de longitud al traducirlo

Lo que cabe en un botón en inglés suele ocupar más en alemán o en español (a menudo entre un 20 % y un 40 % más). El diseño debe aguantarlo: nada de anchos fijos para botones o menús, y probar las pantallas con el idioma más largo, no solo con el original.

### Idiomas de derecha a izquierda

El árabe y el hebreo se escriben de derecha a izquierda, y la interfaz entera se refleja: el menú pasa al otro lado, las flechas cambian de sentido. En HTML se declara con `dir`:

```html
<html lang="ar" dir="rtl">
```

Para que el CSS funcione en ambos sentidos, usa **propiedades lógicas** en lugar de físicas:

```css
/* ❌ solo sirve de izquierda a derecha */
.card { margin-left: 1rem; padding-right: 2rem; text-align: left; }

/* ✅ sigue el sentido de escritura */
.card { margin-inline-start: 1rem; padding-inline-end: 2rem; text-align: start; }
```

`inline-start` significa «el lado donde empieza el texto»: la izquierda en español y la derecha en árabe. Si la web no va a ofrecer nunca un idioma RTL, no hace falta más, pero escribir el CSS así desde el principio no cuesta nada.

## Contenido que vive en la base de datos

Los textos de la interfaz van en ficheros de idioma, pero el nombre y la descripción de un producto o el cuerpo de un artículo son **datos**: los editan personas, no el equipo de desarrollo. Tener una columna `name_es`, `name_en`, `name_ca` en la tabla `products` obliga a una migración cada vez que se añade un idioma. El modelo habitual es una tabla de traducciones aparte:

```sql
CREATE TABLE products (
    id    INT PRIMARY KEY,
    price NUMERIC(10, 2) NOT NULL          -- lo que no depende del idioma
);

CREATE TABLE product_translations (
    product_id  INT NOT NULL REFERENCES products(id),
    locale      VARCHAR(10) NOT NULL,      -- 'es', 'ca', 'en'
    name        VARCHAR(200) NOT NULL,
    description TEXT,
    slug        VARCHAR(200) NOT NULL,     -- la URL también se traduce
    PRIMARY KEY (product_id, locale),
    UNIQUE (locale, slug)
);
```

- Lo que **no depende del idioma** (precio, stock, fecha de alta) se queda en la tabla principal.
- Se consulta con el idioma de la petición y, si no existe fila, se cae al idioma por defecto, igual que con los textos de interfaz.
- El `slug` traducido (`/es/productos/zapatillas-rojas`, `/en/products/red-sneakers`) hace que la URL esté en el idioma de quien lee. Esa URL traducida debe aparecer en los `hreflang` de cada versión, así que el modelo tiene que poder devolver el slug de cualquier idioma a partir de la misma entidad.

---

## Buenas prácticas avanzadas

- **Traduce frases completas con contexto, no palabras sueltas.** «Close» es «Cerrar» en un botón y «Cercano» en un mapa; «Order» es «Pedido» o «Orden» según el caso. Quien traduce necesita ver dónde aparece el texto: añade una descripción o una captura junto a cada clave, o usa claves que lo dejen claro (`checkout.order.title`).
- **Haz que el selector de idioma enlace a la misma página en otro idioma.** De `/es/productos/x` a `/en/products/x`, no a la portada del otro idioma. Perder el sitio donde se estaba es la queja más frecuente de quien cambia de idioma.
- **Haz que los correos, los PDF y las notificaciones usen el idioma del destinatario, no el de la petición.** Un correo de confirmación se genera a menudo en un proceso en segundo plano sin petición HTTP: guarda el idioma preferido en el perfil de la persona o en el pedido, y úsalo ahí.
- **Prueba con pseudolocalización.** Una pseudotraducción automática alarga y adorna los textos (`[!!! Ñéw prödúct !!!]`) y revela a simple vista los textos escritos a fuego en el código, los anchos fijos y los marcadores mal colocados, antes de pagar una sola traducción.
- **Ordena y compara cadenas con la cultura, no con el código de carácter.** Ordenar nombres alfabéticamente con una comparación ordinal pone la «Á» detrás de la «Z»; `Intl.Collator` o `StringComparer` con cultura lo resuelve. Lo mismo con las mayúsculas: `ToUpper()` sin cultura falla con la «i» turca (`i` y `İ`).
- **No traduzcas lo que debe ser estable.** Identificadores, claves de API, valores de enumeración, nombres de campos y códigos de estado van en un idioma neutro; solo se traduce lo que se muestra. Si el texto que ve la persona se usa también como valor en la base de datos, cambiar de idioma rompe filtros y comparaciones.

## Documentación oficial

- [Localized versions of your page (Google Search Central)](https://developers.google.com/search/docs/specialty/international/localized-versions) — cómo declarar las versiones por idioma con `hreflang` y `x-default`, incluidos los errores de reciprocidad que hacen que se ignore la anotación. Es la primera lectura para la parte de SEO.
- [Language declarations in HTML (W3C Internationalization)](https://www.w3.org/International/questions/qa-html-language-declarations) — cuándo y cómo usar `lang` y `dir`, con ejemplos de fragmentos mezclados.
- [`Intl` (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl) — el formateo de números, fechas, plurales y ordenación del navegador; cada constructor tiene su página con todas las opciones.
- [Globalización y localización en ASP.NET Core (Microsoft Learn)](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/localization) — el middleware de localización, los ficheros de recursos y los proveedores de cultura, incluido cómo escribir uno propio.
- [RFC 5646 (BCP 47)](https://www.rfc-editor.org/rfc/rfc5646) — la fuente normativa de las etiquetas de idioma, para cuando la duda es qué etiqueta es válida.

## Recursos didácticos

- [Reglas de plurales de CLDR](https://www.unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html) — tabla con las categorías de plural de cada idioma y ejemplos de números; deja claro por qué `n === 1` no basta.
- [Sintaxis de mensajes ICU](https://unicode-org.github.io/icu/userguide/format_parse/messages/) — guía de los marcadores `plural`, `select` y `selectordinal` que usan la mayoría de librerías de i18n.
- [Propiedades lógicas de CSS (MDN)](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_logical_properties_and_values) — equivalencias entre `margin-left` y `margin-inline-start` y cómo se comportan en cada modo de escritura.

---

*En resumen: una web multiidioma se resuelve con una URL por idioma, `lang` y `hreflang` que lo declaren, frases completas en lugar de texto pegado y formatos que pasen por `Intl`; el idioma deja de ser un parche cuando cada una de esas piezas está desde el principio.*
