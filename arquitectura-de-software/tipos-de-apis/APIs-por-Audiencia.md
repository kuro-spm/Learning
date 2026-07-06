# APIs por Audiencia

**¿Qué es?** Una clasificación de las APIs no por su tecnología, sino por **quién puede usarlas**: públicas (abiertas a cualquiera), privadas (solo para uso interno) y de socios (*partner*, para terceros de confianza). La misma API REST o GraphQL puede caer en cualquiera de estas categorías según a quién se exponga.

---

## ¿Por qué existe?

No todas las APIs tienen el mismo público ni el mismo riesgo. Una API que da el tiempo al mundo entero no se gobierna igual que una que mueve datos internos de nóminas. Distinguir por audiencia ayuda a decidir cuánta seguridad, qué límites y qué documentación necesita cada una.

> Piensa en las puertas de un edificio: la entrada principal (pública) está abierta con vigilancia, las oficinas internas (privada) solo para empleados con tarjeta, y la zona de proveedores (socios) para colaboradores concretos con cita. Misma estructura, distintos niveles de acceso.

---

## ¿Cuándo y para qué se usa?

Es una decisión de diseño y negocio: ¿abro mi API a todo el mundo para que construyan sobre ella, la reservo para mis propias apps, o la comparto solo con empresas aliadas? Esa elección marca la seguridad, el control de acceso y el soporte que tendrás que ofrecer.

---

## Lo mínimo que necesitas saber

**1. API pública (*open / external*)**

Abierta a cualquier desarrollador, a veces tras registrarse. Suele exigir *API key*, tener documentación cuidada y límites de uso (*rate limiting*) para evitar abusos.

```http
GET /api/weather?city=Madrid
X-API-Key: clave-publica-del-desarrollador
```

*Ejemplo:* una API del clima o de mapas que cualquier app puede consumir.

**2. API privada (*internal*)**

Solo para uso dentro de la propia organización: que el frontend hable con el backend, o que dos microservicios internos se comuniquen. No se publica al exterior.

```http
GET /internal/inventory/42        ← solo accesible desde la red interna
```

*Ejemplo:* la API que usa tu propia web para hablar con tu servidor.

**3. API de socios (*partner*)**

Compartida con terceros **concretos y de confianza** (empresas aliadas, integradores), bajo acuerdo. Acceso más restringido que la pública y credenciales específicas por socio.

```http
GET /partner/orders
Authorization: Bearer token-del-socio-acme
```

*Ejemplo:* una tienda que da acceso a su catálogo a un comparador de precios aliado.

**4. La tecnología es independiente de la audiencia**

Una API REST puede ser pública, privada o de socios. La categoría no cambia *cómo* está construida, sino *a quién* se abre y con qué controles.

---

## Lo que NO hace

- **No describe la tecnología** — no dice si es REST, GraphQL o gRPC; solo a quién se dirige.
- **No es una etiqueta fija** — una API interna puede abrirse a socios o al público más adelante (con más seguridad).
- **No sustituye a la autenticación** — clasificar por audiencia no protege nada por sí solo; cada nivel necesita sus controles reales.

---

## Buenas prácticas avanzadas

- **Si tu frontend la llama desde el navegador, ya no es privada** — el error conceptual más común: la API "interna" que consume tu propia web es visible para cualquiera que abra las herramientas de desarrollador, y será llamada con parámetros que tu frontend jamás enviaría. "Privada" de verdad es la que vive en una red interna entre servidores; la que llega al navegador debe protegerse (autenticación, validación, *rate limiting*) exactamente como una pública.
- **Credenciales por socio, nunca una clave compartida** — dar la misma *API key* a tres socios funciona hasta que hay que retirarle el acceso a uno o averiguar quién causó el pico de tráfico de anoche. Cada socio necesita credenciales propias, con permisos limitados a lo suyo (*scopes*) y que se puedan rotar o revocar individualmente sin afectar al resto.
- **Diseña la interna como si fuera a hacerse pública** — es el destino natural: la API que hoy usa solo tu equipo mañana la pide un socio y pasado se abre al mundo. Las internas diseñadas "rápido y sucio" (sin versionado, con datos de más en las respuestas) se convierten en deuda imposible de abrir sin reescribir. La disciplina del contrato no depende de la audiencia; los controles de acceso, sí.

---

*En resumen: clasificar APIs por audiencia es mirar quién entra por la puerta —todos (pública), solo empleados (privada) o aliados concretos (socios)— una decisión de acceso y seguridad, no de tecnología.*
