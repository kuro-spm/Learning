# Roles y permisos

## ¿Qué es?

El sistema de autorización de una organización de GitHub, que se organiza en dos niveles: los **roles de organización** (qué puedes administrar en la organización) y los **permisos de repositorio** (qué puedes hacer en cada repositorio concreto).

## ¿Por qué existe?

Porque "tener acceso" no es una cosa, son muchas. La persona que revisa pull requests no necesita poder borrar el repositorio. Quien paga las facturas no necesita leer el código. Quien atiende el backlog de issues no necesita permiso de escritura. Un modelo con solo dos estados —dentro o fuera— obliga a dar a todo el mundo el permiso del caso más exigente, que es exactamente lo contrario del mínimo privilegio.

GitHub separa por eso los dos ejes. Y el motivo de que sean dos y no uno es útil de entender: los roles de organización responden a *"¿qué puedes cambiar en la configuración común?"*, y los permisos de repositorio a *"¿qué puedes hacer con este código?"*. Una misma persona puede ser miembro raso de la organización y administradora de un repositorio concreto.

> Si ya conoces la autorización basada en roles de una aplicación web, es el mismo patrón con dos ámbitos: un rol global y un rol por recurso. La suma de los dos decide.

## ¿Cuándo y para qué se usa?

Cada vez que entra alguien nuevo, cada vez que alguien cambia de responsabilidad y —el que más se olvida— cada vez que alguien se va. También en la auditoría periódica: revisar quién tiene qué es una de esas tareas que solo duele si no se hace nunca.

## Los roles de organización

### Owner

Control total: facturación, ajustes de seguridad, creación y borrado de repositorios, gestión de miembros, y acceso administrativo a **todos** los repositorios de la organización independientemente de lo que digan los equipos.

Debe haber **al menos dos**, por continuidad, y **no muchos más**, porque un Owner puede saltarse casi cualquier protección. En una organización de diez personas, dos o tres Owners es la proporción sana.

### Member

El rol por defecto de cualquier persona de la organización. Su acceso al código es exactamente el que le dan (a) el permiso base de la organización y (b) los equipos a los que pertenece. Un Member no puede cambiar ajustes de la organización.

### Billing manager

Solo facturación: ver y cambiar el método de pago, los asientos y las facturas. **Sin acceso al código ni a los repositorios.** Es el rol para la persona de administración o para la gestoría, y es una de las mejores demostraciones de para qué sirve separar roles: quien paga no necesita leer.

### Security manager

Acceso de lectura a todos los repositorios más la capacidad de gestionar las alertas y la configuración de seguridad de la organización. Sin poder administrativo sobre el resto de ajustes ni sobre el código.

Es el rol correcto para quien lleva la seguridad sin ser Owner: puede revisar alertas de dependencias, de secretos filtrados y de análisis de código en toda la organización sin que haya que convertirlo en administrador de todo.

### Moderator

Puede bloquear e ignorar usuarios y gestionar el contenido de las discusiones e issues públicas. Solo relevante en organizaciones con proyectos abiertos y comunidad externa.

### Roles personalizados

Los planes Enterprise permiten definir roles de organización y de repositorio a medida, partiendo de uno base y añadiendo o quitando permisos concretos. Es potente y, precisamente por eso, fácil de convertir en un laberinto que nadie entiende seis meses después. Si estás empezando, quédate con los roles estándar.

## Los cinco permisos de repositorio

Aquí es donde ocurre el trabajo real. De menos a más:

| Permiso | Qué añade sobre el anterior |
|---|---|
| **Read** | Clonar, leer, abrir issues y pull requests, comentar. |
| **Triage** | Gestionar issues y PRs ajenos: etiquetar, asignar, cerrar, marcar duplicados. **Sigue sin poder escribir código.** |
| **Write** | `git push` a ramas no protegidas, fusionar pull requests, gestionar releases, ejecutar workflows. |
| **Maintain** | Configuración del repositorio sin lo destructivo ni lo sensible: temas, ramas, webhooks. **No** ve ni gestiona secrets, **no** puede borrar el repositorio. |
| **Admin** | Todo: secrets, reglas de rama y rulesets, colaboradores, visibilidad, borrado. |

Los dos que la gente no usa y debería:

**Triage** es para quien atiende el proyecto sin programar en él: soporte, gestión de producto, QA que reporta y clasifica. Da todo lo necesario para ordenar el backlog y nada para tocar código.

**Maintain** es el rol de "responsable del repositorio" y resuelve el salto brusco de Write a Admin. Alguien con Maintain puede llevar el día a día del repositorio —ajustes, ramas, integraciones— sin ver los secretos de despliegue ni poder borrarlo. En la práctica es el permiso correcto para el *tech lead* de un proyecto, mientras Admin se reserva a los Owners.

```bash
# Conceder Maintain a un equipo sobre un repositorio
gh api -X PUT /orgs/acme-store/teams/backend/repos/acme-store/api-pedidos \
  -f permission=maintain
```

Un detalle de la API que conviene tener a mano, porque los nombres no coinciden con la interfaz: en las llamadas se usan `pull`, `triage`, `push`, `maintain` y `admin` para Read, Triage, Write, Maintain y Admin respectivamente. `push` es Write y `pull` es Read.

## Cómo se combinan: el permiso efectivo

El permiso real de una persona sobre un repositorio es **el más alto** de todas las vías por las que le llega:

1. El **permiso base** de la organización (aplicado a todos los miembros sobre todos los repositorios).
2. Los permisos de los **equipos** a los que pertenece (y de los equipos padre, que se heredan hacia abajo).
3. Las **concesiones directas** como colaborador del repositorio.
4. El **rol de organización**: un Owner tiene Admin en todo, y nada de lo anterior lo limita.

