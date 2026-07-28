# Plugin marketplaces

## ¿Qué es?

Un **plugin marketplace** es un catálogo que distribuye extensiones de Claude Code. Es un fichero JSON, `marketplace.json`, publicado normalmente en un repositorio de Git, que lista uno o varios **plugins** y dice dónde está cada uno. Quien quiera usarlos añade el catálogo una vez e instala los plugins que le interesen.

## ¿Por qué existe?

Porque personalizar un agente de codificación es fácil, pero **compartir** esa personalización no lo era. Todo lo que puedes añadir a Claude Code —una *skill*, un subagente, un *hook* que pasa el linter después de cada edición, un servidor MCP preconfigurado— vive en el directorio `.claude/` de tu máquina o de tu repositorio. Eso funciona perfectamente para ti, y nada más: si quieres que lo use el resto del equipo, la única opción es copiar carpetas a mano, sin versión, sin actualizaciones y sin forma de saber quién tiene qué.

El marketplace convierte eso en una dependencia normal: se publica en un repositorio, se instala con un comando, se actualiza sola y tiene versiones.

> Es la misma jugada que npm o NuGet, con dos diferencias que importan. La primera: aquí **el catálogo lo publicas tú**, no hay un registro central obligatorio — un marketplace puede ser un repositorio privado de tu organización con tres plugins internos. La segunda: un plugin no es una librería que tu código llama, es **configuración e instrucciones que el agente ejecuta**, así que la analogía con las dependencias se rompe exactamente donde hablaremos de seguridad.

## ¿Cuándo y para qué se usa?

- **Estandarizar el uso del agente en un equipo.** El *hook* que impide comitear sin pasar los tests, el subagente revisor con los criterios de la casa, la *skill* que sabe cómo se despliega aquí. Se publica una vez y todo el mundo lo tiene igual.
- **Reutilizar tus propias personalizaciones entre proyectos.** Si has escrito una *skill* buena, un marketplace personal —aunque sea un repositorio con un solo plugin— te la deja instalada en todas tus máquinas y todos tus repositorios.
- **Consumir extensiones ya hechas.** Hay integraciones publicadas para GitHub, GitLab, Jira, Linear, Sentry o Figma, y plugins de LSP que dan al agente navegación de código y errores de tipos reales.
- **Repartir configuración distinta a grupos distintos.** Un canal «estable» y otro «al día» del mismo plugin, asignados a equipos diferentes.

Y **no** se usa cuando la personalización es tuya y de un solo proyecto. Para eso, `.claude/` es más simple: sin manifiesto, sin catálogo, sin instalación, y las *skills* se invocan con nombre corto (`/desplegar`) en vez del nombre con espacio de nombres (`/mi-plugin:desplegar`). El marketplace paga cuando hay al menos dos destinatarios.

---

## Primero: qué es un plugin

No se puede entender un catálogo sin entender qué cataloga. Un **plugin** es un directorio autocontenido con componentes que Claude Code sabe cargar. No hace falta que tenga todos; lo normal es que tenga uno o dos.

```text
quality-review/                  ← raíz del plugin
├── .claude-plugin/
│   └── plugin.json              ← manifiesto: identidad del plugin
├── skills/
│   └── revision/
│       └── SKILL.md             ← se invoca como /quality-review:revision
├── agents/
│   └── revisor-seguridad.md     ← subagente
├── hooks/
│   └── hooks.json               ← manejadores de eventos
├── .mcp.json                    ← servidores MCP preconfigurados
└── bin/                         ← ejecutables que se añaden al PATH
```

La regla que más gente se salta está en la primera línea de ese árbol: **dentro de `.claude-plugin/` va únicamente `plugin.json`**. Las carpetas `skills/`, `agents/`, `hooks/` y el resto van en la raíz del plugin, no dentro del directorio del manifiesto. Un plugin con `skills/` metido dentro de `.claude-plugin/` carga sin errores visibles y sin ninguna *skill*.

El manifiesto es corto:

```json
{
  "name": "quality-review",
  "description": "Revisión de calidad y seguridad antes de comitear",
  "version": "1.0.0",
  "author": { "name": "Equipo de Plataforma" }
}
```

Qué hace cada campo:

- **`name`** es el identificador y, sobre todo, el **espacio de nombres**. Las *skills* del plugin se invocan con ese prefijo: la carpeta `skills/revision/` de este plugin se llama `/quality-review:revision`. El prefijo no es opcional ni decorativo — evita que dos plugins con una *skill* llamada `revision` choquen entre sí.
- **`description`** es lo que se lee al navegar el catálogo antes de instalar.
- **`version`** es opcional y tiene más consecuencias de las que parece; volveremos a ello en «Versiones y actualizaciones».
- **`author`**, `homepage`, `repository`, `license` y `keywords` son metadatos.

Para probar un plugin en desarrollo no hace falta catálogo ni instalación: se arranca el agente apuntando a su directorio.

