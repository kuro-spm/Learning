# JWT + Refresh vs Tokens opacos

## ¿Qué es?

Son las **dos formas concretas de implementar un token** de sesión, y representan una decisión de diseño que se repite en casi cualquier sistema de autenticación. Un [JWT + Refresh](JWT-Refresh.md) es un token *autocontenido* (o *de valor*): lleva la información firmada dentro. Un [token opaco](Tokens-Opacos.md) es un token *de referencia*: no lleva nada y apunta a un estado guardado en el servidor. Esta ficha compara ambos frente a frente.

> Esta es la versión "de implementación" de la disyuntiva. Si buscas la distinción conceptual de más alto nivel —guardar el estado en el servidor o llevarlo contigo—, esa está en [Sesiones vs Tokens](Sesiones-vs-Tokens.md).

## ¿Por qué existe?

Ambos modelos resuelven lo mismo: recordar que ya te autenticaste sin volver a pedir credenciales. La diferencia es *dónde vive la verdad*. En el modelo autocontenido, la verdad viaja en el token y se verifica con una firma; el servidor no necesita recordar nada. En el modelo opaco, la verdad vive en el servidor y el token es solo el puntero.

Esa única decisión arrastra todo lo demás: la capacidad de revocar, la facilidad de escalar, cuánta información se filtra y cuánto cuesta validar cada request. No hay un ganador universal; hay un conjunto de compromisos que conviene entender antes de elegir.

## ¿Cuándo y para qué se usa?

- **Elige JWT + Refresh** cuando el desacoplamiento y la escalabilidad mandan: microservicios que se pasan la identidad sin consultar una base central, APIs con muchas instancias, o clientes muy distribuidos donde ahorrar una consulta por request importa.
- **Elige tokens opacos** cuando el control manda: sistemas que necesitan revocación inmediata (banca, salud), aplicaciones web con sesión de servidor clásica, o cuando no quieres que ninguna información viaje fuera de tu infraestructura.
- **En la práctica, muchos sistemas mezclan**: access token JWT de vida corta para las requests + refresh token opaco y revocable en el servidor. Justo el modelo de la ficha de [JWT + Refresh](JWT-Refresh.md).

## Lo mínimo que necesitas saber

**1. La tabla frente a frente**

| Dimensión | JWT + Refresh (autocontenido) | Token opaco (de referencia) |
|---|---|---|
| Dónde vive la información | En el token, firmada | En el servidor |
| Validación por request | Local: verificar la firma | Consulta al almacén / introspección |
| Revocación del access | No (hasta que expira) | Inmediata (borras la sesión) |
| Escalado multi-instancia | Trivial: sin estado compartido | Requiere almacén común (Redis/BD) |
| Filtración de datos | El payload es legible por cualquiera | Nula: no hay nada que leer |
| Tamaño del token | Grande (crece con los claims) | Diminuto (bytes aleatorios) |
| Estado en el servidor | Solo para el refresh | Para toda la sesión |

**2. El eje que lo decide casi todo: revocación vs statelessness**

Estos dos deseos están en tensión directa. El JWT es imbatible en escalado precisamente *porque* el servidor no recuerda nada; pero no recordar nada es justo lo que le impide revocar. El token opaco revoca al instante *porque* el servidor lo recuerda todo; pero recordarlo todo es lo que le obliga a consultar en cada request. No puedes maximizar ambos a la vez.

**3. Cómo cierra sesión cada uno**

```text
JWT autocontenido:  el token sigue vivo hasta su 'exp'. "Cerrar sesión" es,
                    en el mejor caso, borrarlo del cliente y esperar a que caduque.

Token opaco:        borras la entrada en el servidor → la siguiente request
                    falla al instante. Cierre de sesión real e inmediato.
```

**4. El punto medio que usa casi todo el mundo**

El patrón dominante no elige uno puro, sino que aprovecha cada modelo donde brilla: el **access token** es un JWT de vida corta (validación local, escala sin esfuerzo) y el **refresh token** es opaco y guardado en el servidor (revocable). Así, revocar el refresh corta la capacidad de renovar, y la vida corta del access limita el daño mientras tanto.

## Lo que NO hace

- **La comparación no dicta un ganador** — "JWT porque es moderno" es un error clásico; para un monolito con login clásico, la sesión opaca es más simple y te regala la revocación gratis.
- **Ninguno de los dos es más seguro por definición** — ambos se pueden robar (XSS, red comprometida); la seguridad depende de cómo se transmiten y guardan, no del modelo.
- **El JWT no es "sin estado de verdad" si necesitas revocar** — en cuanto añades lista de revocación o refresh tokens guardados, has reintroducido estado en el servidor; solo lo has movido de sitio.

## Buenas prácticas avanzadas

- **Deja que el requisito de revocación decida primero** — antes de comparar rendimiento o comodidad, pregúntate: ¿necesito cortar el acceso al instante? Si la respuesta es sí y es innegociable, o eliges opaco, o aceptas access tokens de vida muy corta con un refresh revocable; el JWT puro de larga duración queda descartado de entrada.
- **Si mezclas, no dejes JWT de vida larga por comodidad** — el híbrido solo funciona si el access token dura de verdad poco (minutos). Un JWT de 24 h "para no molestar con refrescos" reabre exactamente el agujero de revocación que intentabas cerrar.
- **No metas datos sensibles ni volátiles en un JWT** — como el payload es legible y no se puede revocar, un rol o un permiso incrustado en el token seguirá diciendo lo mismo aunque se lo hayas retirado a la usuaria hace un minuto. Los datos que cambian o que no deben verse piden el modelo opaco o una consulta aparte.
- **Mide el coste real de la consulta antes de descartar el opaco por rendimiento** — la "consulta por request" que asusta suele ser un `GET` a Redis de fracciones de milisegundo. En la mayoría de sistemas, esa latencia es despreciable frente a la ganancia de poder revocar; el escalado sin estado del JWT solo compensa de verdad a escalas muy grandes o entre servicios.

---

*En resumen: el JWT autocontenido cambia revocación por escalabilidad sin estado, y el token opaco hace justo lo contrario; entender esa tensión —y el híbrido access-JWT + refresh-opaco que la equilibra— es lo que te permite elegir con criterio en vez de por moda.*
