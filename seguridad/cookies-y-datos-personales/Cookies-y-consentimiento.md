# Cookies y consentimiento

> Esta guía explica el funcionamiento técnico de las cookies y la normativa de la Unión Europea y de España que las regula. No es asesoramiento legal: ante una duda concreta, consulta a quien lleve la privacidad de la organización.

## ¿Qué es?

Una cookie es un pequeño dato que el servidor pide al navegador que guarde y que este devuelve en cada petición posterior. Sirve para que una web "recuerde" algo entre una página y la siguiente: quién ha iniciado sesión, qué hay en el carrito o qué idioma se eligió.

## ¿Por qué existe?

HTTP no tiene memoria: cada petición llega al servidor como si fuera la primera. Sin un mecanismo extra, la tienda online no sabría que quien pide `/carrito` es la misma persona que hace un minuto añadió un producto. La cookie es ese mecanismo: el servidor entrega un identificador y el navegador lo devuelve siempre.

El mismo mecanismo permite algo más delicado: reconocer a una persona entre sitios y a lo largo del tiempo, para analizar su comportamiento o mostrarle publicidad. Por eso las cookies están reguladas: la parte técnica es neutra, pero su uso puede afectar a la privacidad.

> Si ya conoces los tokens de sesión, piensa en una cookie como el sitio habitual donde el navegador guarda ese token y lo adjunta solo, sin que el código de la página tenga que hacer nada.

## ¿Cuándo y para qué se usa?

- **Mantener la sesión** de una persona que ha iniciado sesión en una tienda online.
- **Recordar preferencias**: idioma, moneda, tema oscuro.
- **Medir el uso** del sitio: cuántas visitas hay y qué páginas se consultan.
- **Publicidad y seguimiento** entre sitios, que es lo que más regulación recibe.

---

## Cómo funciona: el ciclo en HTTP

El servidor crea la cookie con la cabecera `Set-Cookie` de su respuesta. A partir de ahí, el navegador la adjunta con la cabecera `Cookie` en cada petición al mismo sitio.

```http
HTTP/1.1 200 OK
Set-Cookie: sessionId=a3f9c1; Max-Age=28800; Path=/; Secure; HttpOnly; SameSite=Lax
```

```http
GET /carrito HTTP/1.1
Host: tienda.ejemplo.com
Cookie: sessionId=a3f9c1
```

Los atributos que importan:

| Atributo | Qué hace | Cuándo se usa |
|---|---|---|
| `Max-Age` o `Expires` | Fija cuánto vive. Sin ellos es una cookie de **sesión** y desaparece al cerrar el navegador | Siempre que la cookie no deba ser permanente |
| `Secure` | Solo viaja por HTTPS | Siempre, en producción |
| `HttpOnly` | El JavaScript de la página no puede leerla | En toda cookie que guarde un identificador de sesión, para que un XSS no la robe |
| `SameSite` | Limita cuándo se envía en peticiones que vienen de otros sitios. `Lax` es un buen valor por defecto; `Strict` es más restrictivo; `None` exige `Secure` y habilita el envío entre sitios | Siempre; `None` solo si se necesita de verdad |
| `Domain` y `Path` | Acotan a qué dominios y rutas se envía | Cuanto más estrechos, mejor |

En ASP.NET Core, la misma cookie se crea así:

```csharp
Response.Cookies.Append("sessionId", sessionId, new CookieOptions
{
    HttpOnly = true,
    Secure = true,
    SameSite = SameSiteMode.Lax,
    MaxAge = TimeSpan.FromHours(8),
});
```

Esto envía la cabecera `Set-Cookie` anterior. Si olvidas `HttpOnly`, cualquier script de la página, también uno inyectado, puede leer el identificador de sesión.

## Tipos de cookies

Se clasifican de tres formas, y las tres cuentan para la normativa.

| Según… | Tipos | Qué significa |
|---|---|---|
| **Quién la gestiona** | Propias / de terceros | Las propias las crea el dominio que visitas. Las de terceros, otro dominio incrustado en la página (un anuncio, un vídeo, una herramienta de análisis) |
| **Para qué sirve** | Técnicas, de preferencias, de análisis, publicitarias | La finalidad es lo que decide si hace falta consentimiento |
| **Cuánto dura** | De sesión / persistentes | Las de sesión desaparecen al cerrar el navegador. Una cookie exenta de consentimiento debe tener una duración acorde con su finalidad |