```bash
claude --plugin-dir ./quality-review
```

Con eso el plugin se carga solo para esa sesión. Dentro, `/reload-plugins` recoge los cambios que vayas haciendo sin reiniciar. Si el plugin ya está instalado desde un marketplace, la copia local tiene prioridad durante esa sesión, que es justo lo que quieres para probar un cambio sin desinstalar nada.

---

## El catálogo: `marketplace.json`

El marketplace es un fichero en `.claude-plugin/marketplace.json`, en la raíz del repositorio que lo publica. Tres campos son obligatorios: `name`, `owner` y `plugins`.

```json
{
  "name": "acme-tools",
  "owner": {
    "name": "Equipo de Plataforma",
    "email": "plataforma@acme.example"
  },
  "description": "Extensiones internas de Acme para Claude Code",
  "plugins": [
    {
      "name": "quality-review",
      "source": "./plugins/quality-review",
      "description": "Revisión de calidad y seguridad antes de comitear",
      "version": "1.0.0"
    }
  ]
}
```

Lo que hay que retener de ese fichero:

- **`name` es público y es el sufijo de instalación.** Quien instale ese plugin escribirá `quality-review@acme-tools`. Y ojo con una consecuencia poco intuitiva: **cada usuario solo puede tener registrado un marketplace por nombre**. Si añade un segundo con el mismo `name`, reemplaza al primero. Para publicar varios plugins bajo un mismo nombre, van todos en el mismo `marketplace.json`; no se hacen dos marketplaces con el mismo nombre.
- **`source` dice de dónde se saca ese plugin concreto**, y no tiene que ser este repositorio. Es la sección siguiente.
- El resto de campos de una entrada (`author`, `homepage`, `license`, `category`, `tags`, `displayName`…) son metadatos de catálogo.

Hay un campo opcional que ahorra ruido cuando tienes varios plugins en el mismo repositorio, `metadata.pluginRoot`, que prefija las rutas relativas:

```json
{
  "name": "acme-tools",
  "owner": { "name": "Equipo de Plataforma" },
  "metadata": { "pluginRoot": "./plugins" },
  "plugins": [
    { "name": "quality-review", "source": "quality-review" },
    { "name": "deploy-tools",   "source": "deploy-tools" }
  ]
}
```

Con `pluginRoot` puesto, `"source": "quality-review"` resuelve a `./plugins/quality-review`. Sin él, cada entrada repite el prefijo.

> **Nombres reservados.** Hay una lista de nombres que no puedes usar porque están reservados para marketplaces oficiales de Anthropic: `claude-plugins-official`, `claude-plugins-community`, `claude-community`, `anthropic-plugins`, `agent-skills` y varios más. También se bloquean los nombres que imitan a uno oficial, tipo `official-claude-plugins`. La comprobación se hace **cada vez que se carga el marketplace**, no solo al añadirlo, así que un nombre que pasa a estar reservado deja de cargar y reporta que el marketplace viene de un origen no confiable. Es una defensa contra el marketplace de terceros que se presenta como fuente oficial.

---

## De dónde viene cada plugin: el campo `source`

Aquí está la pieza que hace que un marketplace sea un catálogo y no un monorepo. **El catálogo y los plugins pueden vivir en sitios distintos**, y cada uno se ancla por separado. Cinco tipos de origen:

| Tipo | Forma | Para qué |
|---|---|---|
| Ruta relativa | `"./plugins/quality-review"` | el plugin vive en el mismo repositorio que el catálogo |
| `github` | `{ "source": "github", "repo": "acme-corp/quality-review" }` | el plugin tiene su propio repositorio en GitHub |
| `url` | `{ "source": "url", "url": "https://gitlab.com/acme/plugin.git" }` | cualquier host de Git: GitLab, Bitbucket, servidor propio |
| `git-subdir` | `{ "source": "git-subdir", "url": "...", "path": "tools/plugin" }` | el plugin está en un subdirectorio de un monorepo |
| `npm` | `{ "source": "npm", "package": "@acme/claude-plugin" }` | distribuido como paquete npm, público o de registro privado |

La ruta relativa es la más común para empezar, y tiene dos reglas propias. Resuelve **contra la raíz del marketplace** (el directorio que contiene `.claude-plugin/`), no contra el directorio del fichero:

```json
{ "name": "quality-review", "source": "./plugins/quality-review" }
```

Aunque `marketplace.json` esté en `<repo>/.claude-plugin/marketplace.json`, esa ruta apunta a `<repo>/plugins/quality-review`. Y la segunda regla: **no se puede usar `../`**. El validador lo rechaza con `plugins[0].source: Path contains ".."`.

Los tres orígenes de Git —`github`, `url` y `git-subdir`— aceptan `ref` (rama o etiqueta) y `sha` (*commit* exacto de 40 caracteres):

