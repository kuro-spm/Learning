# Secrets, variables y entornos

## ¿Qué es?

El sistema de configuración de GitHub Actions, con tres ámbitos anidados —organización, repositorio y entorno— donde se guardan valores cifrados (**secrets**) y valores en claro (**variables**), y donde los **entornos** añaden además reglas que deciden *si* un despliegue puede ejecutarse.

## ¿Por qué existe?

Un pipeline necesita credenciales: la clave SSH del servidor, el token del registro de imágenes, la contraseña de la base de datos. Esas credenciales no pueden estar en el repositorio, porque el repositorio se clona, se comparte y se filtra. Pero tampoco pueden estar solo en la cabeza de alguien, porque el pipeline se ejecuta sin nadie delante.

La solución de GitHub es guardarlas **fuera del repositorio**, cifradas, e inyectarlas en la máquina efímera que ejecuta el job justo cuando hace falta. Y como distintos repositorios comparten credenciales (el token del registro es el mismo para todos) mientras otras son específicas (la clave del servidor de producción de un proyecto concreto), los ámbitos están anidados.

> Si ya conoces la configuración por capas de una aplicación (`appsettings.json` sobrescrito por `appsettings.Production.json` y este por variables de entorno), el modelo es idéntico: lo más específico gana.

> Los fundamentos de por qué no se meten credenciales en el repositorio están en [Secrets y variables de entorno](../ci-cd/Secrets-y-Variables.md) de la colección de CI/CD. Esta ficha se centra en lo que aporta el nivel de organización.

## ¿Cuándo y para qué se usa?

Cada vez que un workflow necesita hablar con algo de fuera: un registro de imágenes, un servidor por SSH, un proveedor de cloud, una API de terceros. Y el nivel de organización, en concreto, cuando el mismo valor lo necesitan varios repositorios: mantenerlo en un solo sitio significa que rotarlo es una operación, no quince.

## Cómo funciona un secret

Cuatro propiedades que definen todo su comportamiento:

**Se cifra al escribirlo y ya no se puede leer.** GitHub cifra el valor con la clave pública del repositorio o de la organización (usando una *sealed box* de libsodium). Después de guardarlo, nadie —ni tú, ni un Owner, ni el soporte de GitHub— puede recuperar el valor. Solo sobrescribirlo o borrarlo. Si es el único sitio donde estaba esa clave privada, la has perdido.

**Se inyecta en tiempo de ejecución.** Existe como variable de entorno dentro del runner, durante ese job, y desaparece con la máquina.

**Se enmascara en los logs.** GitHub sustituye las apariciones del valor exacto por `***`. Pero solo del valor exacto: si el script lo transforma, el enmascarado no lo sigue.

```yaml
- name: Cómo se filtra un secret sin querer
  run: |
    echo "${{ secrets.API_TOKEN }}"              # ***  (enmascarado)
    echo "${{ secrets.API_TOKEN }}" | base64     # se imprime en claro: otra cadena
    echo "${{ secrets.API_TOKEN }}" | cut -c1-10 # se imprime el trozo: otra cadena
```

La segunda y la tercera línea publican la credencial en un log que puede leer cualquiera con acceso al repositorio. Es la fuga más común y no la detecta ninguna herramienta.

**No se exponen a pull requests desde forks.** Un `pull_request` que llega de un fork ejecuta el workflow **sin** secretos. Es deliberado: si no, cualquiera podría abrir un PR con un workflow modificado y exfiltrar tus credenciales. La consecuencia práctica es que los workflows de CI que solo compilan y pasan tests deben poder funcionar sin secretos.

## Secrets frente a variables

Se configuran en la misma pantalla y hay que elegir bien:

| | Secret | Variable |
|---|---|---|
| Se accede con | `${{ secrets.NOMBRE }}` | `${{ vars.NOMBRE }}` |
| Almacenamiento | Cifrado, no legible | Texto plano, visible en la interfaz |
| En los logs | Enmascarado | Se imprime tal cual |
| Para qué | Credenciales, claves, tokens | Configuración: URLs, interruptores, nombres de recurso |

