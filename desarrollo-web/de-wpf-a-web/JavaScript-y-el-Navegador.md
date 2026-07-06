# JavaScript y el navegador

**¿Qué es?** JavaScript es el lenguaje de programación que se ejecuta **dentro del
navegador**, en el ordenador de quien usa la web. Es el que hace que una página
reaccione a clics, valide formularios sin recargar o actualice partes de la pantalla al
vuelo, sin pedir una página nueva al servidor.

---

## ¿Por qué existe?

En WPF, el código que respondía a un clic (tu code-behind o tu comando de `ViewModel`)
corría en la misma máquina que la interfaz. En la web, la interfaz vive en el navegador
de otra persona, y ahí tu C# **no se ejecuta**: el navegador solo sabe ejecutar
JavaScript. Por eso, cualquier interactividad que ocurra sin ir al servidor (abrir un
menú, mostrar un error de validación al instante, arrastrar un elemento) la hace
JavaScript.

> Si vienes de WPF: JavaScript es el "code-behind del cliente". Es el código que se
> ejecuta junto a la interfaz para reaccionar a lo que hace la usuaria, igual que tus
> manejadores de eventos en escritorio.

---

## ¿Cuándo y para qué se usa?

Aparece siempre que la página tiene que reaccionar al instante, sin recargarse. En una
tienda online: actualizar el contador del carrito al añadir un producto, mostrar "el
correo no es válido" mientras escribes, o cargar más resultados al hacer scroll. Todo
eso, sin ir y volver del servidor cada vez, lo hace JavaScript.

---

## Lo mínimo que necesitas saber

**1. Tu C# corre en el servidor; JavaScript corre en el navegador**

Esta es la frontera clave. El servidor (tu C#) prepara y envía la página; a partir de
ahí, lo que pasa en pantalla sin recargar lo maneja JavaScript.

```text
Servidor (tu C#)      →  genera el HTML y los datos
Navegador (JavaScript) →  reacciona a clics, valida, actualiza la pantalla
```

**2. JavaScript manipula el HTML ya dibujado (el DOM)**

El navegador representa la página como un árbol de objetos llamado **DOM**. JavaScript
puede leerlo y modificarlo en caliente:

```javascript
document.querySelector("#contador-carrito").textContent = "3";
```

**3. Suele hablar con tu servidor por HTTP en segundo plano**

Cuando JavaScript necesita datos frescos, hace una petición HTTP a tu API (ver
[Web API y REST](Web-API-y-REST.md)) y actualiza solo la parte necesaria de la página:

```javascript
const respuesta = await fetch("/api/productos");
const productos = await respuesta.json();
```

**4. Si te incomoda salir de C#, existe Blazor**

[Blazor](Blazor.md) te permite escribir esa interactividad de cliente en C# en lugar de
JavaScript. Es la opción más cómoda para alguien que viene de WPF y no quiere aprender
otro lenguaje de golpe.

---

## Lo que NO hace

- **No reemplaza al servidor** — la lógica de negocio sensible y los datos siguen en tu C#; el cliente no es de fiar.
- **No es C#** — es otro lenguaje, con su propia sintaxis; conviene conocer lo básico aunque uses Blazor.
- **No persiste datos de forma segura** — lo que guarda el navegador es manipulable; la verdad vive en el servidor.

---

## Buenas prácticas avanzadas

- **JavaScript tiene un solo hilo, como la UI de WPF** — todo el código de la página corre en el mismo hilo que la dibuja: un bucle pesado congela la pantalla igual que un cálculo largo congelaba tu ventana de WPF. La diferencia es que aquí `async/await` no crea hilos: solo cede el turno mientras se espera (a la red, a un temporizador). Lo asíncrono no bloquea; lo intensivo en CPU, sí, aunque lo envuelvas en `async`.
- **`fetch` no falla cuando el servidor devuelve un error** — un `404` o un `500` son, para `fetch`, respuestas perfectamente válidas: la promesa solo se rechaza si la red falla. El bug clásico es procesar como éxito un error del servidor. Comprueba siempre `response.ok` (o `response.status`) antes de leer el JSON:

  ```javascript
  const respuesta = await fetch("/api/productos");
  if (!respuesta.ok) throw new Error(`Error ${respuesta.status}`);
  ```
- **Carga los scripts sin bloquear la página** — un `<script>` clásico en el `<head>` detiene el dibujado del HTML hasta descargarse y ejecutarse. Usa `<script type="module">` (o el atributo `defer`): el script se descarga en paralelo y se ejecuta cuando el HTML ya está listo, con lo que además puedes tocar el DOM sin esperas ni trucos.

---

*En resumen: JavaScript es el lenguaje del navegador y el responsable de la interactividad en el cliente; tu C# se queda en el servidor, y si no quieres salir de C# para el cliente, Blazor es tu puente.*
