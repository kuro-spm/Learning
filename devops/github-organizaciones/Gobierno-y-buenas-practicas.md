# Gobierno y buenas prácticas

## ¿Qué es?

El conjunto de convenciones, plantillas y rutinas que hacen que una organización de GitHub siga siendo comprensible cuando crece: cómo se nombran las cosas, quién decide qué, qué se comparte entre repositorios y con qué frecuencia se revisa que todo siga teniendo sentido.

## ¿Por qué existe?

La configuración de una organización no se degrada de golpe, se degrada por acumulación. Alguien concede un acceso directo "solo por hoy" y nadie lo retira. Se crea un equipo para un proyecto que terminó hace un año. Un colaborador externo sigue en la lista dos empresas después. Se copia y pega el mismo workflow en ocho repositorios y ahora hay ocho versiones ligeramente distintas de lo mismo.

Ninguno de esos hechos es un problema el día que ocurre. El problema es el estado al que llevan: una organización donde nadie puede responder "¿quién tiene acceso a qué?" sin una investigación, y donde cambiar una política significa tocar quince sitios. El gobierno es lo que evita llegar ahí, y consiste sobre todo en decidir pocas cosas y revisarlas de vez en cuando.

> Si ya conoces la deuda técnica en el código, esto es exactamente lo mismo aplicado a la configuración: cada atajo es barato hoy y se cobra con intereses cuando hay que entender el conjunto.

## ¿Cuándo y para qué se usa?

Las convenciones se establecen al principio, cuando no cuestan nada. Las rutinas de revisión se instalan en cuanto la organización pasa de un puñado de repositorios. Y el momento en que todo esto se agradece de verdad es cuando alguien nuevo se incorpora, o cuando alguien se va en mitad de un proyecto.

## Convenciones de nombres

Poco glamuroso y de rendimiento inmediato, porque los nombres se leen mil veces y se escriben una.

**Repositorios**: minúsculas y guiones, y un prefijo o sufijo que diga qué tipo de cosa es. Que la lista de repositorios se pueda ordenar alfabéticamente y tenga sentido no es estética: es lo que permite escribir patrones para los rulesets.

```
api-pedidos           web-tienda            infra-terraform
api-facturacion       web-admin             sandbox-pruebas-ana
```

Ese último prefijo, `sandbox-`, tiene una función concreta: permite excluir los repositorios de experimentos de las reglas estrictas con un patrón (`exclude: ["sandbox-*"]`) en lugar de mantener una lista a mano.

**Equipos**: el nombre de la función o del producto, en minúsculas, sin prefijos redundantes. `backend`, no `equipo-backend-acme`; el contexto ya lo da la organización cuando se menciona como `@acme-store/backend`.

**Secrets y variables**: mayúsculas con guiones bajos, con el recurso primero. `VPS_HOST`, `VPS_USER`, `GHCR_TOKEN`, `AZURE_CLIENT_ID`. Con el recurso delante, la lista se agrupa sola al ordenarse y ves de un golpe todo lo que pertenece al mismo sistema.

## El repositorio `.github`

Ya mencionado al crear la organización, pero merece verse como lo que es: **el mecanismo de "no te repitas" de la configuración**. Un repositorio llamado exactamente `.github` proporciona valores por defecto a todos los demás.

```
.github/
├── profile/README.md              ← portada de la organización
├── CODEOWNERS                     ← propiedad por defecto
├── PULL_REQUEST_TEMPLATE.md
├── ISSUE_TEMPLATE/{bug,feature}.yml
├── dependabot.yml
└── workflows/
    ├── tests-dotnet.yml           ← reutilizable
    └── tests-node.yml
```

La pieza que más ahorra son los **workflows reutilizables**. En lugar de copiar el pipeline de tests en cada repositorio, se define una vez con `workflow_call`:

