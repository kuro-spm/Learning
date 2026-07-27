# Credenciales en Llamadas Salientes

## ¿Qué es?

El conjunto de prácticas para que la credencial que tu servidor envía al llamar a una API externa (un token, una `api-key`) **no se filtre por el camino**: ni a los logs, ni a un host equivocado tras un redirect, ni a una URL de un tercero (un almacenamiento en la nube, un CDN).

## ¿Por qué existe?

Cuando tu backend actúa de **cliente** de otra API con autenticación, manda un secreto en una cabecera (normalmente `Authorization: Bearer ...` o `api-key: ...`). El secreto puede estar perfectamente guardado *en reposo* (en user-secrets o un *vault*) y aun así **filtrarse en el momento de usarlo**, por caminos que no saltan a la vista:

- un log que serializa la petición entera con sus cabeceras,
- una respuesta `301/302` que reenvía tu `Authorization` a **otro dominio**,
- la descarga de un fichero desde una URL firmada, a la que le cuelas tu token aunque no haga falta.

> Piensa en el secreto como la llave de tu casa: no basta con guardarla bien en un cajón (reposo); también hay que evitar enseñársela a cualquiera que te pare por la calle (uso).

## ¿Cuándo y para qué se usa?

Siempre que tu servidor consuma una API externa autenticada: cobrar con una pasarela de pago, subir un fichero a un almacenamiento en la nube, pedirle a un proveedor que genere o procese algo. El riesgo crece cuando el flujo tiene **varios saltos**: te autenticas contra la API del proveedor, esta te devuelve una URL firmada de un bucket, y luego descargas el resultado de un CDN. Cada salto es una ocasión de filtrar la credencial a un dominio que no es el tuyo ni el del proveedor.

## Lo mínimo que necesitas saber

**1. Nunca loguees el secreto — loguea el hecho y el destinatario**

```csharp
// MAL: vuelca la cabecera Authorization al log
logger.LogInformation("Llamando al proveedor: {Request}", request);

// BIEN: registra qué pasó y contra quién, nunca la credencial ni el body
logger.LogInformation("POST a {Host}: {Status}", request.RequestUri!.Host, (int)response.StatusCode);
```

**2. El token, solo al host previsto**

No reutilices un `HttpClient` que ya lleva el `Authorization` puesto para llamar a un dominio distinto. Un cliente autenticado sirve a **un** proveedor.

**3. Desactiva el auto-redirect (`AllowAutoRedirect = false`)**

Por defecto, ante un `3xx` el cliente sigue la redirección **reenviando tus cabeceras**, incluida `Authorization`, aunque el destino sea otro dominio. Desactívalo y trata el redirect a mano si de verdad lo esperas.

```csharp
var handler = new HttpClientHandler { AllowAutoRedirect = false };
using var client = new HttpClient(handler);
```

**4. Cliente "pelado" (sin auth) para URLs de terceros**

Para descargar de una URL firmada o de un CDN, usa un cliente **sin** cabecera de autenticación: la autorización ya viaja en la propia URL (la firma en la query string); añadir tu Bearer solo lo expone en un dominio ajeno.

```csharp
// La presigned URL ya está autorizada; este cliente NO lleva Authorization
using var descarga = new HttpClient();
var bytes = await descarga.GetByteArrayAsync(urlFirmada);
```

**5. Exige `https` y pon un timeout**

Nunca envíes un secreto por `http` (viajaría en claro). Y fija un `Timeout`: una llamada colgada a un tercero no debe agotar los recursos de tu servidor.

```csharp
if (endpoint.Scheme != Uri.UriSchemeHttps)
    throw new InvalidOperationException("El proveedor debe usar https.");
client.Timeout = TimeSpan.FromSeconds(30);
```

## Lo que NO hace

- **No sustituye a guardar bien el secreto en reposo** — esto protege el secreto *en uso*; de dónde sale la credencial es otro tema (ver [Gestión de secretos en desarrollo](../gestion-de-secretos-en-desarrollo/README.md)).
- **No cifra el contenido** — de la confidencialidad en tránsito se encarga TLS (`https`); estas prácticas evitan mandar el secreto al sitio equivocado, no reemplazan al cifrado del canal.
- **No valida qué puede hacer el token** — que la credencial no se filtre no dice nada de sus permisos; eso es autorización.

## Buenas prácticas avanzadas

- **`AllowAutoRedirect = false` por defecto en todo cliente autenticado** — el reenvío de `Authorization` a otro dominio en un redirect es una fuga real y silenciosa que casi nadie desactiva porque el cliente HTTP "simplemente funciona". Actívalo solo si controlas a dónde redirige.
- **Un `HttpClient` por integración, configurado, y el "pelado" aparte** — con una fábrica de clientes (p. ej. `IHttpClientFactory`), da a cada proveedor su cliente con su handler y credencial, y reserva un cliente sin auth para descargas de terceros. Compartir el cliente autenticado "para no crear otro" es justo como se cuela el token donde no debe.
- **Redacta las cabeceras sensibles en el logging estructurado** — si registras peticiones o respuestas, pon un filtro que enmascare `Authorization`, `api-key` y `Set-Cookie`. Un log a nivel *debug* olvidado en producción es la vía de fuga más habitual.
- **La firma va en la URL y el token en la cabecera: no los mezcles** — una URL firmada ya está autorizada por sí misma; añadirle tu Bearer no aporta nada y lo expone en el dominio del tercero y en sus logs.
- **Valida esquema y host antes de adjuntar la credencial** — si el endpoint del proveedor viene de configuración, comprueba que es `https` y, cuando puedas, que el host es el esperado *antes* de poner la cabecera con el secreto; así una config manipulada no desvía tu token a otro sitio.

## Recursos didácticos

- [webhook.site](https://webhook.site) — te da una URL temporal que muestra **exactamente** qué cabeceras y cuerpo recibe. Perfecto para comprobar en vivo si tu cliente está filtrando un `Authorization` (por ejemplo, apuntando ahí un redirect de prueba).
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) — qué registrar y, sobre todo, qué **no** registrar nunca.

---

*En resumen: guardar bien un secreto no sirve de nada si luego lo filtras al usarlo — no lo loguees, no dejes que un redirect lo reenvíe a otro dominio, y no se lo cuelgues a las URLs de terceros.*