```json
{
  "name": "quality-review",
  "source": {
    "source": "github",
    "repo": "acme-corp/quality-review",
    "ref": "v2.0.0",
    "sha": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0"
  }
}
```

Cuando pones los dos, **el `sha` es el ancla efectiva**: se descarga y se hace *checkout* de ese *commit* directamente. La ventaja práctica es que en GitHub, GitLab y Bitbucket la instalación sigue funcionando aunque la rama o la etiqueta que nombra `ref` se haya borrado, siempre que el *commit* siga siendo alcanzable. En servidores que no permiten traer *commits* por SHA, como AWS CodeCommit, el `ref` tiene que seguir existiendo.

> **Dos cosas que se llaman «source» y no son lo mismo.** El **origen del marketplace** es de dónde se descarga el `marketplace.json`, y se fija al añadirlo (acepta `ref`, pero no `sha`). El **origen del plugin** es de dónde se descarga cada plugin listado dentro, y se fija en su entrada (acepta `ref` y `sha`). Un catálogo alojado en `acme-corp/plugin-catalog` puede listar un plugin que se descarga de `acme-corp/quality-review`: son repositorios distintos y se anclan por separado.

---

## Montar un marketplace de cero

Con las piezas explicadas, el recorrido completo son cuatro ficheros. Partimos de un directorio vacío:

```bash
mkdir -p acme-tools/.claude-plugin
mkdir -p acme-tools/plugins/quality-review/.claude-plugin
mkdir -p acme-tools/plugins/quality-review/skills/revision
```

**Paso 1: la *skill*.** Es el contenido útil del plugin — lo que el agente acabará haciendo. En `acme-tools/plugins/quality-review/skills/revision/SKILL.md`:

```markdown
---
description: Revisa los cambios pendientes buscando errores, problemas de seguridad y rendimiento
---

Revisa los cambios pendientes del working tree y busca:
- Errores potenciales y casos límite no cubiertos
- Problemas de seguridad
- Consultas a base de datos dentro de bucles
- Nombres y estructura que dificulten la lectura

Sé concreto: cita fichero y línea, y propón el cambio.
```

El `description` del *frontmatter* es lo que decide que el agente use la *skill* en el momento adecuado, así que dice **cuándo** aplicarla, no solo qué hace.

**Paso 2: el manifiesto del plugin.** En `acme-tools/plugins/quality-review/.claude-plugin/plugin.json`:

```json
{
  "name": "quality-review",
  "description": "Revisión de calidad y seguridad antes de comitear",
  "version": "1.0.0",
  "author": { "name": "Equipo de Plataforma" }
}
```

**Paso 3: el catálogo.** En `acme-tools/.claude-plugin/marketplace.json`:

```json
{
  "name": "acme-tools",
  "owner": { "name": "Equipo de Plataforma" },
  "description": "Extensiones internas de Acme para Claude Code",
  "plugins": [
    {
      "name": "quality-review",
      "source": "./plugins/quality-review",
      "description": "Revisión de calidad y seguridad antes de comitear"
    }
  ]
}
```

**Paso 4: validar antes de compartir.** Este comando es el que separa «lo publiqué y no cargaba» de «lo publiqué y funcionó»:

```bash
claude plugin validate .
```

Apuntado a un directorio de marketplace, comprueba el esquema de `marketplace.json`, detecta nombres de plugin duplicados y rechaza rutas con `..`. Para cada entrada con `source` local valida además el `plugin.json` de ese plugin y avisa si la `version` de la entrada no coincide con la del manifiesto. Los problemas de un plugin concreto salen prefijados con su índice:

```text
plugins[0] plugin.json → Invalid JSON syntax: Unexpected token '}' at position 142
```

Para validar también el *frontmatter* de las *skills*, los agentes y los *hooks* de un plugin, se apunta al directorio del plugin:

```bash
claude plugin validate ./plugins/quality-review
```

**Paso 5: probarlo en local.** Desde el directorio que contiene `acme-tools`:

```shell
/plugin marketplace add ./acme-tools
/plugin install quality-review@acme-tools
/reload-plugins
```

Y ya se puede invocar la *skill*, con su espacio de nombres:

```shell
/quality-review:revision
```

**Paso 6: publicarlo.** Un `git push` a GitHub, GitLab o donde sea, y el resto del mundo lo añade con la forma corta `owner/repo`:

```shell
/plugin marketplace add acme-corp/acme-tools
```

A partir de ahí, publicar una versión nueva es empujar *commits*; quien lo tenga instalado refresca su copia con `/plugin marketplace update`.

> **Los plugins se copian a una caché.** Al instalar, Claude Code copia el directorio del plugin a `~/.claude/plugins/cache`. Esto tiene una consecuencia práctica inmediata: **un plugin no puede referenciar ficheros fuera de su directorio** con rutas tipo `../utilidades-compartidas`, porque esos ficheros no se copian. Para apuntar a ficheros propios desde un *hook* o un servidor MCP se usa la variable `${CLAUDE_PLUGIN_ROOT}`, que resuelve al directorio de instalación real.

