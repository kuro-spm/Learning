# HTML y CSS

## ¿Qué es?

**HTML** es el lenguaje con el que se describe la estructura de una página web (los
títulos, los párrafos, los botones, los campos de un formulario). **CSS** es el lenguaje
con el que se le da aspecto (colores, tamaños, márgenes, posición). El navegador lee
ambos y dibuja la interfaz.

## ¿Por qué existe?

En WPF describías tu interfaz con **XAML**: una etiqueta `<Button>`, un `<TextBox>`, un
`<StackPanel>`... y dabas estilo con `Style`, `Setter` y recursos. El navegador no
entiende XAML, así que la web tiene su propio par equivalente: HTML para la estructura y
CSS para el estilo. Conceptualmente es la misma idea —un lenguaje de marcado declarativo
para describir una UI— solo que con otra sintaxis y otro motor que lo dibuja.

> Si vienes de WPF: HTML es tu XAML (el árbol de elementos) y CSS es tu hoja de estilos
> (lo que en WPF repartías entre `Style`, `ResourceDictionary` y propiedades como
> `Margin` o `Background`).

## ¿Cuándo y para qué se usa?

Toda interfaz web es HTML + CSS, siempre, sin excepción. Da igual si la genera el
servidor con Razor, si la pinta un framework de JavaScript o si la produce Blazor: el
resultado final que el navegador dibuja es HTML con estilos CSS. Por ejemplo, la ficha
de un producto en una tienda es HTML (la foto, el nombre, el precio, el botón) y CSS (que
el botón sea naranja y el precio grande).

## Lo mínimo que necesitas saber

**1. HTML es un árbol de etiquetas anidadas**

Igual que en XAML anidabas paneles y controles, en HTML anidas etiquetas. Cada etiqueta
abre con `<tag>` y cierra con `</tag>`:

```html
<div class="tarjeta-producto">
  <h2>Camiseta</h2>
  <p>Precio: 19,99 €</p>
  <button>Añadir al carrito</button>
</div>
```

Equivalencias mentales rápidas:

```text
WPF                     HTML
StackPanel / Grid   →   <div>
TextBlock           →   <p>, <span>, <h1>..<h6>
TextBox             →   <input>
Button              →   <button>
Image               →   <img>
```

**2. Los atributos configuran cada elemento**

```html
<input type="email" placeholder="Tu correo" />
<a href="/contacto">Contacto</a>
<img src="/logo.png" alt="Logo" />
```

**3. CSS aplica estilos buscando elementos por su selector**

En lugar de poner el estilo elemento por elemento, defines reglas que apuntan a una
clase, una etiqueta o un id:

```css
/* todos los elementos con class="tarjeta-producto" */
.tarjeta-producto {
  border: 1px solid #ddd;
  padding: 16px;
  border-radius: 8px;
}

/* todos los <button> */
button {
  background-color: #ff6600;
  color: white;
}
```

> La clase CSS (`class="tarjeta-producto"`) es muy parecida a aplicar un `Style` con
> `Key` a varios controles de WPF: defines el aspecto una vez y lo reutilizas.

**4. El layout se hace con Flexbox y Grid, no con `StackPanel` y `Grid` de XAML**

Para colocar elementos en fila, columna o rejilla, CSS tiene **Flexbox** (1 dimensión,
como un `StackPanel`) y **CSS Grid** (2 dimensiones, como el `Grid` de XAML):

```css
.barra-superior {
  display: flex;          /* coloca los hijos en fila, como un StackPanel horizontal */
  justify-content: space-between;
  align-items: center;
}
```

**5. Hoy casi nadie escribe CSS a mano del todo**

En proyectos reales es habitual usar utilidades o frameworks de CSS (como Tailwind) o
bibliotecas de componentes ya estilizados. Pero entender HTML y CSS "a pelo" es la base:
sin ella, esas herramientas son magia incomprensible.

## Lo que NO hace

- **HTML no tiene lógica** — no hay bucles ni condiciones; eso lo aporta quien genera el HTML (el servidor, Blazor o JavaScript).
- **CSS no cambia el contenido** — solo el aspecto y la posición; no añade ni quita texto.
- **No reaccionan solos a eventos** — el "click" que hacía algo en tu code-behind de WPF aquí lo gestiona JavaScript, el servidor o Blazor, no el HTML por sí mismo.

## Buenas prácticas avanzadas

- **Usa la etiqueta semántica correcta, no un `<div>` con estilos** — un `<div onclick="...">` disfrazado de botón *parece* un botón, pero no se puede pulsar con el teclado, no lo anuncian los lectores de pantalla y no dispara el envío de formularios. `<button>`, `<a>`, `<nav>`, `<form>` o `<label>` traen gratis accesibilidad y comportamiento que tendrías que reimplementar a mano. Es la diferencia entre una web usable por todo el mundo y una que solo funciona con ratón.
- **Entiende la especificidad antes de pelearte con ella** — cuando un estilo "no se aplica", casi siempre es que otro selector más específico gana (un id pesa más que una clase; una clase más que una etiqueta). La reacción refleja es añadir `!important`, que solo escala la guerra. La práctica experta es al revés: selectores planos (una sola clase), nada de ids para estilar, y el inspector del navegador para ver qué regla está ganando y por qué.
- **Diseña fluido: la web no tiene tamaño de ventana** — en WPF controlabas el tamaño mínimo de tu ventana; en la web tu interfaz se verá en un móvil de 360px y en un monitor de 32". Evita anchos fijos en píxeles: usa `max-width`, porcentajes, `rem` para tamaños de texto y *media queries* para adaptar el layout. Una interfaz que solo se ve bien a un tamaño es una interfaz rota para media audiencia.
- **Anima `transform` y `opacity`, no `width` ni `top`** — cambiar el tamaño o la posición "real" de un elemento obliga al navegador a recalcular el layout de la página en cada fotograma (y se nota: animaciones a trompicones). `transform` y `opacity` se animan en la GPU sin tocar el layout. Es la razón por la que dos animaciones aparentemente iguales rinden tan distinto.

---

*En resumen: HTML y CSS son el XAML de la web —uno describe la estructura, el otro el aspecto— y son el formato final de cualquier interfaz, la genere quien la genere.*
