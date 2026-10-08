# WCAG

## ¿Qué es?

WCAG (*Web Content Accessibility Guidelines*) es el estándar internacional que define, con criterios verificables, cuándo un sitio web es accesible para personas con discapacidad visual, auditiva, motriz o cognitiva. Lo publica el W3C.

## ¿Por qué existe?

Una web puede funcionar perfectamente para quien ve la pantalla y maneja un ratón, y ser inutilizable para quien navega con un lector de pantalla, solo con teclado, con la letra ampliada al 200 % o sin distinguir ciertos colores. Sin una referencia común, "accesible" significa lo que cada equipo quiera, y no se puede exigir en un contrato ni comprobar en una revisión.

WCAG convierte ese objetivo en una lista de **criterios de éxito** que se cumplen o no se cumplen. Este es un criterio real, el que exige que las imágenes tengan alternativa de texto:

```html
<!-- ❌ Un lector de pantalla dirá "imagen" o leerá el nombre del fichero -->
<img src="grafico-ventas-2025.png">

<!-- ✅ Lee el contenido que transmite la imagen -->
<img src="grafico-ventas-2025.png" alt="Las ventas crecieron un 18 % en 2025">
```

> Si ya conoces los tests automáticos, piensa en WCAG como la suite de pruebas que la web debe pasar: cada criterio es un caso con un resultado claro (cumple / no cumple), y el nivel de conformidad indica cuántos casos de la suite se exigen.

## ¿Cuándo y para qué se usa?

- **Al diseñar y maquetar**: define contrastes de color, tamaños de botón, orden de foco y estructura de los formularios desde el principio, en lugar de corregirlos al final.
- **Al revisar una entrega**: es la lista que se usa para decidir si una página nueva (una tienda online, un blog, un formulario de registro) se da por buena.
- **En contratos y requisitos legales**: casi toda norma de accesibilidad web remite a WCAG. En la Unión Europea, la normativa para el sector público y la Ley Europea de Accesibilidad aplican la norma EN 301 549, que se apoya en WCAG 2.1 nivel AA. Si te aplica una u otra depende de quién eres y de qué ofreces, así que conviene comprobarlo con quien asesore en lo legal; esta guía no lo sustituye.

---

## Cómo se organiza: principios, niveles y versiones

### Los cuatro principios (POUR)

Todos los criterios cuelgan de cuatro principios. Sirven para ubicar un problema antes de buscar el criterio exacto:

| Principio | Pregunta que responde | Ejemplo de fallo |
|---|---|---|
| **P**erceptible | ¿Se puede percibir el contenido con al menos un sentido? | Texto gris claro sobre blanco, vídeo sin subtítulos |
| **O**perable | ¿Se puede manejar con teclado, ratón, táctil o voz? | Menú que solo se abre al pasar el ratón |
| **C**omprensible | ¿Se entiende el contenido y lo que hace la interfaz? | Error de formulario que dice "Campo inválido" sin decir cuál |
| **R**obusto | ¿Lo interpretan bien navegadores y tecnologías de apoyo? | Un `<div>` que hace de botón y no se anuncia como tal |

### Los tres niveles de conformidad

Cada criterio de éxito tiene un nivel, y los niveles son acumulativos:

| Nivel | Qué significa | Cómo se usa |
|---|---|---|
| **A** | Mínimo. Sin esto, parte de las personas no puede usar la web en absoluto | Se cumple siempre |
| **AA** | Barreras importantes eliminadas | Es el nivel que se suele exigir en normativa y contratos |
| **AAA** | Máxima accesibilidad | No se exige como nivel global: el propio W3C avisa de que algunos contenidos no pueden cumplir todo el AAA |

"Cumplir AA" significa cumplir **todos** los criterios A **y** AA, no solo los AA.

### Las versiones

| Versión | Año | Qué aporta |
|---|---|---|
| 2.0 | 2008 | La base. Es también la norma ISO/IEC 40500 |
| 2.1 | 2018 | Añade criterios para móvil, baja visión y accesibilidad cognitiva |
| 2.2 | Oct. 2023 | Añade 9 criterios nuevos (ver más abajo) y elimina uno obsoleto, el 4.1.1 *Parsing*. Tiene 86 criterios en total |

Las versiones son **compatibles hacia atrás**: la 2.2 incluye todos los criterios de la 2.1, salvo el 4.1.1 que se ha retirado. En la práctica, una web que cumple 2.2 AA cumple también 2.1 AA.

## Qué nivel y qué versión elegir

| Situación | Elección razonable |
|---|---|
| Un contrato o norma pide "WCAG 2.1 AA" | Cumple 2.1 AA como mínimo. Apuntar a 2.2 AA no te perjudica |
| No hay requisito externo y se empieza de cero | **2.2 AA**: es la versión vigente y los criterios añadidos son baratos si se tienen en cuenta desde el diseño |
| Hay una web existente a la que se quiere dar un primer paso | Empieza por **A** y los criterios AA más frecuentes (contraste, teclado, formularios) y avanza por plantillas |
| Se plantea AAA | Solo para criterios o secciones concretas que lo justifiquen, nunca como objetivo de toda la web |

