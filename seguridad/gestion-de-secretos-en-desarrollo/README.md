# Gestión de secretos en desarrollo — Guía de tecnologías

Cómo manejar secretos —claves de API, contraseñas de base de datos, claves de cifrado— mientras desarrollas, sin que acaben nunca en el control de versiones y sin depender de que nadie se acuerde de borrarlos antes del commit.

La idea central en una frase: **un secreto nunca debe vivir en el mismo sitio que el código**. Quien lee el repositorio no debería obtener con ello las llaves de nada.

Los ejemplos comparten escenario: la API de una tienda online (`src/Tienda.Api`) que necesita la clave de una pasarela de pagos y la cadena de conexión de su base de datos.

---

## Orden de lectura recomendado

### 1. El principio

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Por qué los secretos no van a git](Por-Que-Los-Secretos-No-Van-A-Git.md) | Qué distingue un secreto de una configuración, por qué el historial de git es una trampa de la que no se sale borrando el fichero, y el procedimiento correcto cuando una clave ya tocó un commit: rotar primero, limpiar después. |

### 2. La herramienta en desarrollo

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [User-secrets en .NET](User-Secrets-En-Dotnet.md) | El mecanismo oficial para secretos de desarrollo: el orden de precedencia de la configuración de .NET, dónde vive `secrets.json`, los comandos uno a uno, y cómo la misma clave llega por variable de entorno en producción. |

### 3. Cuando el secreto es una clave de cifrado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [Cifrado en reposo de credenciales](Cifrado-En-Reposo-De-Credenciales.md) | Si guardas credenciales de terceros en tu propia base de datos hay que cifrarlas, y esa clave maestra es en sí misma un secreto: hash frente a cifrado, AES-256-GCM paso a paso, rotación de claves y qué protege realmente esta capa. |

---

## Dónde vive el secreto según el entorno

| Entorno | Dónde vive | Quién lo provee | Si se pierde |
|---|---|---|---|
| Desarrollo local | `secrets.json`, en el perfil de usuario y fuera del repositorio | Cada persona pone los suyos | Se vuelve a poner con `dotnet user-secrets set` |
| CI | Los secretos de la plataforma, inyectados como variables de entorno | La configuración del pipeline | Se regenera en el proveedor |
| Producción | Variable de entorno o gestor de secretos (Key Vault, Secrets Manager, Vault) | La plataforma de despliegue | Depende: una clave de API se rota; una clave maestra de cifrado perdida deja los datos ilegibles para siempre |

## Comprobaciones antes de dar por bueno un repositorio

Tres comandos que caben en una terminal y responden a las tres preguntas que importan:

```bash
# ¿Hay algún secreto en el historial completo, no solo en el estado actual?
gitleaks detect --source . --verbose

# ¿Qué ficheros sensibles está siguiendo git ahora mismo?
git ls-files | grep -Ei '\.env|secrets\.json|\.pfx|\.pem'

# ¿Qué claves espera la aplicación? Deben estar declaradas y vacías.
cat src/Tienda.Api/appsettings.json
```

Si el primero encuentra algo, el orden es siempre el mismo: **rotar la credencial en el proveedor antes de tocar el historial**. Limpiar el historial sin rotar no protege de nada, porque cualquier copia previa del repositorio sigue teniendo la clave. Está desarrollado en [Por qué los secretos no van a git](Por-Que-Los-Secretos-No-Van-A-Git.md).

---

> El complemento en tiempo de ejecución de esta colección es [Secretos en llamadas salientes](../secretos-en-llamadas-salientes/README.md): cómo evitar filtrar una credencial en el momento de usarla, en logs, redirecciones, URLs de terceros o por una validación de TLS desactivada. Y si el secreto es la contraseña de una persona usuaria, lo que toca no es cifrarla sino hashearla: [Algoritmos de hash](../algoritmos-de-hash/README.md).