```yaml
# .github/workflows/tests-dotnet.yml (en el repositorio .github)
on:
  workflow_call:
    inputs:
      ruta:
        required: true
        type: string

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '10.x', cache: true }
      - run: dotnet test ${{ inputs.ruta }}
```

Y cada repositorio lo consume en cuatro líneas:

```yaml
# .github/workflows/ci.yml (en api-pedidos)
on: [push, pull_request]

jobs:
  tests:
    uses: acme-store/.github/.github/workflows/tests-dotnet.yml@main
    with:
      ruta: backend/
```

La ruta repetida (`acme-store/.github/.github/workflows/...`) no es un error tipográfico: el primer `.github` es el nombre del repositorio y el segundo la carpeta dentro de él. Despista siempre la primera vez.

El beneficio real llega el día que hay que cambiar la versión del SDK: se toca un fichero y los quince repositorios quedan actualizados. Con copiar y pegar, ese día son quince pull requests y tres repositorios que se quedan atrás sin que nadie lo note.

## Documentar las decisiones

Una organización acumula decisiones cuyo motivo se olvida: por qué el permiso base es `none`, por qué `main` exige dos aprobaciones, por qué las acciones de terceros están limitadas. Sin el motivo escrito, en seis meses alguien "simplifica" la política porque le estorba, sin saber qué estaba previniendo.

Los **ADR** (*Architecture Decision Records*) son el formato barato para esto: un fichero corto por decisión, con contexto, decisión y consecuencias, en `docs/adr/` del repositorio `.github` o de un repositorio de documentación. La regla útil: si alguien preguntó "¿por qué está así?" y la respuesta tardó más de un minuto, merece un ADR.

## Las rutinas de revisión

El gobierno no es un documento, son unas cuantas comprobaciones repetidas. Estas caben en media hora al trimestre.

**Accesos** — la que más rinde:

```bash
# Colaboradores externos: la lista que más sorpresas da
gh api /orgs/acme-store/outside_collaborators --jq ".[].login"

# Quién es Owner (que no crezca sin darse cuenta)
gh api "/orgs/acme-store/members?role=admin" --jq ".[].login"

# Concesiones directas en un repositorio: deberían ser cero si todo va por equipos
gh api /repos/acme-store/api-pedidos/collaborators \
  --jq '.[] | select(.role_name != "read") | .login + " -> " + .role_name'
```

**Repositorios** — buscar los que están sin protección o abandonados:

```bash
# Todos los repositorios con su visibilidad y su última actualización
gh repo list acme-store --limit 100 \
  --json name,visibility,updatedAt,isArchived \
  --jq '.[] | select(.isArchived == false) | .name + "  " + .visibility + "  " + .updatedAt'
```

Lo que no se toca desde hace un año, **archívalo**. Un repositorio archivado pasa a solo lectura: sigue consultable y deja de ser superficie de ataque y de ruido. Es la operación de higiene más subestimada.

**Coste** — minutos de Actions y almacenamiento (ver [Planes y facturación](Planes-y-facturacion.md)).

**Políticas** — que los ajustes sigan donde los dejaste:

```bash
gh api /orgs/acme-store --jq \
  '{base: .default_repository_permission, dosFA: .two_factor_requirement_enabled,
    creanRepos: .members_can_create_repositories}'
```

## Incorporación y salida

Dos listas cortas que evitan la mayoría de los problemas.

**Al incorporar a alguien:**

- [ ] Invitación como `Member` (no Owner por defecto).
- [ ] Añadir a los **equipos** que le corresponden. Ninguna concesión directa.
- [ ] Comprobar que tiene 2FA (si no, no podrá entrar).
- [ ] Enlazarle las convenciones: formato de rama, formato de commit, flujo de pull request.

**Al dar de baja:**

- [ ] Quitar de la organización.
- [ ] Comprobar que no queda como colaborador directo en ningún repositorio.
- [ ] **Revocar sus PATs y deploy keys.** El punto que se olvida siempre.
- [ ] **Rotar los secretos que pudo ver.** Si tuvo Admin, asume que los vio.
- [ ] Reasignar sus issues y pull requests abiertos.
- [ ] **Quitarla de `CODEOWNERS`**, o los pull requests que toquen sus rutas quedarán bloqueados esperando una aprobación imposible.

