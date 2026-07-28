# Seguridad de la organización

## ¿Qué es?

El conjunto de políticas y herramientas que una organización de GitHub aplica sobre las cuentas que acceden a su código (segundo factor, inicio de sesión único, tokens) y sobre el código mismo (análisis de dependencias, detección de secretos filtrados, análisis estático), más el registro de auditoría que deja constancia de todo.

## ¿Por qué existe?

Una organización con código privado es un objetivo. No necesariamente por lo que vale el código: por lo que hay **dentro** del código y de sus pipelines. Una clave de acceso a un cloud en un secret, una credencial de base de datos en un fichero de configuración de hace dos años, un token con permiso de escritura en el registro de paquetes. Quien entra en tu organización no roba tu repositorio: usa lo que encuentra dentro para entrar en otro sitio.

Y la vía de entrada casi nunca es un fallo de GitHub. Es la cuenta de una persona: una contraseña reutilizada que apareció en una filtración de otro servicio, un token pegado en un chat, un portátil sin cifrar. De ahí que casi todo lo que sigue trate de **cuentas y credenciales**, no de código.

> Si ya conoces el modelo de amenazas de una aplicación web, aquí el "usuario autenticado" es cada miembro de la organización, y la "escalada de privilegios" es que su cuenta comprometida tiene permiso de escritura en un repositorio cuyo pipeline despliega en producción.

## ¿Cuándo y para qué se usa?

La configuración base se hace al crear la organización y se revisa cuando algo cambia: entra gente, se conecta un proveedor de identidad, un proyecto pasa a manejar datos sensibles. Y hay un momento en que se usa de verdad, sin planificarlo: cuando hay un incidente y toca averiguar qué pasó y qué hay que rotar.

## Segundo factor obligatorio

La medida con mejor relación esfuerzo/beneficio de toda la lista, disponible en todos los planes, y la única que por sí sola neutraliza el vector de ataque más común (una contraseña filtrada o reutilizada).

Se activa en *Settings → Authentication security → Require two-factor authentication*, y hay que conocer su efecto secundario antes de pulsar: **todo miembro o colaborador externo sin 2FA configurado es expulsado de la organización automáticamente**. No bloqueado: expulsado. Recuperarlo es reinvitarlo.

El procedimiento correcto tiene tres pasos:

```bash
# 1. Ver quién no lo tiene todavía
gh api "/orgs/acme-store/members?filter=2fa_disabled" --jq ".[].login"

# 2. Los colaboradores externos van en otra lista y también se ven afectados
gh api "/orgs/acme-store/outside_collaborators?filter=2fa_disabled" --jq ".[].login"
```

3. Avisar, dar plazo, comprobar que ambas listas están vacías y entonces activar.

Sobre el tipo de segundo factor: las **passkeys** y las llaves de seguridad físicas son resistentes al *phishing*, porque están vinculadas al dominio y no se pueden entregar a un sitio falso. Las apps de códigos temporales (TOTP) son sólidas. El **SMS es el más débil** de los tres y conviene desaconsejarlo explícitamente al equipo: el secuestro de número de teléfono es un ataque conocido y barato.

## Inicio de sesión único (SAML) y aprovisionamiento (SCIM)

Disponible en los planes Enterprise. **SAML SSO** delega la autenticación en el proveedor de identidad de la empresa (Entra ID, Okta, Google Workspace): para acceder a la organización hay que pasar antes por el portal corporativo.

Lo que resuelve de verdad no es la comodidad de una contraseña menos: es el **offboarding**. Sin SSO, dar de baja a alguien es un procedimiento manual en GitHub que hay que acordarse de hacer; con SSO, desactivar su cuenta corporativa le corta el acceso a GitHub de inmediato, junto con el del resto de sistemas. El olvido deja de ser posible.

**SCIM** completa el círculo aprovisionando y desaprovisionando cuentas automáticamente desde el proveedor de identidad, y **la sincronización de equipos** hace lo mismo con la pertenencia a los teams a partir de los grupos corporativos.