Un detalle que cambia cómo se planifica: la conformidad se mide por **páginas completas** y por **procesos completos**. Si una compra tiene cinco pantallas, o cumplen las cinco o el proceso no cumple.

## Los fallos más frecuentes, con su arreglo

Estos son los criterios que más se incumplen en revisiones reales. Cada uno se muestra con su fallo y su arreglo.

### Texto sin contraste suficiente (1.4.3, AA)

El texto normal necesita un contraste mínimo de **4,5:1** frente a su fondo; el texto grande (18 pt o 14 pt en negrita, más o menos 24 px o 19 px en negrita), **3:1**. Es el criterio que más se incumple y se arregla con los colores del diseño, no con código.

```css
/* ❌ Gris #777 sobre blanco: contraste 4,48:1, falla por muy poco */
.ayuda { color: #777777; background: #ffffff; }

/* ✅ #767676 sobre blanco: 4,54:1, es el gris más claro que cumple */
.ayuda { color: #767676; background: #ffffff; }
```

Los componentes de interfaz (bordes de un campo, un icono que transmite información) piden **3:1** (criterio 1.4.11).

### Campos de formulario sin etiqueta asociada (1.3.1 y 3.3.2, A)

Un `placeholder` no es una etiqueta: desaparece al escribir y muchos lectores de pantalla no lo anuncian de forma fiable.

```html
<!-- ❌ El lector de pantalla anuncia "campo de edición" y nada más -->
<input type="email" placeholder="Correo electrónico">

<!-- ✅ La etiqueta queda asociada al campo y se anuncia al enfocarlo -->
<label for="correo">Correo electrónico</label>
<input id="correo" type="email" autocomplete="email">
```

El atributo `autocomplete` cumple además el criterio 1.3.5 (AA): permite al navegador rellenar el campo.

### Controles que no son controles (4.1.2 y 2.1.1, A)

Un `<div>` con un `onclick` se ve como un botón, pero no recibe foco con el teclado, no responde a Enter ni a Espacio y no se anuncia como botón.

```html
<!-- ❌ Solo funciona con ratón -->
<div class="boton" onclick="enviar()">Enviar pedido</div>

<!-- ✅ El elemento nativo ya trae teclado, foco y rol -->
<button type="button" onclick="enviar()">Enviar pedido</button>
```

Es el motivo de la regla que más se repite en accesibilidad: **usa el elemento HTML nativo antes de añadir ARIA**.

### Foco invisible (2.4.7, AA)

Quien navega con teclado necesita ver dónde está. Quitar el contorno sin ofrecer otro es uno de los fallos más habituales.

```css
/* ❌ Elimina la única pista visual */
button:focus { outline: none; }

/* ✅ Un indicador propio, bien visible */
button:focus-visible { outline: 3px solid #1a56db; outline-offset: 2px; }
```

### Mensajes de error que no dicen qué falló (3.3.1, A)

```html
<!-- ❌ -->
<p class="error">Campo inválido</p>

<!-- ✅ Dice qué campo, qué pasó y queda anunciado -->
<label for="telefono">Teléfono</label>
<input id="telefono" type="tel" aria-describedby="telefono-error" aria-invalid="true">
<p id="telefono-error">El teléfono debe tener 9 dígitos. Has escrito 7.</p>
```

### Otros que conviene tener presentes

| Criterio | Qué exige | Cómo se incumple |
|---|---|---|
| 1.4.1 Uso del color (A) | No transmitir información *solo* con color | "Los campos en rojo son obligatorios" |
| 1.4.4 Cambio de tamaño (AA) | Poder ampliar el texto al 200 % sin perder contenido | Contenedores de altura fija que cortan el texto |
| 1.4.10 *Reflow* (AA) | Sin scroll horizontal con un ancho equivalente a 320 px | Tablas o maquetaciones de ancho fijo |
| 2.4.1 Saltar bloques (A) | Un enlace para saltar la cabecera y el menú | Quien usa teclado tabula 30 enlaces en cada página |
| 2.4.2 Título de la página (A) | Cada página con un `<title>` que la describa | Todas se llaman "Inicio" |
| 3.1.1 Idioma de la página (A) | `<html lang="es">` correcto | Sin `lang`: el lector pronuncia mal |
| 4.1.3 Mensajes de estado (AA) | Anunciar cambios sin mover el foco | "Producto añadido" aparece en pantalla pero no se anuncia |

## Qué añadió la versión 2.2

Nueve criterios nuevos. Los tres de nivel AA con más impacto en el día a día:

| Criterio | Nivel | Qué pide | Dónde suele fallar |
|---|---|---|---|
| 2.4.11 Foco no oculto (mínimo) | AA | El elemento enfocado no puede quedar totalmente tapado por contenido fijo | Una cabecera o un banner fijos que tapan el campo al tabular |
| 2.5.7 Movimientos de arrastre | AA | Todo lo que se haga arrastrando debe poder hacerse también con un clic | Ordenar una lista solo arrastrando |
| 2.5.8 Tamaño del objetivo (mínimo) | AA | Los controles táctiles y de puntero miden al menos 24 × 24 px CSS, o tienen espacio suficiente alrededor | Iconos de acción pegados unos a otros |
| 3.3.8 Autenticación accesible (mínimo) | AA | No obligar a memorizar o transcribir para iniciar sesión | Bloquear el pegado en el campo de contraseña |