Así se ve en una entrada que declara un *hook* y un servidor MCP:

```json
{
  "name": "quality-review",
  "source": { "source": "github", "repo": "acme-corp/quality-review" },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "${CLAUDE_PLUGIN_ROOT}/scripts/validar.sh" }
        ]
      }
    ]
  },
  "mcpServers": {
    "catalogo": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/catalogo",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"]
    }
  }
}
```

Sin `${CLAUDE_PLUGIN_ROOT}`, ese `scripts/validar.sh` se buscaría en el directorio de trabajo de la sesión y no se encontraría nunca. Para datos que deban **sobrevivir a las actualizaciones** del plugin existe `${CLAUDE_PLUGIN_DATA}`, que apunta a un directorio persistente fuera de la caché versionada.

---

## El lado de quien instala

Añadir un marketplace y instalar un plugin son dos pasos separados, y conviene tenerlo claro: **añadir el catálogo no instala nada**. Da acceso a navegarlo.

Las cuatro formas de añadir un catálogo:

```shell
/plugin marketplace add acme-corp/acme-tools                      # GitHub, forma corta
/plugin marketplace add https://gitlab.com/acme/plugins.git       # cualquier host de Git
/plugin marketplace add ./acme-tools                              # directorio local
/plugin marketplace add https://acme.example/marketplace.json     # URL directa al JSON
```

Para anclar a una rama o etiqueta se añade `#ref` a la URL de Git, o `@ref` a la forma corta de GitHub:

```shell
/plugin marketplace add https://gitlab.com/acme/plugins.git#v1.0.0
/plugin marketplace add acme-corp/acme-tools@v2.0
```

Dos avisos sobre esas formas. La URL **necesita su esquema**: `gitlab.example.com/acme/plugins` sin `https://` se interpreta como una forma corta de GitHub mal escrita y se rechaza. Y la cuarta forma, la URL directa al JSON, tiene una limitación importante que veremos en los errores frecuentes: solo descarga ese fichero.

Instalar un plugin usa el identificador `plugin@marketplace`:

```shell
/plugin install quality-review@acme-tools
```

Eso abre la vista de detalle, donde se elige el **alcance** de la instalación. Los tres alcances no son un detalle administrativo: deciden quién más se ve afectado.

| Alcance | Dónde se escribe | Efecto |
|---|---|---|
| **User** | configuración de usuario | para ti, en todos tus proyectos |
| **Project** | `.claude/settings.json` del repositorio | para todo el mundo que colabore en ese repositorio |
| **Local** | `.claude/settings.local.json` | para ti, solo en ese repositorio |

El alcance *project* es el que se comparte por Git, así que es el que se usa para estandarizar un equipo — y también el que hay que pensar dos veces antes de elegir, porque instala en la máquina de otras personas. Puede aparecer además un alcance **managed**, que instala un administrador por política y no se puede modificar.

Después de instalar hace falta activar el plugin en la sesión en curso:

```shell
/reload-plugins
```

Recarga plugins, *skills*, agentes, *hooks* y los servidores MCP y LSP de los plugins, y muestra el recuento de cada cosa. Sin eso, el plugin está instalado pero no cargado hasta la siguiente sesión.

La gestión del día a día se hace desde `/plugin`, que abre un panel con cuatro pestañas —**Discover**, **Installed**, **Marketplaces** y **Errors**—, o con comandos directos:

```shell
/plugin list                                   # instalados (--enabled / --disabled)
/plugin disable quality-review@acme-tools      # desactivar sin desinstalar
/plugin enable  quality-review@acme-tools
/plugin uninstall quality-review@acme-tools
/plugin marketplace list
/plugin marketplace update acme-tools          # refrescar el catálogo
/plugin marketplace remove acme-tools
```

Todos tienen equivalente no interactivo con `claude plugin …` para *scripts* y CI, y ahí el alcance se pasa como opción:

```bash
claude plugin install quality-review@acme-tools --scope project
claude plugin marketplace add acme-corp/acme-tools --scope project
```

> **Quitar un marketplace desinstala sus plugins.** `/plugin marketplace remove` no solo desregistra el catálogo: se lleva por delante los plugins que hubieras instalado desde él. Si lo que quieres es refrescar el catálogo, el comando es `update`, no `remove` seguido de `add`.

---

## Distribuirlo a un equipo sin pedirle a nadie que ejecute comandos

Esta es la parte que convierte un marketplace en una decisión de equipo. En el `.claude/settings.json` del repositorio se declara qué catálogos hacen falta:

```json
{
  "extraKnownMarketplaces": {
    "acme-tools": {
      "source": {
        "source": "github",
        "repo": "acme-corp/acme-tools"
      },
      "autoUpdate": true
    }
  },
  "enabledPlugins": {
    "quality-review@acme-tools": true,
    "deploy-tools@acme-tools": true
  }
}
```