## Los tres antipatrones más frecuentes

**El Owner único.** Es el que motivó crear la organización y el que reaparece por inercia: una organización con quince personas y un solo Owner tiene el mismo punto único de fallo que la cuenta personal de la que se huyó. Mínimo dos, con vacaciones no solapadas.

**El acceso concedido "temporalmente".** Nadie retira nunca un acceso temporal, porque no hay ninguna señal que avise. La única defensa realista es la revisión periódica; si de verdad es temporal, apúntalo en algún sitio con fecha el mismo día que lo concedes.

**La política documentada que no existe.** El más peligroso de los tres, porque genera confianza infundada. Escribir en un ADR que "la rama de producción está protegida" cuando el plan no permite protegerla significa que todo el equipo actúa como si hubiera una red debajo. Si una regla no se puede imponer, documéntala como convención y di explícitamente que no está impuesta.

## Buenas prácticas avanzadas

- **Convierte el gobierno en comandos, no en un documento.** Un fichero titulado "política de accesos" no detecta nada. Un script de diez líneas con las cuatro llamadas a `gh api` de arriba, ejecutado cada trimestre —o desde un workflow programado que abre un issue con el resultado—, sí. La diferencia entre gobierno real y gobierno declarativo es si hay una salida que alguien lee.
- **Archiva agresivamente.** Es la operación de más valor y menos coste: un repositorio archivado deja de aparecer en las búsquedas, deja de recibir alertas de dependencias que nadie va a atender y deja de ser una vía de entrada, sin perder ni un byte de historial. Si dudas, archiva: desarchivar es un clic.
- **Nombra los repositorios pensando en los patrones de los rulesets.** Un prefijo `sandbox-` para lo experimental permite escribir un ruleset de organización que aplique a `~ALL` con esa exclusión, en lugar de mantener a mano una lista de repositorios exentos que se queda obsoleta el día que alguien crea el siguiente.
- **Mueve los workflows compartidos al repositorio `.github` en cuanto el mismo YAML esté en tres sitios.** Tres es el umbral donde copiar y pegar empieza a costar más que centralizar. Y el síntoma de haber esperado demasiado es reconocible: la versión del SDK es distinta en cada repositorio y nadie sabe cuál es la correcta.
- **Escribe el motivo, no la regla.** "Base permission = none" es un ajuste que cualquiera puede leer de la configuración; "base permission = none porque distintos repositorios pertenecen a clientes distintos y el contrato de confidencialidad lo exige" es lo que impide que alguien lo relaje dentro de un año para quitarse fricción. La configuración se puede consultar; la razón, no.
- **Trata el repositorio `.github` como código de producción.** Su contenido se aplica automáticamente al resto de la organización, y sus workflows reutilizables se ejecutan en todos los repositorios que los invocan. Quien pueda modificarlo sin revisión puede cambiar el comportamiento de todos los pipelines a la vez: es el repositorio que más merece `CODEOWNERS` y revisión obligatoria de dos equipos.

## Recursos didácticos

- [Buenas prácticas para organizaciones](https://docs.github.com/organizations/collaborating-with-groups-in-organizations/best-practices-for-organizations) — recomendaciones oficiales de estructura y gobierno.
- [Workflows reutilizables](https://docs.github.com/actions/using-workflows/reusing-workflows) — la referencia de `workflow_call`, con el paso de secretos e *inputs*.
- [adr.github.io](https://adr.github.io/) — plantillas y ejemplos de Architecture Decision Records, para no inventar el formato desde cero.

---

*En resumen: el gobierno no es un documento de políticas, son cuatro comandos que alguien ejecuta cada trimestre y un repositorio `.github` que evita quince copias de lo mismo.*
