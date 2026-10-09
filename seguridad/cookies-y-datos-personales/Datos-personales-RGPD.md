# Datos personales y RGPD

> Esta guía resume el Reglamento General de Protección de Datos (RGPD) y su aplicación en España desde el punto de vista de quien desarrolla. No es asesoramiento legal: ante una duda concreta, consulta a quien lleve la privacidad de la organización.

## ¿Qué es?

El RGPD es el reglamento de la Unión Europea que protege a las personas cuando alguien trata información sobre ellas: define qué es un dato personal, qué obligaciones tiene quien lo usa y qué derechos tiene la persona a la que pertenece. En España lo complementa la Ley Orgánica 3/2018 (LOPDGDD), y la autoridad que vigila su cumplimiento es la AEPD.

## ¿Por qué existe?

Cuando una web guarda un correo, una dirección IP o un historial de compras, esa información dice cosas sobre una persona concreta, y esa persona normalmente no controla qué pasa con ella después. El RGPD pone el control del lado de la persona: para tratar sus datos hace falta una razón válida, hay que decirle para qué, solo se guarda lo necesario y puede pedir verlos, corregirlos o borrarlos.

> Si ya conoces el principio de mínimo privilegio en seguridad, piensa en el RGPD como su equivalente para los datos: acceder solo a lo que hace falta, solo durante el tiempo que hace falta y dejando claro quién responde.

## ¿Cuándo y para qué se usa?

Aplica en cuanto una web o una app trata datos de personas que están en la Unión Europea, aunque la empresa esté fuera. En la práctica, casi siempre:

- Un **formulario de registro o de contacto** (nombre, correo, teléfono).
- Las **cuentas de usuario** y el historial de pedidos de una tienda online.
- La **analítica web** y la publicidad.
- Los **registros del servidor** (logs), que guardan direcciones IP.
- Un servicio de terceros que recibe datos de tus visitantes (correo, mapas, vídeo, chat).

---

## Qué es un dato personal

Es cualquier información sobre una persona identificada o que se pueda identificar, directa o indirectamente. El RGPD cita expresamente los **identificadores en línea**, como las direcciones IP y los identificadores de las cookies.

| Dato | ¿Es personal? |
|---|---|
| Nombre y apellidos, correo con nombre (`ana.perez@…`) | Sí |
| Dirección IP | Sí, porque puede asociarse a una persona |
| Identificador guardado en una cookie | Sí, si permite reconocer a alguien |
| Número de pedido unido al nombre de quien lo hizo | Sí |
| Estadística agregada ("el 12 % de las visitas viene de móvil") | No, si no permite identificar a nadie |
| Datos anonimizados de verdad, sin forma de revertirlos | No |

Los datos de **categorías especiales** (salud, origen étnico, opiniones políticas, creencias, vida sexual, datos biométricos para identificar) tienen una protección reforzada, y en general su tratamiento está prohibido salvo que concurra una excepción concreta.

**Tratar** datos es cualquier operación con ellos: recogerlos, guardarlos, consultarlos, enviarlos o borrarlos.

## Quién es quién

| Figura | Qué es | Ejemplo en una tienda online |
|---|---|---|
| **Responsable** | Decide para qué y cómo se tratan los datos | La empresa que vende |
| **Encargado** | Trata datos por cuenta del responsable, bajo un contrato | El proveedor de correo o el servicio de alojamiento |
| **Persona interesada** | A quien pertenecen los datos | Quien compra |

Si un proveedor trata los datos por tu cuenta, hace falta un **contrato de encargo** (art. 28) que fije qué puede hacer con ellos.

## Los principios

Todo el reglamento cuelga de siete principios (art. 5). Son lo que mejor ayuda a decidir ante un caso que nadie ha previsto:

| Principio | Lo que significa al programar |
|---|---|
| Licitud, lealtad y transparencia | Hay una base legal y la persona sabe qué pasa con sus datos |
| Limitación de la finalidad | No uses para otra cosa lo que recogiste para una |
| Minimización | Pide solo los campos necesarios |
| Exactitud | Permite corregir los datos y no guardes datos obsoletos |
| Limitación del plazo | Cada dato tiene fecha de borrado |
| Integridad y confidencialidad | Controla accesos y cifra lo que lo merezca |
| Responsabilidad proactiva | Tienes que poder demostrar que cumples |

## La base jurídica: por qué puedes tratar un dato

Para cada tratamiento tiene que existir una de las seis bases del art. 6. Las que aparecen casi siempre en una web:

