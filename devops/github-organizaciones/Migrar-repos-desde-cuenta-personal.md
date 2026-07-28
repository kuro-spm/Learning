# Migrar repositorios desde una cuenta personal

## ¿Qué es?

La transferencia de un repositorio que vive en una cuenta personal a una organización, conservando su historial, sus issues, sus pull requests y sus releases, y dejando redirecciones desde las URLs antiguas.

## ¿Por qué existe?

Muchos proyectos empiezan como un `git init` en la carpeta de alguien. Cuando ese proyecto crece, se factura a un cliente o pasa a tener más de una persona detrás, la propiedad tiene que moverse. GitHub ofrece una **transferencia** nativa precisamente para que eso no implique perder el historial de conversación del proyecto, que es tan valioso como el código: las decisiones discutidas en un pull request de hace ocho meses, el issue donde se documentó un bug raro, las releases publicadas.

La alternativa casera —crear un repositorio nuevo en la organización y empujar el código— conserva los commits y pierde todo lo demás. Es lo que hay que evitar.

> Si ya conoces las bases de datos, la diferencia es la que hay entre un `RESTORE` completo y un `INSERT INTO ... SELECT` de la tabla principal: en el segundo caso te llevas las filas y te dejas las claves foráneas.

## ¿Cuándo y para qué se usa?

Al formalizar un proyecto: pasa de personal a profesional, entra un segundo desarrollador, o se descubre que el código de un cliente está en la cuenta de GitHub de alguien. También al consolidar: mover a una organización repositorios repartidos entre varias cuentas personales del equipo.

## Qué se lleva la transferencia y qué no

Esto es lo que hay que saber antes de pulsar el botón. **Se transfiere**:

- Todo el historial de git: commits, ramas, tags.
- Issues y pull requests, con sus comentarios y su numeración intacta.
- Wiki, releases (con sus binarios adjuntos), stars y watchers.
- Los webhooks y los colaboradores directos del repositorio.
- **Redirecciones**: las URLs y los remotos de git antiguos siguen funcionando.

Lo que hay que dar por **no transferido** y rehacer a mano:

- **Los secrets y las variables de Actions.** Y aquí está el punto crítico: los secrets **no se pueden leer**, ni antes ni después. Si no tienes los valores originales guardados en otro sitio (un gestor de contraseñas), migrar el repositorio significa perderlos para siempre. Apunta la **lista de nombres** antes de transferir y asegúrate de tener los valores localizados.
- **Los paquetes publicados.** Una imagen en `ghcr.io/ana-dev/api-pedidos` no se mueve a `ghcr.io/acme-store/api-pedidos`. Las etiquetas ya publicadas se quedan en el namespace viejo y hay que republicar o migrar los paquetes por separado.
- **Las reglas de rama y los rulesets**, que conviene revisar y rehacer: aunque algo sobreviva, las condiciones de plan cambian (lo que en una cuenta personal Free no existía, en la organización Team sí).
- **Los environments** y sus reglas de protección.
- **Las GitHub Pages** con dominio personalizado, que necesitan reconfigurar el DNS.

## Requisitos previos

Dos, y ambos bloquean:

1. Ser **propietaria** del repositorio de origen (no basta rol Admin como colaborador).
2. Tener **permiso para crear repositorios** en la organización de destino. Si el paso 3 de la configuración inicial restringió la creación a los Owners, quien transfiere tiene que ser Owner o pedir que se le conceda temporalmente.

Si el nombre ya existe en la organización, la transferencia falla; hay que renombrar antes uno de los dos.

## El procedimiento

### 1. Inventario antes de tocar nada

Apunta lo que vas a tener que rehacer:

```bash
# Nombres de los secrets y variables (los valores no son legibles: por eso solo hay nombres)
gh secret list   --repo ana-dev/api-pedidos
gh variable list --repo ana-dev/api-pedidos

# Paquetes publicados bajo la cuenta personal
gh api "/users/ana-dev/packages?package_type=container" --jq ".[].name"

# Quién colabora hoy, para reconstruir el acceso vía teams después
gh api /repos/ana-dev/api-pedidos/collaborators --jq ".[].login"
```

Guarda esa salida. Es tu lista de tareas del paso 4.

### 2. Transferir

Por interfaz web: *Settings → General → Danger Zone → Transfer ownership*, escribir el nombre de la organización de destino y confirmar el nombre del repositorio. GitHub pide confirmación explícita porque, aunque es reversible (puedes volver a transferir), no es instantáneo de deshacer.

Desde la CLI:

```bash
gh api -X POST /repos/ana-dev/api-pedidos/transfer -f new_owner=acme-store
```

Si la organización tiene la transferencia sujeta a aprobación, el repositorio queda pendiente hasta que un Owner la acepta.

### 3. Actualizar los clones locales

La redirección hace que el push siga funcionando desde los clones antiguos, pero conviene no confiar en ella: se rompe el día que alguien registre un repositorio con el nombre viejo en la cuenta personal. En cada máquina:

```bash
git remote set-url origin git@github.com:acme-store/api-pedidos.git
git remote -v
# origin  git@github.com:acme-store/api-pedidos.git (fetch)
# origin  git@github.com:acme-store/api-pedidos.git (push)
```

Si el equipo es grande, un mensaje en el canal común con esa línea exacta ahorra media docena de preguntas.

### 4. Rehacer secrets, variables y accesos

Con la lista del paso 1:

