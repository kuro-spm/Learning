# Seguridad — Guías

Fundamentos de seguridad aplicada que toda desarrolladora full-stack debería manejar.

---

## Contenido

### [Algoritmos de hash modernos](algoritmos-de-hash/README.md)
Qué algoritmo de hash usar en cada situación y por qué: las propiedades del hash criptográfico, las familias rápidas (SHA-2, SHA-3, BLAKE) frente a las funciones lentas para contraseñas (PBKDF2, bcrypt, scrypt, Argon2id), la sal y sus homónimos, las herramientas de .NET, y el criterio que lo decide todo — la entropía del secreto.

### [Autenticación y autorización](autenticacion-y-autorizacion/README.md)
Quién eres y qué puedes hacer: sesiones y tokens, JWT, OAuth2, OpenID Connect y control de acceso con RBAC, claims y ACL.

### [Gestión de secretos en desarrollo](gestion-de-secretos-en-desarrollo/README.md)
Cómo manejar claves de API, contraseñas y claves de cifrado sin que acaben en git: por qué se separan del código, qué hacer cuando una ya tocó un commit, user-secrets de .NET y el orden de precedencia de la configuración, y el cifrado en reposo de credenciales con AES-GCM y su clave maestra.

### [Protección del login](proteccion-del-login/README.md)
Cómo frenar la fuerza bruta y el DoS contra un endpoint de login en ASP.NET Core: límite de intentos por IP y por cuenta con backoff, el diccionario en memoria que crece sin cota, y cómo obtener la IP real del cliente detrás de un reverse proxy sin abrir un bypass.

### [Secretos en llamadas salientes](secretos-en-llamadas-salientes/README.md)
El complemento en runtime: cómo evitar filtrar una credencial al usarla para llamar a una API externa (logs, redirects, URLs de terceros, TLS).