Ese fichero va versionado con el proyecto. Cuando alguien clona el repositorio y confía en la carpeta, Claude Code le propone instalar el marketplace y los plugins declarados. `enabledPlugins` dice cuáles deben quedar activos.

Un detalle que confunde: un plugin declarado solo en `enabledPlugins` del proyecto, y cuyo origen es externo (GitHub, npm), **no se carga hasta que se instala**. Claude Code lo reporta como no instalado y muestra el comando `claude plugin install` que hay que ejecutar. La declaración expresa la intención; no descarga nada por sí sola.

Sobre las **actualizaciones automáticas**: cuando están activas para un marketplace, Claude Code refresca el catálogo y actualiza los plugins instalados en segundo plano después de arrancar, con un retardo aleatorio de hasta diez minutos —de forma que la sesión en curso sigue usando las versiones que cargó al arrancar—. Si algo se actualiza, avisa para que ejecutes `/reload-plugins`. Los marketplaces oficiales de Anthropic vienen con auto-actualización activada; los de terceros y los locales de desarrollo, desactivada. Un administrador puede forzarla con `"autoUpdate": true` en la entrada de `extraKnownMarketplaces` de la configuración gestionada.

Para entornos de contenedores y CI hay una vía que evita clonar nada en tiempo de ejecución: preparar un directorio semilla en tiempo de construcción de la imagen y apuntar `CLAUDE_CODE_PLUGIN_SEED_DIR` a él. Los marketplaces de la semilla son de solo lectura, tienen prioridad sobre la configuración del usuario y no se auto-actualizan.

```bash
# Durante el build de la imagen: instalar directamente en la ruta semilla
CLAUDE_CODE_PLUGIN_CACHE_DIR=/opt/claude-seed claude plugin marketplace add acme-corp/acme-tools
CLAUDE_CODE_PLUGIN_CACHE_DIR=/opt/claude-seed claude plugin install quality-review@acme-tools

# En el entorno de ejecución del contenedor
export CLAUDE_CODE_PLUGIN_SEED_DIR=/opt/claude-seed
```

Con eso, el contenedor arranca con los plugins ya disponibles y sin dependencia de red.

---

## Versiones y actualizaciones

Aquí está la trampa que más tiempo hace perder de todo el sistema de plugins, así que merece la pena entender el mecanismo antes de elegir.

**La versión de un plugin determina la ruta de la caché y la detección de cambios.** Si la versión que se resuelve coincide con la que ya tienes, `/plugin update` y la auto-actualización **se saltan el plugin**. No comparan contenido: comparan versión.

Claude Code resuelve la versión con el primero de estos que exista:

1. `version` en el `plugin.json` del plugin
2. `version` en la entrada del marketplace
3. El SHA del *commit* del origen del plugin

De ahí salen dos estrategias, y hay que elegir una a conciencia:

- **Omitir `version` por completo.** Con orígenes basados en Git —`github`, `url`, `git-subdir` y rutas relativas dentro de un marketplace alojado en Git— cada *commit* nuevo cuenta como versión nueva. Es la opción simple y correcta para plugins internos o en desarrollo activo: empujas y la gente lo recibe.
- **Declarar `version` y subirla en cada entrega.** Da control y semántica, pero **si no la subes, no pasa nada**: puedes empujar veinte *commits* y quien lo tenga instalado seguirá con la copia cacheada, porque la versión que ve es la misma.

Y un fallo silencioso que conviene grabarse: **no declares `version` en los dos sitios**. Claude Code usa siempre la de `plugin.json`, sin avisar. Un manifiesto con una versión antigua olvidada enmascara la versión que hayas puesto con cuidado en `marketplace.json`, y el síntoma es «actualizo el catálogo y nadie recibe el cambio».

Con esa mecánica clara, los **canales de publicación** salen casi gratis: dos marketplaces que apuntan a *refs* distintos del mismo repositorio.

```json
{
  "name": "acme-stable",
  "owner": { "name": "Equipo de Plataforma" },
  "plugins": [
    {
      "name": "quality-review",
      "source": { "source": "github", "repo": "acme-corp/quality-review", "ref": "stable" }
    }
  ]
}
```

```json
{
  "name": "acme-latest",
  "owner": { "name": "Equipo de Plataforma" },
  "plugins": [
    {
      "name": "quality-review",
      "source": { "source": "github", "repo": "acme-corp/quality-review", "ref": "latest" }
    }
  ]
}
```

Luego se asigna cada catálogo al grupo de usuarios que toque mediante `extraKnownMarketplaces` en la configuración gestionada. La condición para que esto funcione es la de siempre: **cada canal tiene que resolver a una versión distinta**. Si omites `version`, los SHA distintos ya los distinguen. Si usas versiones explícitas, el `plugin.json` de cada *ref* debe declarar una diferente; si los dos *refs* resuelven a la misma cadena, Claude Code los considera idénticos y no actualiza.