La regla: si filtrarlo obliga a rotarlo, es un secret. Si no, es una variable. Y hay una razón positiva para usar variables donde toque, más allá de no abusar del cifrado: **se pueden leer**. Una variable `PROD_URL` mal escrita se detecta mirándola; un secret mal escrito solo se detecta cuando el despliegue falla con un error incomprensible.

## Los tres ámbitos y la precedencia

El mismo nombre puede existir en los tres niveles. Cuando eso pasa, **gana el más específico**:

```
entorno  >  repositorio  >  organización
```

Esto habilita el patrón más útil de todo el sistema: definir el valor por defecto en la organización y sobrescribirlo solo donde haga falta.

```yaml
jobs:
  desplegar:
    runs-on: ubuntu-latest
    environment: production        # ← esta línea decide qué valores llegan
    steps:
      - run: echo "Desplegando en ${{ vars.API_URL }}"
```

Si `API_URL` está definida en la organización como la URL de pruebas y en el entorno `production` como la de producción, ese job usa la de producción y cualquier otro job sin `environment:` usa la de la organización. El mismo YAML sirve para los dos casos.

### Secrets de organización y su alcance

Al crear un secret de organización eliges a qué repositorios llega: **todos**, **todos los privados e internos**, o **una lista concreta**. La opción por defecto ("todos") es cómoda y casi nunca correcta: significa que un repositorio nuevo, o el repositorio de experimentos de alguien, recibe automáticamente las credenciales de producción.

```bash
# Secret de organización limitado a repositorios concretos
gh secret set GHCR_TOKEN --org acme-store \
  --visibility selected \
  --repos api-pedidos,web-tienda < ~/claves/ghcr.txt
```

Nótese el `< fichero`: pasar el valor por *stdin* evita que quede en el historial del shell, que es un fichero de texto plano en el disco de tu máquina.

Una nota que despista: los secrets de **Dependabot** son un almacén distinto del de Actions. Si Dependabot necesita credenciales para acceder a un registro privado de paquetes, hay que darlas ahí; no hereda las de Actions.

## Entornos

Un entorno es una etiqueta con reglas. Se declara en el job con `environment:` y sirve para tres cosas: **aislar valores** (secrets y variables propios), **imponer condiciones** antes de ejecutar, y **registrar** el historial de despliegues.

Lo que de verdad lo distingue de un simple prefijo de nombre son las reglas de protección:

- **Required reviewers** — el job se **pausa** y espera aprobación humana explícita. Hasta seis personas o equipos; basta con que uno apruebe.
- **Wait timer** — retardo obligatorio antes de arrancar, que da margen a cancelar.
- **Deployment branches** — solo ciertas ramas o tags pueden desplegar en este entorno. Es lo que impide que alguien despliegue producción desde una rama de trabajo.

Esa primera regla es la más valiosa: convierte una revisión que era una convención documentada en una puerta que el sistema no deja pasar.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Construyendo la imagen..."

  desplegar-produccion:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
      url: ${{ vars.PROD_URL }}     # aparece como enlace en la vista de despliegues
    steps:
      - name: Deploy por SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}      # secret DEL ENTORNO, no del repositorio
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /opt/tienda
            TAG=${{ github.ref_name }} docker compose pull
            TAG=${{ github.ref_name }} docker compose up -d
```

Con *required reviewers* configurado en `production`, la ejecución llega hasta `desplegar-produccion`, se detiene, y GitHub notifica a quien tenga que aprobar. En la interfaz aparece un botón *Review deployments*. Nada toca el servidor hasta que alguien pulsa.

Las reglas de protección de entorno en repositorios privados suelen requerir un plan de pago; en repositorios públicos están disponibles sin coste.

## Activar el despliegue con una variable

Un patrón que merece conocerse, porque resuelve el problema de tener el pipeline de despliegue escrito antes de tener el servidor: condicionar el job a una variable en lugar de comentar código.

```yaml
jobs:
  desplegar:
    if: vars.DEPLOY_ENABLED == 'true'      # la variable no existe → no se ejecuta
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: echo "Desplegando de verdad"

  aviso-dry-run:
    if: vars.DEPLOY_ENABLED != 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "::notice::Simulación: imagen publicada, despliegue omitido."