| Base | Cuándo encaja | Ejemplo |
|---|---|---|
| **Consentimiento** | La persona acepta de forma libre e informada, y puede retirarlo | Suscribirse a un boletín, cookies de análisis |
| **Contrato** | El dato es necesario para cumplir lo que la persona ha contratado | La dirección para enviar un pedido |
| **Obligación legal** | Una ley obliga a guardar el dato | Facturas |
| **Interés legítimo** | Hay un interés del responsable que no perjudica los derechos de la persona; exige hacer una ponderación | Seguridad del sistema o prevención de fraude |

Tiene que haber **una base por finalidad**. El mismo correo puede tratarse por contrato (avisos del pedido) y, solo si la persona lo acepta aparte, por consentimiento (publicidad). El consentimiento de cookies y el de un formulario son decisiones distintas.

## Los derechos de las personas

Los artículos 15 a 22 dan a la persona estos derechos, y el responsable debe poder atenderlos **en un mes** como norma general (prorrogable en casos complejos):

- **Acceso**: saber qué datos se tienen y para qué.
- **Rectificación**: corregir los inexactos.
- **Supresión** ("derecho al olvido"): borrarlos cuando ya no hay base para guardarlos.
- **Limitación**: pedir que se congelen mientras se resuelve una disputa.
- **Portabilidad**: recibirlos en un formato estructurado.
- **Oposición**: oponerse a ciertos tratamientos, entre ellos el marketing directo.
- A no ser objeto de **decisiones automatizadas** con efectos relevantes.

En código, esto significa que los datos de una persona deben poder **localizarse, exportarse y borrarse** sin excavar. Si están repartidos por seis tablas, un fichero de logs y una hoja de cálculo, un derecho de supresión se convierte en un proyecto.

## Informar: la política de privacidad

Al recoger datos hay que explicar (arts. 13 y 14) quién es el responsable, para qué se usan, con qué base legal, a quién se comunican, cuánto tiempo se guardan y cómo ejercer los derechos. La AEPD recomienda presentarlo **por capas**: un resumen breve y visible donde se recogen los datos (junto al formulario) y la política completa a un clic.

## Ejemplo guiado: un formulario de contacto

Un formulario sencillo aplica casi todos los principios a la vez. Esta es la versión que no los respeta:

```html
<!-- ❌ Pide de más, premarca el consentimiento y no informa -->
<form>
  <input name="nombre" required>
  <input name="correo" required>
  <input name="telefono" required>
  <input name="fechaNacimiento" required>
  <textarea name="mensaje"></textarea>
  <label><input type="checkbox" name="boletin" checked> Quiero recibir novedades</label>
  <button>Enviar</button>
</form>
```

Y esta, la que sí:

```html
<!-- ✅ Solo lo necesario, consentimiento aparte, información a la vista -->
<form>
  <label for="correo">Correo</label>
  <input id="correo" name="correo" type="email" required>

  <label for="mensaje">Mensaje</label>
  <textarea id="mensaje" name="mensaje" required></textarea>

  <label>
    <input type="checkbox" name="boletin">
    Quiero recibir novedades por correo (opcional)
  </label>

  <p>Usaremos tus datos para responder a tu consulta.
     Más información en la <a href="/privacidad">política de privacidad</a>.</p>
  <button>Enviar</button>
</form>
```

Qué cambia: el teléfono y la fecha de nacimiento ya no se piden (**minimización**), la casilla del boletín no viene marcada y es una decisión **separada** de la consulta (consentimiento libre), y la persona ve la finalidad junto al botón (**transparencia**).

El lado del servidor tiene que completar lo que el formulario promete. Si se dice que los mensajes se atienden y se guardan un tiempo, tiene que haber un borrado automático:

```csharp
// Borra las consultas con más de 12 meses; se ejecuta cada noche
var limite = DateTime.UtcNow.AddMonths(-12);
var borradas = await db.ConsultasContacto
    .Where(c => c.CreadaEn < limite)
    .ExecuteDeleteAsync();
```

Esto elimina de golpe en la base de datos todas las filas anteriores a la fecha límite, sin cargarlas en memoria, y devuelve cuántas se borraron.

## Las direcciones IP en los registros

Los logs del servidor suelen guardar la IP completa de cada petición, y esa IP es un dato personal. Si no la necesitas completa, **acórtala** al registrar:

```csharp
// 203.0.113.57 → 203.0.113.0
static string AnonimizarIp(IPAddress ip)
{
    var bytes = ip.GetAddressBytes();
    if (bytes.Length == 4) bytes[3] = 0;          // IPv4: último octeto
    else for (var i = 6; i < 16; i++) bytes[i] = 0; // IPv6: se conservan los primeros 48 bits
    return new IPAddress(bytes).ToString();
}
```