Los navegadores restringen cada vez más las cookies de terceros (Safari y Firefox las bloquean por defecto), así que conviene no apoyarse en ellas para nada esencial.

Una nota para quien se pregunte por `localStorage`, píxeles de seguimiento o la huella digital del navegador: la normativa regula **guardar o acceder a información en el equipo de la persona**, sea con una cookie o con otra tecnología. Cambiar `localStorage` por una cookie no evita la obligación.

## Qué exige la normativa

La regla base viene de la Directiva ePrivacy (art. 5.3), que en España recoge la LSSI (art. 22.2). El consentimiento tiene el sentido del RGPD: libre, específico, informado e inequívoco. Lo que sigue resume la *Guía sobre el uso de las cookies* de la AEPD (actualizada en mayo de 2024).

**Cookies que no necesitan consentimiento** (las exceptuadas): las que son imprescindibles para prestar el servicio que la persona ha pedido. La guía recoge, entre otras:

- Cookies de "entrada del usuario" (por ejemplo, el contenido de un formulario en varios pasos o un carrito).
- Autenticación o identificación de usuario, solo de sesión.
- Seguridad del usuario.
- Sesión de reproductor multimedia.
- Sesión para equilibrar la carga.
- Personalización de la interfaz que la persona ha elegido (idioma, por ejemplo).

Basta con informar de ellas, de forma genérica, en la política de cookies o en la de privacidad.

**Cookies que sí lo necesitan:** cualquier otra, propia o de terceros, de sesión o persistente. Esto incluye las de análisis: la guía recuerda que no están exentas, aunque sean poco intrusivas.

**Cómo debe ser el aviso.** La primera capa, la que ve la persona al llegar, tiene que ofrecer:

1. Un botón para **aceptar** todas las cookies.
2. Un botón para **rechazarlas**, similar al anterior: si hay un botón de aceptar, tiene que haber uno de rechazar.
3. Un botón o enlace claramente visible para **configurar**, que lleva a un panel donde elegir por finalidad. No tiene por qué ser igual de destacado que los otros dos.

Y además:

- **Nada de cookies no exceptuadas antes de la decisión.** El consentimiento es previo.
- **Revocar tiene que ser tan fácil como aceptar**, con acceso permanente a la configuración.
- **Guarda la decisión** y no la vuelvas a pedir en cada visita. La AEPD considera buena práctica que no dure más de 24 meses.
- **Si cambian los fines o los terceros**, hay que actualizar la política y pedir una decisión nueva.

## Implementarlo: bloquear antes de consentir

El error más habitual no está en el aviso, sino en que las herramientas ya se han cargado cuando aparece. Lo correcto es **no ejecutar el script** hasta tener el consentimiento:

```js
function cargarAnalitica() {
  const s = document.createElement('script');
  s.src = 'https://analitica.ejemplo.com/script.js';
  s.async = true;
  document.head.appendChild(s);
}

// Se ejecuta al cargar la página y cada vez que cambia la decisión
function aplicarConsentimiento(decision) {
  if (decision.analitica) cargarAnalitica();
}
```

El script de análisis no existe en la página hasta que `decision.analitica` es verdadero. Con `Consent Mode v2` de Google, la herramienta se carga siempre, pero en modo restringido, y se le avisa del estado con señales:

```js
// Antes de cargar las etiquetas: todo denegado por defecto
gtag('consent', 'default', {
  ad_storage: 'denied',
  analytics_storage: 'denied',
  ad_user_data: 'denied',
  ad_personalization: 'denied',
  wait_for_update: 500,
});

// Cuando la persona acepta el análisis:
gtag('consent', 'update', { analytics_storage: 'granted' });
```

Son las cuatro señales de la versión 2 del modo de consentimiento; `ad_user_data` y `ad_personalization` son las que Google pide para la publicidad en el Espacio Económico Europeo. Mientras estén en `denied`, las etiquetas de Google ajustan lo que guardan. Aun así, el modo de consentimiento **no sustituye** al aviso: es la forma de comunicar la decisión, no de tomarla.