```

Mientras `DEPLOY_ENABLED` no exista, cada release publica la imagen y avisa de que el despliegue se omite. El día que el servidor esté listo, se crea la variable y **no se toca ni una línea de YAML**. El pipeline ya estaba probado; lo único que cambiaba era el interruptor.

## OIDC: la alternativa a guardar credenciales

El mejor secret es el que no existe. Con **OpenID Connect**, el workflow obtiene un token de corta duración directamente del proveedor de cloud, que confía en GitHub como emisor de identidad. No hay clave guardada que rotar ni que filtrar.

```yaml
permissions:
  id-token: write        # imprescindible: autoriza a pedir el token OIDC
  contents: read

steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}    # identificadores, no secretos
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

Ni una credencial en `secrets`: solo identificadores públicos en `vars`. La confianza se configura una vez en el proveedor de cloud, restringida a una organización, un repositorio y opcionalmente un entorno concretos. AWS, Azure y Google Cloud lo soportan. Si tu despliegue va a un cloud, esto es lo correcto; los secrets de larga duración quedan para lo que no lo soporta, como un servidor propio con SSH.

## Buenas prácticas avanzadas

- **Nunca transformes un secret en un comando que escriba en el log.** El enmascarado solo cubre el valor exacto: `base64`, `cut`, `jq`, un `curl -v` que imprime cabeceras, o un mensaje de error de una librería que incluye la URL con el token. Todos publican la credencial en claro. La regla operativa: los secrets se pasan como variables de entorno a un programa, nunca se interpolan en `echo` ni en cadenas que se puedan imprimir.
- **Limita el alcance de los secrets de organización a repositorios seleccionados.** Con "todos los repositorios", el repositorio de pruebas que alguien cree mañana nace con acceso a las credenciales de producción. Y como cualquiera con permiso de escritura puede añadir un workflow, ese acceso es explotable en un commit.
- **Guarda el valor original en un gestor de contraseñas en el mismo momento en que creas el secret.** GitHub es un almacén de solo escritura: no es tu copia de seguridad. Si la clave privada de despliegue solo está en un secret de GitHub, no la tienes.
- **Usa el `environment:` incluso cuando no necesites secretos distintos**, solo por los *required reviewers* y el registro de despliegues. Es la forma de convertir "acordamos que producción se revisa" en algo que el sistema impone y que además deja constancia de quién aprobó qué y cuándo.
- **Migra a OIDC todo lo que lo soporte.** Una clave de larga duración es una responsabilidad permanente: hay que rotarla, hay que saber quién la ha visto y hay que asumir que algún día se filtrará. Un token de quince minutos emitido bajo demanda elimina las tres preocupaciones de golpe.
- **Fija las acciones de terceros por SHA, no por etiqueta.** `uses: alguien/accion@v1` apunta a una etiqueta que su autor puede mover a cualquier commit; ese código se ejecuta en el mismo job que tus secretos. `uses: alguien/accion@a1b2c3d...` apunta a un commit inmutable. Dependabot sabe actualizar esos SHA por pull request, así que no pierdes las actualizaciones.

## Recursos didácticos

- [Usar secrets en GitHub Actions](https://docs.github.com/actions/security-guides/using-secrets-in-github-actions) — incluye la lista de límites (tamaño máximo, número de secrets) que sorprenden cuando se topa con ellos.
- [Usar entornos para el despliegue](https://docs.github.com/actions/deployment/targeting-different-environments/using-environments-for-deployment) — reglas de protección explicadas una a una.
- [Endurecer la seguridad de Actions](https://docs.github.com/actions/security-guides/security-hardening-for-github-actions) — la lectura obligatoria: explica los vectores reales de fuga de secretos en un pipeline, con ejemplos.

---

*En resumen: tres ámbitos donde el más específico gana, un almacén que solo se puede escribir, y entornos que convierten "hay que revisarlo" en un botón que alguien tiene que pulsar.*
