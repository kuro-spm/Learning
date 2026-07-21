# Gestión de secretos en desarrollo — Guía práctica

Tutorial introductorio sobre **cómo manejar secretos (claves de API, contraseñas de BD, claves de cifrado) mientras desarrollas**, sin que acaben nunca en el control de versiones. Pensado para desarrolladoras full-stack en el ecosistema .NET, con ejemplos genéricos que se entienden sin conocer ningún proyecto concreto.

La idea central en una frase: **un secreto nunca debe vivir en el mismo sitio que el código**. Quien lee el repositorio no debería obtener con ello las llaves de nada.

---

## Orden de lectura recomendado

### 1. El principio

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Por qué los secretos no van a git](Por-Que-Los-Secretos-No-Van-A-Git.md) | Qué es un secreto, por qué el historial de git es una trampa, y el concepto de "dominio de confianza" separado. |

### 2. La herramienta en .NET

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [User-secrets en .NET](User-Secrets-En-Dotnet.md) | El mecanismo oficial para secretos de **desarrollo**: qué son, dónde viven, cómo se cargan solos, y todos los comandos (`init`/`set`/`list`/`remove`). |

### 3. Cuando el secreto es una clave de cifrado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [Cifrado en reposo de credenciales](Cifrado-En-Reposo-De-Credenciales.md) | Si guardas credenciales de terceros en tu BD, hay que cifrarlas — y esa clave maestra es en sí un secreto. Diferencia con hashing, AES-GCM, y cómo gestionar la clave de dev a producción. |

---

> ¿Te quedas con ganas de más? Buenos siguientes pasos: **gestores de secretos gestionados** (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault) para producción, la **protección de datos de ASP.NET Core** (`IDataProtector`) como alternativa a cifrar a mano, y el **escaneo de secretos** en CI (gitleaks, trufflehog) para cazar los que se cuelan.
