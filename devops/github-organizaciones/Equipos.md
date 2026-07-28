# Equipos (teams)

## ¿Qué es?

Un equipo es un grupo de miembros de la organización al que se conceden permisos en bloque sobre repositorios. Los equipos pueden anidarse, y también sirven para mencionar a un grupo, asignar revisiones automáticamente y expresar la propiedad del código con `CODEOWNERS`.

## ¿Por qué existe?

Sin equipos, el acceso se concede persona a persona y repositorio a repositorio. Con cinco personas y tres repositorios son quince decisiones; con doce personas y quince repositorios son ciento ochenta, y ya nadie sabe quién tiene qué. Peor: cuando alguien se va, hay que recorrer los quince repositorios para desconectarlo, y si te olvidas de uno no hay ninguna señal que te lo diga.

Los equipos introducen un nivel intermedio entre personas y repositorios. Los permisos se conceden al equipo, y la pertenencia al equipo se gestiona en un solo sitio. Añadir a alguien al equipo `backend` le da de golpe el acceso correcto a todos los repositorios de backend; sacarlo se lo quita de todos a la vez.

> Si ya conoces los grupos de un sistema operativo o de un directorio LDAP, es exactamente la misma idea: permisos al grupo, personas al grupo, y nunca permisos a personas.

## ¿Cuándo y para qué se usa?

En cuanto hay más de un repositorio o más de tres personas. Y para cuatro cosas distintas que conviene no confundir, porque un mismo equipo suele servir para todas:

- **Conceder acceso** a repositorios.
- **Mencionar** a un grupo (`@acme-store/backend` en un issue avisa a todo el equipo).
- **Repartir revisiones** automáticamente entre sus miembros.
- **Declarar propiedad** del código en `CODEOWNERS`.

## Crear equipos y organizarlos

```bash
gh api -X POST /orgs/acme-store/teams \
  -f name=backend \
  -f description="Servicios y API" \
  -f privacy=closed
```

El campo `privacy` tiene dos valores y la nomenclatura despista: `closed` significa **visible** para toda la organización (cualquier miembro ve que el equipo existe y quién está en él), y `secret` significa oculto (solo lo ven sus miembros y los Owners). Por defecto usa `closed`: la visibilidad es lo que permite mencionar al equipo y lo que hace que la estructura sea comprensible. `secret` solo tiene sentido para casos concretos, como un equipo de seguridad que gestiona incidentes.

La decisión más importante no es técnica, es de diseño. Hay dos formas de organizar equipos:

- **Por función**: `backend`, `frontend`, `qa`, `infra`. Encaja cuando las mismas personas trabajan en varios proyectos con el mismo rol. Es lo habitual en una consultora.
- **Por producto o cliente**: `proyecto-tienda`, `cliente-acme`. Encaja cuando cada proyecto tiene su gente y el aislamiento entre clientes importa.

Muchas organizaciones acaban combinando las dos con anidamiento, y la trampa es multiplicar equipos hasta que nadie entiende la estructura. Regla práctica: si no puedes explicar en una frase para qué existe un equipo, no debería existir.

## Anidamiento y herencia

Un equipo puede tener equipos hijos, y **los hijos heredan los permisos del padre**. La herencia va en un solo sentido, hacia abajo:

```
ingenieria                    ← Read en todos los repositorios
├── backend                   ← + Write en api-pedidos
│   └── backend-guardia       ← + Maintain en api-pedidos
└── frontend                  ← + Write en web-tienda
```

Quien está en `backend` obtiene el `Read` general que tiene `ingenieria` más el `Write` de su propio equipo. Quien está en `backend-guardia` acumula los tres niveles. Al contrario no funciona: estar en `ingenieria` no da los permisos de `backend`.

```bash
# Crear un equipo hijo (necesita el id numérico del padre, no su nombre)
PADRE=$(gh api /orgs/acme-store/teams/ingenieria --jq .id)
gh api -X POST /orgs/acme-store/teams \
  -f name=backend -F parent_team_id=$PADRE -f privacy=closed
```

El anidamiento resuelve elegantemente el caso "todo el mundo lee todo, cada equipo escribe en lo suyo": das `Read` en el padre y `Write` en cada hijo, en lugar de repetir la concesión de lectura en todos los equipos.

Dos avisos:

- **Con dos niveles casi siempre basta.** Tres se puede justificar. A partir de ahí, calcular el permiso efectivo de alguien deja de ser algo que puedas hacer de cabeza, y eso es justo lo que la estructura debía evitar.
- **La herencia solo suma.** No existe una forma de que un hijo tenga *menos* permiso que su padre. Si necesitas eso, el diseño está mal: el permiso que quieres quitar no debería estar en el padre.

## Conceder acceso a repositorios

```bash
# El equipo backend escribe en api-pedidos
gh api -X PUT /orgs/acme-store/teams/backend/repos/acme-store/api-pedidos \
  -f permission=push

# El equipo qa solo lee
gh api -X PUT /orgs/acme-store/teams/qa/repos/acme-store/api-pedidos \
  -f permission=pull
```

Recuerda la equivalencia de nombres en la API: `pull` es Read, `push` es Write, y `triage`, `maintain` y `admin` se llaman igual que en la interfaz.

Para auditar de un vistazo qué tiene un equipo:

```bash
gh api /orgs/acme-store/teams/backend/repos \
  --jq '.[] | .name + " -> " + .role_name'
```

```
api-pedidos -> push
servicio-pagos -> push
web-tienda -> pull
```

Esa salida es la respuesta a "¿qué toca este equipo?" en una línea, y es el tipo de comprobación que en un modelo de concesiones directas simplemente no se puede hacer.