```bash
# Secrets: desde fichero o por stdin, nunca como argumento
# (un argumento en la línea de comandos queda en el historial del shell)
gh secret set SSH_KEY --repo acme-store/api-pedidos < ~/claves/deploy_key
gh variable set DEPLOY_ENABLED --repo acme-store/api-pedidos --body true
```

Y el acceso: este es el momento de dejar de conceder permisos persona a persona y pasarlos a equipos, que es la razón de haber migrado.

```bash
# En lugar de añadir colaboradores uno a uno
gh api -X PUT /orgs/acme-store/teams/backend/repos/acme-store/api-pedidos -f permission=push
```

### 5. Revisar los workflows

Busca cualquier referencia al propietario antiguo escrita a mano:

```bash
grep -rn "ana-dev" .github/workflows/
```

Si el resultado no está vacío, sustitúyelo por el contexto dinámico, que sobrevive a cualquier transferencia futura:

```yaml
# Mal: se rompe en cuanto el repositorio cambia de dueño
tags: ghcr.io/ana-dev/api-pedidos:${{ github.ref_name }}

# Bien: se resuelve solo
tags: ghcr.io/${{ github.repository_owner }}/api-pedidos:${{ github.ref_name }}
```

Comprueba también que el `permissions:` de cada workflow sigue siendo suficiente: si la organización dejó `GITHUB_TOKEN` en read-only por defecto y el workflow publicaba paquetes sin declarar `packages: write`, empezará a fallar con un `403` justo después de la migración. Es la incidencia más frecuente tras una transferencia y suele diagnosticarse mal, porque el YAML no ha cambiado.

### 6. Validar

No des la migración por buena hasta comprobar cuatro cosas:

- [ ] Un clon nuevo desde la URL nueva compila y pasa los tests.
- [ ] Un pull request de prueba dispara el CI y este pasa en verde.
- [ ] El workflow de release funciona (si es peligroso, con un tag de prueba y el despliegue desactivado).
- [ ] La lista de secrets y variables en el destino coincide con el inventario del paso 1.

## Alternativas y cuándo usarlas

La transferencia no siempre es la herramienta correcta:

- **Fork a la organización.** Crea una copia enlazada, no mueve la propiedad. Sirve para experimentar sin comprometerse, pero deja dos repositorios vivos y la ambigüedad de cuál es el bueno. Rara vez es lo que quieres.
- **Importar (`gh repo create --source`) o empujar a un repositorio nuevo.** Te llevas el código y pierdes issues y pull requests. Solo tiene sentido si el historial de conversación no vale nada, o si quieres cortar deliberadamente con el pasado (por ejemplo, para dejar atrás un historial que contuvo secretos).
- **Migrar solo el código porque el historial arrastra un secreto filtrado.** Aquí la respuesta correcta no es empezar de cero: es rotar el secreto (que es lo urgente, porque el valor filtrado sigue siendo válido esté donde esté el repositorio) y, si de verdad hace falta, limpiar el historial con `git filter-repo`. Empezar un repositorio nuevo sin rotar la credencial no arregla nada.

## Buenas prácticas avanzadas

- **Localiza los valores de los secrets antes de transferir, no después.** Es el único paso verdaderamente irreversible de toda la migración: un secret que nadie guardó en un gestor de contraseñas y que no se puede leer es un secret perdido. Si es la clave privada de despliegue, has perdido el acceso al servidor.
- **Aprovecha la migración para pasar de colaboradores a equipos.** El repositorio llega con una lista de personas con acceso directo. Reconstruir ese acceso vía teams cuesta unos minutos justo ahora, mientras el asunto está fresco, y es lo que hace que la organización sirva de algo. Si no, en un año tendrás una organización con la misma gestión persona-a-persona que tenías antes.
- **Actualiza los remotos aunque la redirección funcione.** La redirección es un puente de cortesía que se cae en silencio si alguien crea un repositorio con el nombre viejo en la cuenta de origen. El fallo aparece semanas después, en la máquina de otra persona, y nadie lo relaciona con la migración.
- **Espera el `403` en el primer release posterior a la migración.** Si la organización endureció los permisos por defecto del `GITHUB_TOKEN`, los workflows que publicaban paquetes o creaban releases sin declarar `permissions:` empezarán a fallar. Añadir el bloque `permissions` explícito en la misma tanda de la migración evita el susto.
- **Migra un repositorio pequeño primero.** Antes de mover el proyecto crítico, transfiere uno secundario y recorre el checklist completo. Descubrirás las particularidades de tu caso (paquetes, Pages, integraciones externas con webhooks) sobre algo que no pasa nada si se rompe una tarde.

## Recursos didácticos

- [Transferir un repositorio](https://docs.github.com/repositories/creating-and-managing-repositories/transferring-a-repository) — la lista oficial y actualizada de qué se transfiere y qué no; consúltala justo antes de migrar, porque cambia.
- [`git filter-repo`](https://github.com/newren/git-filter-repo) — la herramienta recomendada hoy para reescribir historial (sustituye a `git filter-branch`), si de verdad hay que limpiar algo del pasado.
- [`gh repo` en el manual de la CLI](https://cli.github.com/manual/gh_repo) — todos los subcomandos de gestión de repositorios, útiles para automatizar una migración de varios repositorios seguidos.

---

*En resumen: la transferencia conserva el código y las conversaciones; los secretos, los paquetes y los remotos son tuyos y los rehaces a mano.*