Para comprobar que funciona, abre las herramientas de desarrollo del navegador, pestaña *Application* → *Cookies*, y recarga con la caché vacía **sin pulsar nada**. Solo deben aparecer cookies exceptuadas. Después pulsa «Rechazar» y repite: no debe haber nuevas.

## Errores frecuentes

| Error | Por qué falla |
|---|---|
| Cargar la herramienta de análisis y mostrar el aviso a la vez | El consentimiento tiene que ser previo; para cuando se decide, ya se han guardado cookies |
| Solo un botón de «Aceptar», con «Rechazar» escondido en un panel | La AEPD pide un botón de rechazar similar al de aceptar en la primera capa |
| Casillas premarcadas en el panel de configuración | El consentimiento no puede presumirse |
| Aviso que desaparece sin dejar forma de cambiar la decisión | Revocar debe ser tan fácil como aceptar |
| Olvidar las cookies que añade un tercero incrustado | Un vídeo o un mapa puede crear cookies propias; hay que inventariarlas |
| Dar por exentas las cookies de análisis | No lo están, por poco intrusivas que sean |

---

## Buenas prácticas avanzadas

- **Haz un inventario de cookies y repítelo cada cierto tiempo.** Cada tercero nuevo (un widget, una fuente, un vídeo) puede añadir cookies sin que el código propio cambie. Un escáner periódico, o el listado de *Application → Cookies* en cada versión, destapa las que la política no recoge.
- **Guarda la prueba del consentimiento.** Anota, por cada decisión, la fecha, la versión de la política mostrada y el resultado. Si tienes que demostrar que se pidió bien, esa es la evidencia.
- **Automatiza la comprobación "sin consentimiento, sin cookies".** Una prueba de extremo a extremo que cargue la página, no pulse nada y falle si aparece una cookie no exceptuada evita regresiones cada vez que alguien añade una etiqueta:

  ```ts
  test('sin consentimiento no hay cookies no exceptuadas', async ({ page, context }) => {
    await page.goto('https://tienda.ejemplo.com');
    const nombres = (await context.cookies()).map(c => c.name);
    expect(nombres.filter(n => !['sessionId', 'idioma'].includes(n))).toEqual([]);
  });
  ```

- **Incrusta contenido de terceros sin cookies.** Muchos vídeos y mapas tienen un modo que no guarda nada hasta que se pulsa (por ejemplo, `youtube-nocookie.com`), o puedes mostrar una imagen y cargar el reproductor solo al pulsar.
- **Fija `SameSite=Lax`, `Secure` y `HttpOnly` por defecto en las cookies propias**, y cámbialos solo con un motivo. Las cookies de sesión sin `HttpOnly` son la ruta más corta del XSS al robo de sesión.
- **Pide una decisión nueva cuando cambian los fines o los terceros.** Si se añade una herramienta de publicidad a una web que solo medía visitas, el consentimiento anterior no cubre ese fin nuevo.

## Documentación oficial

- [Guía sobre el uso de las cookies (AEPD)](https://www.aepd.es/guias/guia-cookies.pdf) — la fuente para España: tipos de cookies, cuáles están exentas, cómo debe ser el aviso por capas y cuánto dura el consentimiento. Empieza por los apartados 2 (tipos) y 3.2 (cómo obtener el consentimiento).
- [Set-Cookie (MDN)](https://developer.mozilla.org/es/docs/Web/HTTP/Reference/Headers/Set-Cookie) — la referencia técnica de la cabecera y de todos sus atributos, con la compatibilidad por navegador.
- [Configurar el modo de consentimiento en sitios web (Google)](https://developers.google.com/tag-platform/security/guides/consent) — cómo declarar las señales de consentimiento a las etiquetas de Google.

## Recursos didácticos

- [Directrices 05/2020 sobre el consentimiento (CEPD)](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-052020-consent-under-regulation-2016679_en) — cómo entienden las autoridades europeas que debe ser un consentimiento válido, con ejemplos de avisos que no lo son.
- El panel *Application → Cookies* de las herramientas de desarrollo de cualquier navegador: es la forma más rápida de ver qué guarda una web antes y después de aceptar.

---

*En resumen: una cookie es sencilla de crear y fácil de usar mal; lo que decide si cumples es qué guardas, antes de qué decisión y si la persona puede cambiarla tan fácil como la tomó.*