### Renombrar o retirar un plugin

El `name` de un plugin es su identificador estable: aparece en `enabledPlugins`, en `pluginConfigs` y en los comandos de instalación de todo el mundo. Cambiarlo rompe todas las instalaciones existentes. Si solo quieres cambiar la etiqueta que se ve en la interfaz, para eso está `displayName`, y `name` no se toca.

Cuando el cambio es inevitable, o cuando retiras un plugin del array, se añade un mapa `renames` en la raíz del catálogo para que la gente migre sola en vez de encontrarse un `plugin-not-found`:

```json
{
  "name": "acme-tools",
  "owner": { "name": "Equipo de Plataforma" },
  "plugins": [
    { "name": "code-review", "source": "./plugins/code-review" }
  ],
  "renames": {
    "quality-review": "code-review",
    "legacy-linter": null
  }
}
```

Ese mapa dice dos cosas: `quality-review` ahora se llama `code-review`, y `legacy-linter` ya no existe. Al arrancar, Claude Code sigue el mapa, carga el plugin bajo su nombre nuevo, muestra un aviso de una línea y **reescribe la clave antigua en la configuración** de los alcances de usuario, proyecto y local, de forma que el aviso aparece una sola vez. Para una entrada `null`, elimina la clave e informa de que el plugin se retiró.

Dos reglas de uso: **`renames` es histórico y se trata como *append-only*** —si más adelante renombras `code-review` a `review-pro`, añades una segunda entrada en vez de editar la primera, porque Claude Code sigue cadenas y así quien tenga el nombre original migra igual—, y después de editar el mapa se pasa `claude plugin validate .`, que rechaza cadenas cíclicas o que no terminen en `null` o en un nombre listado en `plugins`.

---

## Repositorios privados

Un marketplace interno normalmente vive en un repositorio privado, y ahí hay una asimetría que sorprende: **la instalación manual y la actualización automática se autentican de forma distinta**.

La instalación y las actualizaciones manuales usan tus credenciales de Git de siempre: los *credential helpers* configurados, `gh auth login`, el llavero del sistema. Con SSH funciona si el host ya está en `known_hosts` y la clave está cargada en `ssh-agent`, porque Claude Code suprime los diálogos interactivos de SSH.

La actualización **en segundo plano**, en cambio, desactiva los *credential helpers* para su `git pull`. Consecuencia: sobre HTTPS no puede autenticarse contra un repositorio privado aunque tengas un *helper* configurado. Los remotos SSH no se ven afectados. Cuando ese `pull` falla, Claude Code intenta volver a clonar el marketplace desde cero —eso sí usa tus credenciales, pero puede agotar el tiempo de espera en repositorios grandes—, y de ahí vienen los fallos intermitentes.

Dos ajustes lo dejan predecible:

```bash
# Conservar el clon existente cuando falle el pull en segundo plano,
# en vez de borrarlo y volver a clonar
export CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE=1
```

```bash
# Que el pull en segundo plano se autentique por HTTPS, mediante reescritura de URL.
# Ojo al alcance: se acota al repositorio o a la organización, nunca solo al host.
git config --global \
  url."https://x-access-token:TU_TOKEN@github.com/acme-corp/acme-tools".insteadOf \
  "https://github.com/acme-corp/acme-tools"
```

La reescritura funciona porque el token va embebido en la URL del remoto, así que no depende de los *helpers* desactivados. El precio es que el token queda **en texto plano en tu `.gitconfig`**, de modo que debe ser de solo lectura y limitado a ese repositorio. Y el aviso sobre el alcance es serio: una reescritura cuya base sea únicamente el host se aplica a **todos** los *fetch* y *push* a ese host en la máquina, incluidos los de tus propios repositorios, y sustituye tus credenciales normales.

Cada proveedor espera un usuario distinto en la URL reescrita:

| Proveedor | Forma de la URL reescrita |
|---|---|
| GitHub | `https://x-access-token:TOKEN@github.com/acme-corp/acme-tools` |
| GitLab | `https://oauth2:TOKEN@gitlab.com/acme-corp/acme-tools` |
| Bitbucket | `https://x-token-auth:TOKEN@bitbucket.org/acme-corp/acme-tools` |

---

## Errores frecuentes y cómo se diagnostican

**`File not found: .claude-plugin/marketplace.json`.** El catálogo no está donde se busca. El fichero va en `.claude-plugin/marketplace.json` **en la raíz** del repositorio, no en la raíz suelta ni dentro de la carpeta de un plugin.

**Las *skills* del plugin no aparecen.** Por orden de probabilidad: falta `/reload-plugins`; o `skills/` está dentro de `.claude-plugin/` en vez de en la raíz del plugin; o estás invocando sin el espacio de nombres (`/revision` en vez de `/quality-review:revision`). Si nada de eso es, la caché puede estar corrupta:

```bash
rm -rf ~/.claude/plugins/cache
```

