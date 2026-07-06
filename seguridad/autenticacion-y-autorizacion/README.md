# Autenticación y autorización — Guía introductoria

> 🧭 Colección añadida como recomendación proactiva, no por una necesidad concreta de ningún proyecto.

Tutorial introductorio sobre los conceptos que sostienen el control de acceso en cualquier aplicación web: quién eres, cómo se recuerda eso entre requests, y qué puedes hacer una vez identificada. Cada ficha explica qué es el concepto, por qué existe, cuándo se usa y lo mínimo que necesitas saber, con ejemplos genéricos que se entienden sin conocer ningún proyecto concreto.

Sigue el orden: primero la distinción base entre autenticación y autorización, después cómo se mantiene el login entre requests, luego el formato de token más extendido y los estándares que delegan acceso e identidad, y por último cómo se afina la autorización una vez sabes quién es la usuaria.

---

## Orden de lectura recomendado

### 1. Los conceptos base

Antes de entrar en protocolos concretos, la distinción de la que todo lo demás depende.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Autenticación vs Autorización](Autenticacion-vs-Autorizacion.md) | El "quién eres" frente al "qué puedes hacer". La base de todo lo que sigue. |
| 2 | [Sesiones vs Tokens](Sesiones-vs-Tokens.md) | Cómo se recuerda que ya te autenticaste, request tras request: en el servidor o en el propio token. |

### 2. Tokens y estándares de identidad

El formato de token más usado y los protocolos que lo emiten para resolver acceso delegado e identidad federada.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [JWT](JWT.md) | El formato de token concreto que aparece en el resto de fichas de esta sección. |
| 4 | [OAuth2](OAuth2.md) | Cómo delegar acceso a tus datos en otro servicio sin compartir tu contraseña. |
| 5 | [OpenID Connect](OpenID-Connect.md) | La capa de identidad que OAuth2 no resuelve por sí solo: el "iniciar sesión con...". |

### 3. Autorización avanzada

Una vez sabes quién es la usuaria, cómo decidir con precisión qué puede hacer.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 6 | [RBAC y Claims](RBAC-y-Claims.md) | De los roles simples a las políticas basadas en claims para reglas más finas. |

---

> Si te interesa cómo se protegen las contraseñas y los tokens antes de guardarlos, la guía de [Algoritmos de hash modernos](../algoritmos-de-hash/README.md) es el complemento natural de esta colección.
