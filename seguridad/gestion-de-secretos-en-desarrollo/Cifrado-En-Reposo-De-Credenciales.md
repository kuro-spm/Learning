# Cifrado en reposo de credenciales

## El escenario

A veces tu aplicación tiene que **guardar credenciales de terceros** para usarlas más tarde: la clave de API de un proveedor externo que introduce un administrador, un token para llamar a otro servicio, etc. Ese secreto acaba en tu **base de datos**. Guardarlo en claro es peligroso: un *dump* de la BD, un backup filtrado, una inyección SQL en otro módulo o un backup mal guardado lo expondrían de golpe.

La solución es **cifrarlo en reposo**: se guarda cifrado en la BD y solo se descifra en el momento de usarlo.

## Cifrar no es lo mismo que hashear

Es la distinción clave:

| | Hashing | Cifrado |
|---|---|---|
| Dirección | Solo ida (irreversible) | Ida y vuelta (reversible) |
| Para qué | Contraseñas: solo compruebas si coinciden | Credenciales que hay que **recuperar** para usarlas |
| Ejemplo | `PasswordHasher`, PBKDF2, Argon2 | AES-256-GCM |

Una **contraseña de usuario** se *hashea*: nunca necesitas recuperarla, solo verificar. Una **clave de API de un proveedor** se *cifra*: sí necesitas recuperarla en claro para ponerla en la cabecera de la petición al proveedor. Por eso aquí toca cifrado reversible, no hash.

## Todo cifrado reversible necesita una clave maestra

Y ahí está el matiz importante: para cifrar y descifrar hace falta una **clave maestra**. Esa clave es, en sí misma, **un secreto** — y de los importantes, porque con ella se descifran *todas* las credenciales guardadas.

La analogía: cifrar las credenciales es meterlas en una caja fuerte; la clave maestra es la llave de esa caja fuerte. Si "hardcodeas" la llave en el código, es como pegarla con celo encima de la caja: el candado ya no protege de nada (quien lee el código o roba el repo tiene la llave). Por eso la clave maestra va **fuera del código** exactamente igual que cualquier otro secreto (ver [Por qué los secretos no van a git](Por-Que-Los-Secretos-No-Van-A-Git.md)): user-secrets en dev, variable de entorno o *vault* en producción.

## AES-256-GCM, el estándar razonable

Para cifrado simétrico autenticado, la opción por defecto hoy es **AES-256-GCM**. "Autenticado" (el *GCM*) significa que, además de cifrar, detecta si el dato cifrado ha sido manipulado. Puntos a respetar si lo usas a mano (`System.Security.Cryptography.AesGcm`):

- **Nonce único por cada cifrado.** El *nonce* (12 bytes) es un número que se usa una sola vez. **Nunca** reutilices el mismo nonce con la misma clave: rompe la seguridad de GCM. Genera uno nuevo, aleatorio, en cada operación con un generador criptográfico (`RandomNumberGenerator`).
- **Guarda nonce + tag junto al texto cifrado.** El *tag* (16 bytes) es el sello de autenticación. Un formato típico es concatenar `nonce ‖ tag ‖ textoCifrado` y guardarlo en base64.
- **La clave es de 32 bytes** (256 bits). Valídalo al arrancar: si la clave maestra falta o no mide 32 bytes, mejor **fallar de inmediato** (fail-fast) que arrancar y petar al primer cifrado.

> Alternativa a hacerlo a mano: la **API de protección de datos de ASP.NET Core** (`IDataProtector`), que gestiona claves y rotación por ti. Hacerlo a mano con AES-GCM da más control explícito; `IDataProtector` es más difícil de usar mal. Ambas son válidas.

## Cómo generar una clave maestra de 32 bytes

Una clave AES de 256 bits son 32 bytes aleatorios, que se suelen manejar en **base64** para poder ponerlos en una variable de entorno o en user-secrets:

```bash
openssl rand -base64 32
# -> algo tipo:  i2qYrfRK6OekOcjJ8OBsRPKKMBG2CE7RkIymnl/UCwo=
```

Y se guarda como secreto (nunca en git):

```bash
dotnet user-secrets set "Cifrado:MasterKey" "i2qYrfRK6OekOcjJ8OBsRPKKMBG2CE7RkIymnl/UCwo=" --project src/MiApi
```

## De desarrollo a producción

- **Dev:** clave maestra generada al azar en user-secrets. Es de usar y tirar; si la pierdes, regeneras y vuelves a introducir las credenciales de prueba.
- **Producción:** la clave maestra la provee la plataforma (variable de entorno) o, mejor, un **gestor de secretos** (Azure Key Vault, AWS Secrets Manager, Vault). Hay que pensar además en:
  - **Rotación:** cambiar la clave maestra implica descifrar con la vieja y recifrar con la nueva. Planifícalo antes de tener datos reales.
  - **Backup de la clave:** si pierdes la clave maestra de producción, **las credenciales cifradas se vuelven ilegibles** para siempre.
  - **Multi-instancia:** si hay varias réplicas, todas necesitan la misma clave desde un origen compartido.

## Alcance honesto de la protección

Cifrar en reposo protege frente a **fuga de la BD, backups, logs y accesos casuales** — que es la mayoría de incidentes reales. **No** protege frente a un compromiso total del proceso en ejecución: quien controle el proceso puede leer en memoria tanto la clave maestra (del entorno) como el secreto descifrado. Aun así merece mucho la pena: sube el listón enormemente por un coste bajo. Un *vault* gestionado lo sube todavía más.

## Lo mínimo que hay que recordar

- Credencial recuperable → **cifrado** (reversible). Contraseña → **hash** (irreversible).
- El cifrado necesita una **clave maestra**, que es un secreto: fuera del código, como todos.
- AES-256-GCM: **nonce único por cifrado**, guarda nonce+tag, clave de 32 bytes, fail-fast si falta.
- Genera la clave con `openssl rand -base64 32` y guárdala en user-secrets (dev) / vault (prod).
- Planifica **rotación y backup** de la clave maestra antes de producción.