Un detalle que confunde: con SSO activo, los **tokens de acceso personal y las claves SSH** siguen funcionando, pero hay que **autorizarlos** explícitamente para la organización. Es la causa habitual de "mi script dejó de funcionar cuando activamos el SSO": el token es válido, simplemente no está autorizado para esa organización todavía.

## Políticas de tokens

Un *personal access token* es una credencial de larga duración que actúa en nombre de una persona. Es cómodo y es el tipo de credencial que más se filtra: acaba en un fichero de configuración, en un script, en un mensaje de chat.

Los tokens **fine-grained** son la evolución que hay que usar: se limitan a repositorios concretos, conceden permisos granulares (por ejemplo, solo lectura de contenido y escritura de issues) y **caducan obligatoriamente**. La organización puede exigir que todo token que la toque sea de este tipo y que su creación requiera aprobación de un Owner.

```bash
# Tokens fine-grained pendientes de aprobación
gh api /orgs/acme-store/personal-access-token-requests \
  --jq '.[] | .owner.login + " -> " + .repository_selection'
```

Y para lo que no es una persona sino una máquina, la respuesta correcta **no es un PAT**:

- Un **GitHub App** con permisos mínimos, que emite tokens de una hora. Es lo indicado para automatizaciones permanentes.
- Una **deploy key** por repositorio, si solo hace falta clonar. Y de solo lectura, que es la opción por defecto y la que se suele desmarcar sin pensar.
- **OIDC** para hablar con un proveedor de cloud, sin credencial almacenada (ver [Secrets, variables y entornos](Secrets-variables-y-entornos.md)).

## Las herramientas de análisis

Tres, y las dos primeras están disponibles sin coste añadido:

### Dependabot

Revisa las dependencias declaradas contra la base de datos de vulnerabilidades conocidas y abre pull requests con la actualización. Se configura por repositorio con un fichero:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/backend"
    schedule: { interval: "weekly" }
    open-pull-requests-limit: 5      # sin límite, un lunes te encuentras 40 PRs

  - package-ecosystem: "github-actions"   # también actualiza los `uses:` de tus workflows
    directory: "/"
    schedule: { interval: "weekly" }
```

El segundo bloque es el que casi nadie pone y el que más valor tiene si has fijado las acciones por SHA: Dependabot actualiza esos SHA por pull request, así que ganas inmutabilidad sin renunciar a las actualizaciones.

### Secret scanning

Detecta credenciales en el código —tokens de proveedores conocidos, claves privadas— y avisa. Con **push protection** activado va un paso más allá: **rechaza el push** que introduce el secreto, antes de que llegue al repositorio.

Esa distinción es la que importa. Detectar un secreto ya publicado te obliga a rotarlo, porque el valor está en el historial y en cada clon que exista; bloquear el push evita el problema entero. Actívalo a nivel de organización para todos los repositorios.

Y cuando salta una alerta sobre algo ya publicado, el orden de las acciones es siempre el mismo: **rotar primero, limpiar después**. Reescribir el historial con `git filter-repo` sin haber rotado la credencial no sirve de nada; el valor filtrado sigue siendo válido y probablemente ya está indexado en algún sitio.

### Code scanning

Análisis estático del código (con CodeQL o herramientas de terceros) buscando patrones de vulnerabilidad: inyección SQL, XSS, deserialización insegura. Se ejecuta como un workflow y publica los resultados como alertas.

Es la más costosa de mantener de las tres, porque genera falsos positivos que hay que triar. Merece la pena en código expuesto a internet; en una herramienta interna, empieza por las otras dos.

## El registro de auditoría

Registra las acciones administrativas de la organización: cambios de permisos, creación y borrado de repositorios, cambios de visibilidad, quién añadió o quitó a quién, modificaciones de rulesets y de secrets.

Es la herramienta que usarás en un incidente, y por eso conviene saber consultarla **antes** de necesitarla:

```bash
# Cambios de permisos y de miembros de los últimos días
gh api "/orgs/acme-store/audit-log?phrase=action:org" --jq \
  '.[] | .created_at + "  " + .action + "  " + (.actor // "?")'