Los otros cinco: 3.2.6 *Ayuda coherente* (A), 3.3.7 *Entrada redundante* (A), 2.4.12 *Foco no oculto (mejorado)* (AAA), 2.4.13 *Apariencia del foco* (AAA) y 3.3.9 *Autenticación accesible (mejorado)* (AAA).

## Cómo se comprueba

La comprobación se hace en tres capas, y ninguna basta por sí sola:

1. **Herramientas automáticas** (axe DevTools, Lighthouse, WAVE). Son rápidas y atrapan errores mecánicos (falta de `alt`, contraste insuficiente, etiquetas sin asociar), pero detectan solo una parte de los problemas: no pueden juzgar si un `alt` es útil o si el orden de lectura tiene sentido.
2. **Revisión manual con unas pocas pruebas fijas**:
   - **Solo teclado**: recorre la página con `Tab`, `Shift+Tab`, `Enter`, `Espacio` y `Esc`. ¿Se ve siempre dónde estás? ¿Puedes llegar a todo y salir de todo?
   - **Ampliación**: amplía el navegador al 200 % y a un ancho de 320 px.
   - **Lector de pantalla**: NVDA (gratuito, en Windows) o VoiceOver (macOS e iOS). Con diez minutos se detectan problemas que ninguna herramienta ve.
3. **Revisión de procesos completos**: una compra, un registro, una reserva, de principio a fin.

---

## Buenas prácticas avanzadas

- **Fija por escrito el nivel, la versión y el entorno de prueba antes de empezar.** "Accesible" no es un criterio de aceptación; "WCAG 2.2 AA, probado con teclado, NVDA y Chrome" sí lo es. Sin esa definición, cada revisión rediscute qué se exige.
- **Pon los contrastes en el sistema de diseño, no en cada pantalla.** Si los colores del tema ya están validados (texto, fondos, estados de foco y de error), cada pantalla nueva los hereda correctos. Comprobar contraste página por página es la forma más cara de hacerlo.
- **Primero HTML nativo, ARIA después y solo si hace falta.** ARIA cambia lo que se anuncia, no lo que se puede hacer: un `role="button"` sobre un `<div>` no le da teclado ni foco. Una ARIA mal puesta es peor que ninguna, porque anuncia algo que no es cierto.
- **Prueba los componentes dinámicos: modales, menús desplegables, banners.** Es donde más se rompe el teclado. Al abrir un modal, el foco entra en él y no escapa al contenido de detrás; al cerrarlo, vuelve al elemento que lo abrió. Un banner de consentimiento mal resuelto bloquea a quien navega con teclado nada más cargar la página.
- **Evalúa los contenidos de terceros y decide qué haces con ellos.** Un iframe de reservas, un mapa o un chat forman parte de tu página, aunque no controles su código. WCAG admite una declaración de **conformidad parcial** si documentas qué contenido de terceros queda fuera; lo que no vale es ignorarlo.
- **Anota qué se comprobó, con qué y qué queda pendiente.** Una revisión sin registro no se puede repetir ni defender ante una auditoría. Un fichero con la fecha, la versión de WCAG, las herramientas usadas y la lista de incumplimientos conocidos basta.

## Documentación oficial

- [WCAG 2.2](https://www.w3.org/TR/WCAG22/) — el texto normativo, con los criterios numerados. Se consulta cuando hace falta la redacción exacta de uno.
- [Cómo cumplir WCAG 2.2 (referencia rápida)](https://www.w3.org/WAI/WCAG22/quickref/) — los mismos criterios filtrables por nivel y por tecnología; es el punto de entrada práctico.
- [Entender WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/) — una página por criterio, con la intención, ejemplos y técnicas de cumplimiento. Es donde se resuelven las dudas de interpretación.
- [Patrones de ARIA (APG)](https://www.w3.org/WAI/ARIA/apg/) — cómo construir con teclado y ARIA correctos componentes complejos como modales, pestañas o menús.

## Recursos didácticos

- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/) — pega dos colores y dice si pasan AA o AAA para texto normal, texto grande y componentes.
- [axe DevTools](https://www.deque.com/axe/devtools/) — extensión del navegador que audita la página y enlaza cada problema con su criterio.
- [The A11Y Project: checklist](https://www.a11yproject.com/checklist/) — una lista de comprobación basada en WCAG, organizada por tema y legible sin conocer los números de criterio.
- [NVDA](https://www.nvaccess.org/download/) — lector de pantalla gratuito para Windows, para probar la web como lo haría quien lo usa a diario.

---

*En resumen: WCAG convierte "hazlo accesible" en una lista de criterios que se cumplen o no, y el nivel AA de la versión 2.2 es el punto de partida razonable; lo que no detectan las herramientas lo detecta recorrer la web solo con teclado.*