## Revisiones automáticas

Si pones un equipo como revisor de un pull request, por defecto se notifica a **todos** sus miembros y cualquiera puede aprobar. En un equipo de ocho personas eso genera ruido y difusión de responsabilidad: si todos son responsables, nadie lo es.

La asignación automática reparte: GitHub elige a un número concreto de personas del equipo y solo les pide revisión a ellas. Se configura en la página del equipo (*Settings → Code review*) eligiendo el algoritmo:

- **Round robin**: por turnos, según quién recibió la última.
- **Load balance**: intenta igualar el número de revisiones recientes de cada persona.

Se puede además excluir a quien esté ausente y decidir si, al asignar a personas concretas, se elimina la petición al equipo entero. Es un ajuste de dos minutos que reparte la carga de revisión de forma medible.

## CODEOWNERS

`CODEOWNERS` es un fichero que declara **quién es propietario de qué partes del código**. GitHub lo usa para pedir revisión automáticamente cuando un pull request toca esas rutas y, si la protección de rama lo exige, para **requerir** la aprobación de esas personas.

Va en la raíz, en `docs/` o en `.github/`, y su sintaxis son patrones tipo `.gitignore` seguidos de propietarios:

```
# .github/CODEOWNERS

# Propietario por defecto de todo lo que no encaje más abajo
*                       @acme-store/ingenieria

# Por área del monorepo
/backend/               @acme-store/backend
/frontend/              @acme-store/frontend

# Por tipo de fichero, en cualquier carpeta
*.sql                   @acme-store/dba

# Lo sensible pide dos equipos: ambos tienen que aprobar
/.github/workflows/     @acme-store/infra @acme-store/seguridad
/backend/**/Auth/       @acme-store/backend @acme-store/seguridad
```

La regla que hay que memorizar: **gana la última coincidencia**, no la más específica. Un fichero `backend/schema.sql` en el ejemplo de arriba pertenece a `@acme-store/dba`, porque la línea de `*.sql` está después de la de `/backend/`. Si inviertes el orden de esas dos líneas, el propietario cambia. Es la fuente de la mitad de las sorpresas con `CODEOWNERS`.

Tres cosas más que fallan a menudo:

- Los equipos mencionados deben tener **al menos permiso de lectura** sobre el repositorio, o GitHub ignora la línea en silencio.
- Un error de sintaxis o un equipo inexistente invalida la línea, también en silencio. GitHub muestra los errores en la vista del fichero `CODEOWNERS` en la interfaz web: merece la pena mirarla después de editarlo.
- `CODEOWNERS` **pide** revisión por sí solo, pero solo la **exige** si la protección de la rama activa "Require review from Code Owners". Sin eso, es una sugerencia.

## Sincronizar con el proveedor de identidad

En los planes Enterprise con SAML, la pertenencia a equipos puede sincronizarse con los grupos del proveedor de identidad (Entra ID, Okta): las personas entran y salen de los equipos de GitHub según su grupo corporativo, sin gestión manual.

Es lo correcto a cierta escala, y elimina de golpe el problema del offboarding: al desactivar la cuenta corporativa, el acceso a GitHub desaparece. Si no tienes proveedor de identidad, la alternativa realista es una revisión periódica de la pertenencia a equipos.

## Buenas prácticas avanzadas

- **Concede permisos a equipos incluso cuando el equipo tiene una sola persona.** Parece burocracia y es lo contrario: cuando esa persona se va o entra la segunda, el cambio es de un clic en la pertenencia en vez de una expedición por todos los repositorios. El coste de crear el equipo es cero; el de no tenerlo se paga siempre en el peor momento.
- **Ordena `CODEOWNERS` de lo general a lo específico y ten presente que gana la última línea.** Es contraintuitivo (uno espera que gane la regla más específica, como en un router) y produce el fallo silencioso clásico: el equipo de base de datos nunca es avisado de los cambios de esquema porque una regla posterior de carpeta se los quedó.
- **Exige dos equipos como propietarios en lo sensible.** Poner `@infra @seguridad` en `/.github/workflows/` significa que modificar el pipeline de despliegue necesita las dos aprobaciones. Es el control más eficaz contra el vector de ataque más rentable de un repositorio: cambiar el workflow para que exfiltre los secretos.
- **Activa la asignación automática de revisiones en equipos de más de tres personas.** Sin ella, un pull request notifica a ocho personas y no lo revisa ninguna, porque cada una asume que lo hará otra. Con round robin hay siempre un nombre concreto con la pelota.
- **Revisa `CODEOWNERS` cada vez que alguien deja la organización.** Si su usuario aparece ahí y la rama exige revisión de propietarios, los pull requests que toquen sus rutas quedan bloqueados esperando la aprobación de una cuenta que ya no tiene acceso. El síntoma —"no puedo fusionar y no entiendo por qué"— aparece días después y cuesta relacionarlo con la baja.

## Recursos didácticos

- [Sobre los equipos](https://docs.github.com/organizations/organizing-members-into-teams/about-teams) — jerarquía, visibilidad y menciones, con los detalles de la herencia.
- [Sintaxis de CODEOWNERS](https://docs.github.com/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) — la referencia de patrones; ten a mano la sección de reglas de precedencia.
- La vista del fichero `CODEOWNERS` en la interfaz web de GitHub — valida la sintaxis y señala los errores; es el único sitio donde se ven, y ahorra depurar a ciegas por qué una regla no se aplica.

---

*En resumen: los permisos se dan a equipos y las personas se mueven entre equipos; y en `CODEOWNERS`, la última línea que coincide es la que manda.*