```

```
2026-07-20T09:14:22Z  org.update_member         ana-dev
2026-07-18T16:02:10Z  org.add_outside_collaborator  ana-dev
```

Dos límites que hay que conocer: el acceso por **API** al registro suele requerir plan Enterprise (en la interfaz web está disponible más ampliamente), y la **retención no es eterna** —son meses, no años—. Si necesitas conservar el registro para cumplimiento normativo, hay que exportarlo o enviarlo a un sistema externo (*audit log streaming*) de forma continua.

## Qué hacer cuando algo se compromete

Tener el orden decidido de antemano evita perder la primera hora, que es la que cuenta:

1. **Rotar la credencial.** Antes de investigar, antes de escribir a nadie. Mientras el valor sea válido, sigue siendo utilizable.
2. **Revocar accesos de la cuenta afectada**: quitarla de la organización, revocar sus tokens y sus claves SSH, revocar sus sesiones activas.
3. **Mirar el registro de auditoría** para acotar el alcance: qué se tocó, cuándo, desde dónde.
4. **Revisar lo que se ejecuta solo**: workflows nuevos o modificados, webhooks añadidos, deploy keys nuevas. Es donde se instala la persistencia, porque nadie los mira.
5. **Rotar todo lo que esa cuenta pudo ver**, no solo lo que se sabe que vio. Si tenía Admin en un repositorio, asume que los secrets de ese repositorio están comprometidos.
6. **Escribir qué pasó**, aunque sea media página. Sin eso, el mismo incidente vuelve en seis meses.

## Buenas prácticas avanzadas

- **Comprueba las dos listas de 2FA antes de exigirlo.** Miembros y colaboradores externos van en listas separadas, y la política expulsa de las dos. Activarlo a ciegas es la forma de descubrir a media mañana que la persona que iba a desplegar hoy ya no está en la organización.
- **Recomienda passkeys y desaconseja el SMS explícitamente.** Que el 2FA esté activo no dice qué segundo factor usa cada persona. Un equipo entero con SMS ha marcado la casilla de la política y sigue expuesto al secuestro de número, que es un ataque barato y documentado.
- **No uses PATs para automatizaciones.** Un token de persona en un script es una credencial de larga duración con el nombre de alguien que algún día se irá de la empresa: el día que se dé de baja, el proceso nocturno deja de funcionar y nadie sabe por qué. Un GitHub App o una deploy key de solo lectura no tienen ese problema.
- **Añade `package-ecosystem: "github-actions"` a `dependabot.yml`.** Es la línea olvidada. Fijar acciones por SHA sin Dependabot congela tus workflows en versiones cada vez más viejas, y acabas desactivando el fijado por incomodidad. Con esa línea, tienes inmutabilidad y actualizaciones al mismo tiempo.
- **Activa la protección de push del *secret scanning*, no solo la detección.** La diferencia entre las dos es la diferencia entre "hay que rotar esta clave y limpiar el historial" y "no ha pasado nada". Un secreto ya publicado está comprometido para siempre, aunque se borre a los cinco minutos: el historial se ha clonado y los rastreadores automáticos son rápidos.
- **Consulta el registro de auditoría una vez sin que haya pasado nada.** Descubrir en medio de un incidente que el acceso por API requiere otro plan, que la retención es más corta de lo que pensabas o que la consulta no devuelve lo que esperabas es la peor forma de aprenderlo. Una tarde de exploración preventiva es la mejor inversión de esta ficha.

## Recursos didácticos

- [Buenas prácticas de seguridad para organizaciones](https://docs.github.com/organizations/keeping-your-organization-secure) — la guía oficial, ordenada como una checklist aplicable.
- [Have I Been Pwned](https://haveibeenpwned.com/) — comprobar si un correo del equipo aparece en filtraciones conocidas; es la mejor forma de explicar en treinta segundos por qué el 2FA no es negociable.
- [GitHub Advisory Database](https://github.com/advisories) — la base de vulnerabilidades que usa Dependabot, navegable por ecosistema; útil para entender qué está mirando y por qué abre los pull requests que abre.

---

*En resumen: el atacante no entra por GitHub, entra por la cuenta de alguien; el 2FA obligatorio y los tokens de corta duración son casi toda la defensa.*