Después, reiniciar y reinstalar el plugin.

**«Path not found» al instalar desde un marketplace añadido por URL.** Este es el error que más cuesta ver, y no es culpa de tu JSON. Un marketplace añadido con `/plugin marketplace add https://acme.example/marketplace.json` descarga **solo ese fichero**, nada más. Las rutas relativas tipo `"./plugins/quality-review"` apuntan a ficheros del servidor que no se han descargado, así que no resuelven. Dos salidas: cambiar las entradas a orígenes `github`, `url` o `npm`, o —mejor— alojar el catálogo en un repositorio de Git y añadirlo con la URL del repositorio, que clona todo y hace que las rutas relativas funcionen.

**`Duplicate plugin name "x" found in marketplace`.** Dos entradas comparten `name`. Cada plugin necesita el suyo, y recuerda que el `name` de la entrada del marketplace puede diferir del `name` del `plugin.json`: los comandos `enable`, `disable` y `uninstall` usan el de la entrada.

**`Git clone timed out after 120s`.** El tiempo de espera de las operaciones de Git es de 120 segundos. En repositorios grandes o redes lentas se sube por variable de entorno, en milisegundos:

```bash
export CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS=300000   # 5 minutos
```

**El plugin instala pero sus ficheros no se encuentran.** Rutas que salen del directorio del plugin (`../utilidades`). No se copian a la caché. La solución es `${CLAUDE_PLUGIN_ROOT}` para lo propio y, si de verdad hay que compartir ficheros entre plugins, enlaces simbólicos.

**`YAML frontmatter failed to parse`.** El *frontmatter* de una *skill*, un agente o un comando está mal. En tiempo de ejecución ese fichero carga **sin metadatos**, que es un fallo silencioso feo: la *skill* existe pero sin `description`, así que el modelo no sabe cuándo usarla. Solo se reporta al validar el directorio del plugin, no el del marketplace — razón de más para pasar `claude plugin validate` sobre cada plugin y no solo sobre el catálogo.

**`Invalid JSON syntax` en `hooks.json`.** Un `hooks/hooks.json` malformado **impide cargar el plugin entero**, no solo los *hooks*.

---

## Seguridad: un plugin es código que se ejecuta con tus permisos

Esto no es una sección de cierre por cortesía: es la razón por la que la analogía con npm se rompe.

Un plugin puede traer *hooks* que ejecutan comandos arbitrarios en tu máquina, servidores MCP con acceso a tus sistemas, ejecutables en `bin/` que entran en el `PATH` de la herramienta Bash, y *skills* que son instrucciones que el agente seguirá. Todo eso corre **con tus privilegios de usuario**. Anthropic no controla qué incluye un plugin de terceros ni puede verificar que haga lo que dice. Añadir un marketplace y confiar en quien lo publica es la misma decisión.

Con ese marco, tres medidas concretas.

**Ancla los plugins de terceros que listes en tu catálogo.** Si tu marketplace lista un plugin ajeno con `ref: "main"`, estás distribuyendo a tu equipo lo que haya en la rama principal de otra persona en cada actualización. Con `sha` distribuyes un *commit* que has revisado. Es exactamente la diferencia entre un *lockfile* y no tenerlo, aplicada a algo que ejecuta comandos.

**Restringe los orígenes permitidos si administras una organización.** La configuración gestionada tiene `strictKnownMarketplaces`, que se comprueba **antes de cualquier operación de red o de disco**, y en cada instalación, actualización y auto-actualización, no solo al añadir:

```json
{
  "strictKnownMarketplaces": [
    { "source": "github", "repo": "acme-corp/acme-tools" },
    { "source": "hostPattern", "hostPattern": "^git\\.acme\\.example$" }
  ]
}
```

Los valores posibles son tres: sin definir, no hay restricción; array vacío `[]`, cierre completo y nadie puede añadir marketplaces; una lista, y solo se pueden añadir los que coincidan. Para rechazar además los *flags* de línea de comandos que cargan plugins, agentes y servidores MCP para una sola ejecución, se combina con `disableSideloadFlags`. Y como se define en configuración gestionada, ni los usuarios ni los proyectos pueden saltárselo.

**Trata cada plugin instalado como coste permanente de contexto.** Cada plugin activo aporta *skills*, agentes y herramientas MCP que ocupan sitio en la ventana de contexto **en cada turno**, no solo cuando los usas. La vista de detalle de `/plugin` muestra una estimación de ese coste antes de instalar, y la pestaña **Installed** agrupa bajo **Not used recently** los plugins que no has usado en al menos dos semanas y diez sesiones. Es la lista que hay que revisar de vez en cuando: un plugin que no usas sigue pagando peaje en cada petición. El detalle está en [Ventana de contexto](../context-engineering/Ventana-de-Contexto.md).

---

## Buenas prácticas avanzadas