La consecuencia práctica es la que suele pillar a la gente por sorpresa: **quitar a alguien de un equipo no le quita necesariamente el acceso**. Si además tenía una concesión directa, o si el permiso base de la organización ya le da lectura, el acceso sigue ahí por otra vía.

Para comprobar el permiso efectivo de verdad, en lugar de deducirlo:

```bash
gh api /repos/acme-store/api-pedidos/collaborators/rmartin/permission
```

```json
{
  "permission": "write",
  "role_name": "write",
  "user": { "login": "rmartin" }
}
```

Y para saber **por qué** tiene ese permiso, que es la pregunta útil en una auditoría, hay que mirar las tres vías:

```bash
# ¿Le llega por equipo?
gh api /repos/acme-store/api-pedidos/teams --jq ".[] | .slug + \" -> \" + .permission"

# ¿O es una concesión directa? (role_name a nivel de colaborador)
gh api /repos/acme-store/api-pedidos/collaborators --jq ".[] | .login + \" -> \" + .role_name"
```

## Colaboradores externos

Un *outside collaborator* tiene acceso a repositorios concretos sin ser miembro de la organización: no aparece en la lista de miembros, no le afecta el permiso base y no ve el resto de repositorios.

Es el mecanismo correcto para un freelance, un consultor externo o el equipo de otra empresa en un proyecto conjunto. Dos cosas que hay que tener claras:

- **Consume asiento facturable** en los planes de pago si accede a repositorios privados.
- **El 2FA obligatorio de la organización también le aplica**, y si no lo tiene, al activar la política se le expulsa.

```bash
# Auditoría trimestral: quién sigue teniendo acceso desde fuera
gh api /orgs/acme-store/outside_collaborators --jq ".[].login"
```

Si esa lista tiene nombres que no reconoces al instante, ahí tienes el trabajo de la tarde.

## Dar de baja a alguien correctamente

Quitar a una persona de la organización es más que borrar la fila. La secuencia completa:

```bash
# 1. Quitarle de la organización (elimina de paso su pertenencia a equipos)
gh api -X DELETE /orgs/acme-store/memberships/exempleado

# 2. Comprobar que no queda como colaborador directo en algún repositorio
gh api /repos/acme-store/api-pedidos/collaborators --jq ".[].login"
```

Y después, lo que no se ve en ninguna lista de miembros y es lo que de verdad importa:

- **Revocar los personal access tokens y las deploy keys** que hubiera creado. Un token de esa persona sigue funcionando después de darla de baja si el token pertenece a un ámbito que aún tiene acceso; las deploy keys instaladas en un repositorio no se van con su autora.
- **Rotar los secretos que conocía.** Si tenía Admin, ha podido ver o poner claves de despliegue. Darla de baja no cambia la clave privada del servidor.
- **Reasignar sus issues y pull requests abiertos**, o quedarán huérfanos.
- **Revisar `CODEOWNERS`**: si su usuario aparece ahí, las revisiones requeridas apuntarán a alguien que ya no existe y los pull requests se quedarán bloqueados esperando una aprobación imposible.

## Buenas prácticas avanzadas

- **Concede permisos a equipos, nunca a personas.** Es la regla que sostiene todo lo demás. Con concesiones directas, dar de baja a alguien exige recorrer cada repositorio; con equipos, quitarlo del equipo lo desconecta de golpe. Y la lista de "quién ve qué" se puede leer de un vistazo en lugar de reconstruirse repositorio a repositorio.
- **Usa Maintain para los responsables de repositorio y reserva Admin a los Owners.** El salto de Write a Admin es el error de diseño más común: para que alguien pueda ajustar su repositorio se le acaba dando acceso a los secretos de producción y capacidad de borrarlo. Maintain existe exactamente para ese hueco.
- **Recuerda que quitar de un equipo no revoca el acceso.** El permiso efectivo es el máximo de todas las vías. Si el permiso base de la organización es `Read`, sacar a alguien del equipo `backend` lo deja aún con lectura de todo. Con base `No permission`, el acceso desaparece de verdad. Es otro argumento para poner la base en `none`.
- **Trata las deploy keys y los PATs como parte del offboarding.** Es el agujero silencioso: la persona ya no está en la lista de miembros, no aparece en ninguna auditoría de accesos, y su token sigue clonando el repositorio cada noche desde un script que nadie recuerda. Al dar una baja, busca también las claves.
- **Audita el permiso efectivo, no la configuración.** La pregunta correcta no es "¿a quién he dado acceso?", sino "¿quién tiene acceso ahora mismo?". Son distintas en cuanto la organización tiene equipos anidados y concesiones históricas. `gh api /repos/ORG/REPO/collaborators` con `role_name` responde a la segunda, que es la que importa.

## Recursos didácticos

- [Roles en una organización](https://docs.github.com/organizations/managing-peoples-access-to-your-organization-with-roles/roles-in-an-organization) — la matriz oficial y exhaustiva de qué puede hacer cada rol.
- [Permisos de repositorio para una organización](https://docs.github.com/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/repository-roles-for-an-organization) — la tabla completa de los cinco permisos, permiso por permiso.
- `gh api /repos/ORG/REPO/collaborators --jq '.[] | .login + " " + .role_name'` — la forma más rápida de auditar un repositorio; guárdalo como alias, lo usarás más de lo que crees.

---

*En resumen: dos ejes (rol de organización y permiso de repositorio), permisos siempre a equipos, y recuerda que el efectivo es el máximo de todas las vías, no la última que configuraste.*
