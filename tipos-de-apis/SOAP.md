# SOAP

## ¿Qué es?

**SOAP** (*Simple Object Access Protocol*) es un protocolo veterano para construir APIs en el que los mensajes se intercambian siempre en un formato **XML** muy estructurado y con reglas estrictas. Es más rígido y ceremonioso que REST, pero ofrece garantías formales que aún se valoran en ciertos sectores.

## ¿Por qué existe?

SOAP nació para que sistemas muy distintos (y a menudo de grandes empresas) pudieran comunicarse con un **contrato estricto** y características avanzadas integradas: seguridad a nivel de mensaje, transacciones, garantías de entrega. Donde REST confía en las convenciones, SOAP lo especifica todo formalmente.

> Piensa en la diferencia entre un acuerdo de palabra y un contrato notarial de 40 páginas. REST es ágil y flexible; SOAP es el contrato notarial: pesado de leer, pero deja cada detalle por escrito y con sello.

Hoy se considera "antiguo" para la mayoría de usos, pero sigue muy vivo en banca, seguros, administraciones públicas y sistemas heredados (*legacy*) que llevan décadas funcionando.

## ¿Cuándo y para qué se usa?

Lo encontrarás sobre todo al **integrarte con sistemas existentes**: pasarelas bancarias, servicios de aseguradoras, organismos oficiales o software empresarial antiguo. Rara vez elegirías SOAP para un proyecto nuevo, pero sí tendrás que consumirlo si el sistema con el que hablas solo ofrece SOAP.

## Lo mínimo que necesitas saber

**1. Todo mensaje es un "sobre" XML**

Cada petición y respuesta se envuelve en una estructura fija: el *Envelope*, con su *Header* (metadatos) y su *Body* (los datos).

```xml
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <GetProduct>
      <productId>42</productId>
    </GetProduct>
  </soap:Body>
</soap:Envelope>
```

**2. El contrato se describe en un WSDL**

Un fichero **WSDL** (XML) define formalmente qué operaciones existen, qué parámetros aceptan y qué devuelven. Las herramientas pueden generar código cliente a partir de él.

```xml
<wsdl:operation name="GetProduct">
  <wsdl:input message="GetProductRequest"/>
  <wsdl:output message="GetProductResponse"/>
</wsdl:operation>
```

**3. Suele ir por HTTP, pero no depende de él**

A diferencia de REST, SOAP no usa los verbos HTTP: casi siempre va por `POST` y puede funcionar incluso sobre otros transportes. El significado está en el XML, no en el protocolo.

**4. Trae características formales "de fábrica"**

El estándar **WS-Security** y otros añaden firma, cifrado a nivel de mensaje y transacciones, sin tener que improvisarlos.

**5. El cliente se genera desde el WSDL**

```csharp
// A partir del WSDL, las herramientas generan un cliente tipado
var client = new ProductServiceClient();
var product = await client.GetProductAsync(productId: 42);
```

## Lo que NO hace

- **No es ligero** — el XML envuelto es verboso; consume más ancho de banda y CPU que JSON.
- **No es cómodo a mano** — leer y construir sobres XML manualmente es tedioso; casi siempre se usan herramientas.
- **No aprovecha HTTP como REST** — ignora verbos y códigos de estado; el resultado viaja dentro del XML.
- **No suele ser la elección para proyectos nuevos** — salvo que un sistema con el que integras lo imponga.

---

*En resumen: SOAP es el contrato notarial de las APIs —XML estricto, WSDL formal y seguridad de fábrica— pesado pero firme, todavía imprescindible al integrar con banca, seguros y sistemas heredados.*