Sigue siendo útil para detectar tráfico anómalo por zona y deja de identificar a una persona concreta. Si necesitas la IP completa para seguridad, guárdala con un plazo corto.

## Terceros y transferencias fuera de la Unión Europea

Cada servicio externo que recibe datos de tus visitantes (analítica, correo, mapas, fuentes, chat) es un destinatario y probablemente un encargado:

- **Inventaría** a qué terceros llegan datos y cuáles. Una fuente cargada desde un servidor ajeno o un mapa incrustado reciben la IP de cada visitante.
- **Firma el contrato de encargo** con los que traten datos por tu cuenta.
- Si el tercero está **fuera del Espacio Económico Europeo**, necesitas un mecanismo que lo permita: una decisión de adecuación de la Comisión Europea para ese país o cláusulas contractuales tipo. Comprueba cuál aplica a cada proveedor, porque cambia con el tiempo.

## Brechas y otras obligaciones

- **Brechas de seguridad**: si se pierden, filtran o se alteran datos personales y hay riesgo para las personas, hay que notificarlo a la autoridad **en 72 horas** desde que se conoce (art. 33). Tener un procedimiento escrito antes de que ocurra es lo que hace posible el plazo.
- **Registro de actividades de tratamiento** (art. 30): una lista de qué datos se tratan, para qué, con qué base y cuánto tiempo. Tiene excepciones para organizaciones pequeñas, pero hacerlo sirve igualmente como inventario.
- **Evaluación de impacto** (art. 35): obligatoria cuando un tratamiento tiene un riesgo alto, por ejemplo seguimiento a gran escala.
- **Sanciones**: pueden llegar a 20 millones de euros o el 4 % de la facturación anual, según cuál sea mayor.

---

## Buenas prácticas avanzadas

- **Empieza por el inventario y no por el texto legal.** Antes de redactar una política, haz la lista de qué datos entran, por dónde, dónde se guardan y a quién se envían. Es lo que permite atender un derecho de supresión, detectar un tercero olvidado o decidir qué base legal aplica.
- **Diseña el borrado desde el principio.** Un plazo de conservación escrito en la política que nadie ha implementado es un incumplimiento. Pon cada dato con su fecha de caducidad y automatiza el borrado, como en el ejemplo de las consultas.
- **Separa por finalidad.** Guarda el consentimiento para publicidad como un dato independiente del pedido, con fecha, texto mostrado y resultado. Así retirarlo no toca nada más, y puedes demostrar cuándo y cómo se dio.
- **Pseudonimiza lo que puedas.** Sustituir el correo por un identificador interno en las tablas de análisis, y guardar la correspondencia en un sitio aparte con acceso restringido, reduce el daño de una filtración y facilita borrar a una persona.
- **Trata las peticiones de derechos como un proceso con plazo.** Un mes empieza a contar desde que llega la solicitud, por cualquier canal. Define quién la recibe, cómo se verifica la identidad de quien pregunta y dónde se registra.
- **Ensaya la brecha.** Con 72 horas de plazo, la primera vez que se decide quién avisa a quién no puede ser durante el incidente.

## Documentación oficial

- [Reglamento (UE) 2016/679, texto completo (EUR-Lex)](https://eur-lex.europa.eu/eli/reg/2016/679/oj) — la fuente normativa. Los artículos 5, 6, 12 a 22 y 33 son los que más se consultan al desarrollar; los considerandos explican la intención de cada regla.
- [Sitio de la AEPD](https://www.aepd.es/) — la autoridad española: guías, modelos y resoluciones. Su sección de guías y herramientas es el punto de entrada para casos concretos.
- [Directrices 05/2020 sobre el consentimiento (CEPD)](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-052020-consent-under-regulation-2016679_en) — cuándo un consentimiento es válido, con ejemplos.

## Recursos didácticos

- [Facilita RGPD (AEPD)](https://www.aepd.es/guias-y-herramientas/herramientas/facilita-rgpd) — herramienta gratuita que, con unas preguntas, orienta a una organización pequeña sobre qué obligaciones le aplican.
- [Cookies y consentimiento](Cookies-y-consentimiento.md) — la mitad técnica de esta misma regulación: qué se puede guardar en el navegador, y cuándo hace falta pedir permiso.

---

*En resumen: un dato personal es todo lo que permite reconocer a alguien, incluida su IP; recoge lo mínimo, para algo que puedas explicar, guárdalo el tiempo justo y deja que la persona pueda ver, corregir y borrar lo suyo.*
