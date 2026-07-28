# Crear una organización

## ¿Qué es?

El proceso de dar de alta una organización en GitHub y dejarla configurada con unos ajustes de partida sensatos, antes de meter dentro ningún repositorio ni invitar a nadie.

## ¿Por qué existe?

Crear la organización en sí son dos minutos y un formulario. El trabajo real es lo que viene después: una organización recién creada tiene **los valores por defecto de GitHub**, que están pensados para no estorbar a quien acaba de llegar, no para gobernar código de un cliente. Por defecto cualquier miembro puede crear repositorios, el token de los workflows tiene más permisos de los que necesita y no se exige segundo factor.

Cambiar esos ajustes cuando la organización está vacía es trivial. Cambiarlos cuando ya hay quince repositorios, cuarenta workflows y doce personas trabajando es una negociación. De ahí que merezca la pena tratar la creación como una lista de comprobación y no como un formulario.

## ¿Cuándo y para qué se usa?

Al arrancar: una empresa que formaliza su código, una consultora que separa el trabajo de clientes de las cuentas personales de su equipo, o un proyecto que pasa de una persona a tres. También al reorganizar: a veces la respuesta correcta a "esta organización se ha convertido en un cajón de sastre" es crear una segunda con un propósito claro.

## Paso 1 — Crear la organización

Desde la interfaz web: menú de tu foto de perfil → *Your organizations* → *New organization*, y eliges el plan (se puede cambiar después). El formulario pide tres cosas:

- **Nombre de la organización**, que será el *namespace* de todo: `github.com/<nombre>/<repo>`, `ghcr.io/<nombre>/<imagen>`. Elige el nombre estable de la empresa, en minúsculas y con guiones. No el nombre del proyecto de moda ni el del cliente actual.
- **Correo de contacto de facturación**, que conviene que sea una lista o un alias de la empresa (`facturacion@acme.com`), nunca el correo personal de quien crea la organización. Es el mismo error del punto único de fallo, en versión administrativa.
- **A quién pertenece**: a tu cuenta personal o a una empresa. Para trabajo profesional, empresa.

Verificas que existe:

```bash
gh api /orgs/acme-store
```

De la respuesta interesan estos campos, que confirman el plan y los ajustes por defecto que vas a cambiar a continuación:

```json
{
  "login": "acme-store",
  "plan": { "name": "team", "seats": 5, "filled_seats": 1 },
  "default_repository_permission": "read",
  "members_can_create_repositories": true,
  "two_factor_requirement_enabled": false
}
```

Los tres últimos son exactamente los que hay que revisar.

## Paso 2 — Fijar el permiso base

*Settings → Member privileges → Base permissions*. Es el permiso que **todo miembro** tiene automáticamente sobre **todos** los repositorios de la organización, sin que nadie se lo conceda. Las opciones son `No permission`, `Read`, `Write` y `Admin`.

`Write` y `Admin` como base están descartados: convierten a cualquier recién llegado en alguien que puede empujar código a cualquier proyecto. La decisión real es entre las otras dos:

- **`Read`** — cualquier miembro puede leer cualquier repositorio. Fomenta la transparencia interna, facilita buscar código y reutilizar soluciones. Es la opción razonable para un equipo pequeño donde todo el mundo trabaja en todo.
- **`No permission`** — nadie ve nada hasta que un equipo o una concesión explícita le da acceso. Es mínimo privilegio de verdad. Obligatorio si distintos proyectos pertenecen a **clientes distintos**: el código de un cliente no debe ser legible por quien trabaja para otro.

```bash
# Mínimo privilegio: nadie ve nada por defecto
gh api -X PATCH /orgs/acme-store -f default_repository_permission=none
```

Para una consultora con varios clientes, `none` no es exceso de celo: es probablemente una obligación contractual.

## Paso 3 — Restringir quién crea repositorios

En la misma pantalla de *Member privileges*. Por defecto, cualquier miembro puede crear repositorios, incluidos públicos. Eso significa que alguien puede publicar código de la empresa en internet con dos clics y sin mala intención.

La configuración sensata: dejar la creación de repositorios **solo a los Owners**, o como mucho permitir repositorios privados a los miembros pero **nunca públicos**.

```bash
gh api -X PATCH /orgs/acme-store \
  -F members_can_create_public_repositories=false \
  -F members_can_create_private_repositories=true
```

Conviene revisar también, en esa pantalla, quién puede **borrar o transferir** repositorios (déjalo en Owners) y si los miembros pueden invitar a colaboradores externos por su cuenta (mejor que no: es la vía más silenciosa por la que alguien de fuera acaba con acceso a código privado).

## Paso 4 — Exigir el segundo factor

*Settings → Authentication security → Require two-factor authentication*.

Esto está disponible en todos los planes y no hay ninguna buena razón para no activarlo. Un aviso importante antes de darle: al activarlo, **todo miembro o colaborador externo que no tenga 2FA configurado es expulsado automáticamente** de la organización. No se le bloquea: se le quita. Recuperarlo es reinvitarlo, y pierde por el camino cosas como sus asignaciones.

El procedimiento correcto: avisar al equipo, dar un plazo, comprobar quién falta y solo entonces activar.

```bash
# Antes de activar: quién no tiene 2FA todavía
gh api "/orgs/acme-store/members?filter=2fa_disabled" --jq ".[].login"
```

Si la respuesta es vacía, puedes activar sin expulsar a nadie.

## Paso 5 — Bajar los permisos del token de los workflows

*Settings → Actions → General → Workflow permissions*.