- **Para plugins internos y vivos, omite `version` por completo y deja que el SHA del *commit* haga de versión.** Es contraintuitivo —parece que declarar versiones es lo profesional— pero el modo SHA convierte «empujar» en «publicar» y elimina de golpe el fallo más común del sistema: subir cambios que nadie recibe porque la cadena de versión no cambió. Reserva las versiones explícitas para plugins con consumidores externos, donde la semántica importa más que la inmediatez.
- **Nunca declares `version` en `plugin.json` y en la entrada del marketplace a la vez.** Claude Code usa siempre la del `plugin.json`, **sin avisar**. Una versión olvidada en el manifiesto enmascara silenciosamente la del catálogo, y el síntoma —«actualizo el catálogo y no llega»— no apunta en absoluto a la causa. Elige un único sitio donde vivan las versiones y respétalo.
- **Ancla con `sha`, no solo con `ref`, todo plugin de terceros que liste tu catálogo.** Un `ref: "main"` distribuye a tu equipo lo que haya hoy en la rama de otra persona, con permiso para ejecutar comandos en sus máquinas. Además, cuando `sha` y `ref` están los dos puestos, la instalación sobrevive a que la rama o la etiqueta desaparezcan del *upstream*, porque el *commit* sigue siendo alcanzable.
- **En `strictKnownMarketplaces`, prefiere `hostPattern` a una URL literal.** La comparación es exacta y **no normaliza URLs**: una barra final, el sufijo `.git` o la forma `ssh://` frente a `https://` son valores distintos. Si el repositorio de tu organización se puede clonar de más de una forma —y casi siempre se puede—, una lista de URLs literales bloquea a gente que está haciendo lo correcto, y el mensaje de error no explica por qué.
- **Trata `renames` como historial inmutable, no como configuración.** Cuando renombres por segunda vez, añade una entrada nueva en lugar de editar la existente: Claude Code sigue cadenas, así que quien todavía tenga el nombre original migra a través de las dos. Editar la primera entrada rompe precisamente a la gente que llevaba más tiempo sin actualizar, que es la que más necesita la migración automática.
- **Pasa `claude plugin validate` sobre cada directorio de plugin, no solo sobre el del marketplace.** Validar el catálogo comprueba el esquema y las rutas, pero el *frontmatter* de las *skills*, los agentes y los `hooks.json` solo se revisan cuando apuntas al plugin. La diferencia importa porque los dos fallos que esa pasada detecta son silenciosos en producción: un *frontmatter* YAML roto carga la *skill* **sin metadatos** —existe, pero el modelo nunca sabe cuándo usarla— y un `hooks.json` malformado impide cargar el plugin completo.

## Documentación oficial

- [Crear y distribuir un plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) — la referencia del `marketplace.json`: esquema completo, tipos de origen, resolución de versiones y `strictKnownMarketplaces`. Es la página a la que volver cuando la duda es «¿qué campos acepta esto?».
- [Crear plugins](https://code.claude.com/docs/en/plugins) — el lado del contenido: estructura de directorios, `plugin.json`, y cómo se añaden agentes, *hooks*, servidores MCP y LSP. Empieza por aquí si lo que te falta es el plugin, no el catálogo.
- [Descubrir e instalar plugins](https://code.claude.com/docs/en/discover-plugins) — el lado de quien consume: comandos, alcances de instalación y el catálogo oficial. La sección de seguridad es corta y conviene leerla entera.
- [Referencia de plugins](https://code.claude.com/docs/en/plugins-reference) — las especificaciones técnicas que las otras páginas resumen: sustitución de variables de entorno, caché y resolución de ficheros, dependencias entre plugins.

## Recursos didácticos

- [Marketplace de demostración de Anthropic](https://github.com/anthropics/claude-code/tree/main/plugins) — plugins de ejemplo en un repositorio real. Se añade con `/plugin marketplace add anthropics/claude-code` y, sobre todo, se lee: ver un `marketplace.json` que funciona enseña más que el esquema.
- [Catálogo de la comunidad](https://github.com/anthropics/claude-plugins-community/blob/main/.claude-plugin/marketplace.json) — un `marketplace.json` de verdad, con muchas entradas y todos los tipos de origen en uso. Útil como referencia de cómo se ancla cada cosa a escala.
- **El plugin `plugin-dev`** del marketplace oficial — un kit para crear plugins, instalable con `/plugin install plugin-dev@claude-plugins-official`. Que la herramienta para escribir plugins sea a su vez un plugin es la mejor demostración de para qué sirve el sistema.
- [Catálogo público de plugins](https://claude.com/plugins) — para navegar lo que ya existe antes de escribir algo. La pregunta más rentable antes de empezar un plugin sigue siendo si ya está hecho.

---

*En resumen: un marketplace es un `marketplace.json` en un repositorio que convierte tus personalizaciones del agente en una dependencia instalable, versionada y actualizable — con la contrapartida de que lo que distribuyes es código que se ejecutará con los permisos de quien lo instale.*
