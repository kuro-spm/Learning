# Caching — Guía de tecnologías

Cómo evitar repetir trabajo costoso guardando temporalmente sus resultados, y cómo no pagar por ello sirviendo datos obsoletos sin que nadie se dé cuenta.

La colección va de lo conceptual a lo operativo: primero el vocabulario y los patrones, luego el almacén que se usa en la práctica, después el problema difícil de verdad —decidir cuándo un dato cacheado deja de valer— y por último dónde debe vivir la caché cuando la aplicación corre en varias instancias.

Todas las fichas comparten escenario: una tienda online con la base de datos `TiendaDB`, las tablas `Productos`, `Pedidos` y `Clientes`, claves de caché del estilo `producto:123` y `productos-destacados`, y una API desplegada en cuatro instancias detrás de un balanceador.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Caching](Caching.md) | El vocabulario y los patrones: *hit ratio*, TTL, expulsión, cache-aside, write-through, write-behind. Incluye el cálculo que justifica cachear y la lista de lo que **no** hay que cachear. Todo lo demás se apoya en esta ficha. |
| 2 | [Redis](Redis.md) | El almacén en memoria que se usa de verdad: sus estructuras de datos, TTL, por qué es de un solo hilo, qué ocurre cuando la memoria se llena, y `StackExchange.Redis` desde C#. |
| 3 | [Estrategias de invalidación](Estrategias-de-Invalidacion.md) | La parte difícil: TTL frente a invalidación activa, prefijos generacionales, avisar a las demás instancias por pub/sub, y las formas de sobrevivir a un *cache stampede*. |
| 4 | [Caché distribuida vs. local](Cache-Distribuida-vs-Local.md) | Dónde vive la caché: `IMemoryCache` frente a `IDistributedCache`, el coste real de la red, la caché en dos niveles y `HybridCache`. |

---

## Los tres números que conviene tener en la cabeza

| Pregunta | Respuesta | Dónde se calcula |
|---|---|---|
| ¿Cuánto ahorra subir el *hit ratio*? | Lo que paga la base de datos es el *miss ratio*: pasar del 80 % al 95 % de aciertos no mejora un 15 %, divide la carga por cuatro. | [Caching](Caching.md) |
| ¿Cuánto cuesta ir a la caché distribuida en lugar de a la local? | Tres o cuatro órdenes de magnitud: nanosegundos frente a entre medio milisegundo y un par, más la serialización. | [Caché distribuida vs. local](Cache-Distribuida-vs-Local.md) |
| ¿Qué pasa cuando caduca una clave muy solicitada? | Todas las peticiones que llegan mientras se recalcula golpean el origen a la vez, y son muchas más de las que parece. | [Estrategias de invalidación](Estrategias-de-Invalidacion.md) |

## Los errores que se pagan caros

| Síntoma | Causa habitual | Dónde se explica |
|---|---|---|
| El *hit ratio* es del 3 % y la caché solo añade latencia | La clave incluye dimensiones que la hacen casi única por petición | [Caching](Caching.md) |
| Una persona ve datos de otra | La clave no incluye el identificador de usuario, de idioma o de tenant | [Caching](Caching.md) |
| Una clave se queda en Redis para siempre | Un `SET` sin `KEEPTTL` sobre una clave que ya tenía caducidad borra su TTL | [Redis](Redis.md) |
| El servidor de Redis se congela unos cientos de milisegundos | Un `KEYS` en producción, o un `DEL` de una clave enorme | [Redis](Redis.md) |
| `OOM command not allowed when used memory > 'maxmemory'` | La política por defecto es `noeviction`: no expulsa nada, rechaza escrituras | [Redis](Redis.md) |
| La base de datos se cae en punto a la misma hora | Miles de claves guardadas con el mismo TTL caducan a la vez | [Estrategias de invalidación](Estrategias-de-Invalidacion.md) |
| Se cambia un precio y sigue apareciendo el viejo | Se invalidó la clave del producto, pero no las vistas agregadas que lo contienen | [Estrategias de invalidación](Estrategias-de-Invalidacion.md) |
| El carrito aparece vacío al recargar la página | Está en caché local y el balanceador mandó la petición a otra instancia | [Caché distribuida vs. local](Cache-Distribuida-vs-Local.md) |
| El contenedor se reinicia solo por consumo de memoria | Una caché en memoria sin límite de tamaño es una fuga de memoria con otro nombre | [Caché distribuida vs. local](Cache-Distribuida-vs-Local.md) |

## Comprobaciones antes de dar por buena una caché

```bash
# ¿Cuánta memoria ocupa, cuál es el límite y qué hace al alcanzarlo?
redis-cli INFO memory | grep -E 'used_memory_human|maxmemory_human|maxmemory_policy'
redis-cli DBSIZE

# ¿Sirve de algo? ¿Se están expulsando claves por falta de memoria?
redis-cli INFO stats | grep -E 'keyspace_hits|keyspace_misses|evicted_keys'

# ¿Hay claves sin caducidad que deberían tenerla? (-1 significa "para siempre")
redis-cli --scan --pattern 'producto:*' | head -20 | xargs -n1 redis-cli TTL
```

Si `keyspace_misses` es del mismo orden que `keyspace_hits`, la caché no está haciendo su trabajo y el problema está en el diseño de las claves o en la elección de qué cachear: vuelve a [Caching](Caching.md) antes de tocar nada más.

---

> Si el problema no es que la misma consulta se repita, sino el diseño del esquema o la falta de índices, la caché lo tapa pero no lo arregla: mira [PostgreSQL](../postgresql/README.md) o [Acceso a datos en .NET](../acceso-a-datos-dotnet/README.md). Y si lo que quieres cachear son respuestas HTTP completas, la capa que toca es otra —`Cache-Control` y `ETag`—, no Redis.