Cada ejecución de un workflow recibe un token efímero, `GITHUB_TOKEN`. Históricamente el valor por defecto era **lectura y escritura sobre todo el repositorio**, lo que significa que cualquier workflow (incluido uno que solo corre tests) puede empujar commits, cerrar issues o publicar releases. Si una dependencia comprometida se ejecuta dentro de ese job, hereda ese poder.

El ajuste correcto es **read-only por defecto**, y que cada workflow pida explícitamente lo que necesita:

```yaml
# El workflow declara sus propios permisos, mínimos y visibles en el YAML
permissions:
  contents: read          # clonar el código
  packages: write         # publicar la imagen en ghcr.io

jobs:
  publicar:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # con solo los permisos declarados arriba
```

La ventaja va más allá de la seguridad: leyendo el bloque `permissions` sabes de un vistazo qué es capaz de hacer ese workflow, sin ir a mirar la configuración de la organización.

En la misma pantalla decides también **qué acciones de terceros se permiten**. La opción recomendable en un entorno profesional es limitar a las acciones creadas por GitHub más una lista explícita de terceros verificados, en lugar de permitir cualquier acción del Marketplace: cada `uses:` de un tercero es código ajeno ejecutándose con acceso a tus secretos.

## Paso 6 — Crear el repositorio `.github`

Un repositorio llamado exactamente `.github` dentro de la organización funciona como plantilla común: lo que pongas ahí se aplica por defecto a todos los repositorios que no tengan su propia versión.

```
.github/
├── profile/README.md              ← portada pública de la organización
├── CODEOWNERS                     ← propietarios por defecto
├── PULL_REQUEST_TEMPLATE.md       ← plantilla de PR para toda la organización
├── ISSUE_TEMPLATE/
│   ├── bug.yml
│   └── feature.yml
└── workflows/                     ← workflows reutilizables (llamados con `uses:`)
```

Es la forma barata de que todos los repositorios compartan la misma plantilla de pull request y los mismos formularios de issue sin copiar y pegar nada. Los workflows que pongas aquí pueden invocarse desde cualquier repositorio de la organización con `uses: acme-store/.github/.github/workflows/tests.yml@main`.

## Paso 7 — Nombrar un segundo Owner

El paso que más se olvida y el más importante. Una organización con un único Owner no ha resuelto el problema que la motivó: sigue habiendo una sola persona capaz de administrar.

```bash
# Invitar a alguien directamente como Owner
gh api -X PUT /orgs/acme-store/memberships/rmartin -f role=admin
```

En la API de organizaciones, el rol Owner se llama `admin` (`member` es el otro valor posible). Es una asimetría de nombres que despista: en la interfaz web verás *Owner*.

Dos Owners es el mínimo. Y que estén en zonas horarias o calendarios de vacaciones distintos no es un detalle menor.

## Lista de comprobación

Antes de meter el primer repositorio de verdad:

- [ ] Nombre de la organización estable y en minúsculas.
- [ ] Correo de facturación a un alias de empresa, no personal.
- [ ] **Dos Owners** como mínimo.
- [ ] *Base permissions* decidido a conciencia (`none` si hay varios clientes).
- [ ] Creación de repositorios públicos restringida.
- [ ] 2FA obligatorio activado (después de comprobar que nadie va a caer).
- [ ] `GITHUB_TOKEN` en read-only por defecto.
- [ ] Acciones de terceros limitadas a una lista permitida.
- [ ] Repositorio `.github` creado con plantillas de PR e issues.
- [ ] Un ruleset de organización que proteja `main` en todos los repositorios (ver la ficha de [Rulesets y protección de ramas](Rulesets-y-proteccion-de-ramas.md)).

## Buenas prácticas avanzadas

- **Configura la organización vacía.** Endurecer los ajustes con la organización a cero personas y cero repositorios no rompe nada ni molesta a nadie. Cada semana que pasa, cada ajuste pendiente se vuelve una conversación con más gente afectada.
- **Elige `No permission` como base si tienes más de un cliente.** No es paranoia: es que un miembro que trabaja para el cliente A no debería poder leer el código del cliente B, y con base `Read` puede. Esto suele estar escrito en los contratos de confidencialidad que la propia empresa ha firmado.
- **Comprueba quién no tiene 2FA *antes* de exigirlo.** Activarlo a ciegas expulsa gente de forma silenciosa a mitad de un sprint, y lo descubres cuando alguien te dice que ya no ve los repositorios. Dos comandos de comprobación evitan la mañana entera de reinvitaciones.
- **Restringe las acciones de terceros desde el principio.** Un `uses: cualquiera/accion@v1` sin fijar es código de otra persona ejecutándose con acceso a los secretos del job, y `@v1` es una etiqueta móvil que puede cambiar bajo tus pies. Limitar el Marketplace a una lista permitida y fijar las acciones por SHA es una de las mejores relaciones esfuerzo/beneficio de toda la configuración.
- **Trata el repositorio `.github` como código de infraestructura.** Con revisión por pull request y `CODEOWNERS` propio. Es un repositorio cuyo contenido se aplica automáticamente al resto de la organización: quien pueda modificarlo sin revisión puede cambiar el comportamiento por defecto de todos los proyectos.

## Recursos didácticos

- [Crear una organización](https://docs.github.com/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch) — el procedimiento oficial paso a paso.
- [Endurecer la seguridad de GitHub Actions](https://docs.github.com/actions/security-guides/security-hardening-for-github-actions) — la guía que explica por qué el token en read-only y el fijado por SHA importan tanto.
- `gh api /orgs/ORG` — la forma más rápida de auditar de golpe todos los ajustes de una organización, incluidos los que la interfaz web reparte por cinco pantallas distintas.

---

*En resumen: crear la organización son dos minutos; configurarla mientras está vacía es lo que te ahorra los seis meses siguientes.*
