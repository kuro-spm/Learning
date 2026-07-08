# Autenticación vs Autorización

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Dos preguntas distintas que toda aplicación con usuarios tiene que responder: **autenticación** (*AuthN*) es "¿quién eres?", y **autorización** (*AuthZ*) es "¿qué puedes hacer?". Son procesos secuenciales pero independientes: primero se verifica la identidad y después, con esa identidad ya confirmada, se decide qué acciones o recursos están permitidos.

## ¿Por qué existe?

La confusión entre ambos conceptos es tan frecuente que tiene nombre propio en muchos foros: "AuthN vs AuthZ". Ocurre porque en la práctica van pegados (te logueas y, acto seguido, la aplicación decide qué ves) y porque en inglés ambas palabras empiezan igual. Pero mezclarlas en el diseño de una aplicación lleva a errores de seguridad: dar por hecho que "si está autenticado, puede hacer esto" es exactamente el fallo que abre la puerta a que cualquier usuario logueado acceda a datos o acciones que no le corresponden.

> Piensa en un aeropuerto: enseñar tu DNI en el control de seguridad es autenticación (confirman que eres quien dices ser). La tarjeta de embarque de tu vuelo y asiento concretos es autorización (determina a qué avión y qué asiento puedes acceder). Tener DNI válido no te da derecho a subir a cualquier avión.

## ¿Cuándo y para qué se usa?

Aparecen juntas en cualquier aplicación con distintos niveles de acceso:

- **Una tienda online**: cualquier persona autenticada puede ver su propio historial de pedidos (autorización a nivel de "es su propio recurso"), pero solo el rol "Administrador" puede ver el panel de ventas de todos los clientes.
- **Un blog con comentarios**: para comentar hace falta estar autenticada; para editar o borrar un comentario ajeno hace falta además ser autorizada como moderadora.
- **Una app de tareas compartida**: todas las personas del equipo se autentican igual, pero solo quien creó una tarea (o quien tiene el rol de gestora del proyecto) está autorizada a eliminarla.

## Lo mínimo que necesitas saber

**1. Las siglas AuthN y AuthZ**

Verás estas abreviaturas en documentación, nombres de middleware y de librerías (`app.UseAuthentication()`, `app.UseAuthorization()`). AuthN siempre va antes: no tiene sentido decidir permisos sobre una identidad que aún no se ha verificado.

**2. Los códigos de estado HTTP que corresponden a cada fallo**

```csharp
// 401 Unauthorized: no sabemos quién eres (falta autenticación o el token no es válido)
// 403 Forbidden: sabemos quién eres, pero no tienes permiso para esto (falla la autorización)

[HttpGet("pedidos/{id}")]
[Authorize] // exige autenticación: si falla, 401
public IActionResult ObtenerPedido(int id)
{
    var pedido = _pedidos.Buscar(id);
    if (pedido.ClienteId != UsuarioActualId)
        return Forbid(); // autenticada, pero no autorizada: 403

    return Ok(pedido);
}
```

**3. Dónde vive cada uno en una API típica de ASP.NET Core**

```csharp
app.UseAuthentication(); // identifica a la usuaria a partir del token/cookie
app.UseAuthorization();  // decide si esa identidad puede acceder al endpoint solicitado

// La autenticación resuelve el "quién": rellena HttpContext.User
// La autorización se declara por endpoint o controlador:
[Authorize(Roles = "Administrador")]
[HttpGet("panel-ventas")]
public IActionResult PanelDeVentas() => Ok(_ventas.ObtenerResumen());
```

**4. Un flujo completo, de principio a fin**

1. La usuaria envía sus credenciales (login) → el servidor **autentica** y emite un token o cookie.
2. En cada request posterior, ese token identifica a la usuaria → sigue siendo autenticación.
3. Antes de ejecutar la acción solicitada, el servidor comprueba rol, permisos o propiedad del recurso → eso es **autorización**, y ocurre en cada request, no solo en el login.

## Lo que NO hace

- **Estar autenticada no implica estar autorizada** — son comprobaciones independientes; hay que declarar explícitamente qué puede hacer cada identidad.
- **La autorización no identifica a nadie** — si no hay una autenticación previa fiable, cualquier sistema de roles se construye sobre una identidad que podría ser falsa.
- **Ninguna de las dos sustituye a la validación de datos** — que una usuaria esté autorizada a editar "sus" pedidos no exime de comprobar que el pedido que intenta editar es realmente suyo (ver el ejemplo del punto 2).

## Buenas prácticas avanzadas

- **Diseña en "denegar por defecto"** — cada endpoint nuevo debería nacer bloqueado (`[Authorize]` global o política restrictiva) y abrirse explícitamente, nunca al revés. Olvidar un `[Authorize]` en un endpoint nuevo es un error de omisión mucho más probable que añadir uno de más.
- **No reveles con un 404 lo que en realidad es un 403 mal entendido, ni al revés** — decide conscientemente si un recurso al que no tienes acceso debe parecer "no autorizado" (403, confirma que existe) o "inexistente" (404, oculta su existencia). Mezclarlos sin criterio filtra información o genera una UX confusa.
- **La autorización se revisa en cada request, no solo al construir el token** — si los permisos de una usuaria cambian (se le revoca un rol), un token de larga duración emitido antes seguirá "autenticando" correctamente; la autorización basada en datos actualizados en servidor (no solo en claims del token) es lo que corta el acceso a tiempo.
- **Aplica el principio de mínimo privilegio también a nivel de diseño de API** — no crees un único rol "Usuaria" con acceso a todo lo no-administrativo; separa permisos por acción (leer, escribir, borrar) para poder combinarlos según el caso real, en vez de sobre-conceder por comodidad.
- **Exige reautenticación para acciones sensibles** — que una sesión lleve autenticada seis horas no significa que deba bastar para cambiar la contraseña o eliminar la cuenta; pedir de nuevo la contraseña (*step-up authentication*) reduce el daño si el dispositivo quedó desatendido.

## Recursos didácticos divertidos

Para memorizar la diferencia entre los códigos 401 y 403 (y el resto de la familia HTTP) de forma visual, [http.cat](https://http.cat) ilustra cada código de estado con la foto de un gato.

---

*En resumen: la autenticación confirma quién eres, la autorización decide qué puedes hacer — confundirlas en el diseño es la fuente número uno de fallos de control de acceso.*
