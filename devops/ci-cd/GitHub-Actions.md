# GitHub Actions

## ¿Qué es?

GitHub Actions es la herramienta de CI/CD integrada en GitHub. Permite definir pipelines mediante archivos YAML que viven en tu repositorio y se ejecutan automáticamente cuando ocurre algún evento (un `push`, una pull request, etc.).

## ¿Por qué existe?

Si tu código ya está en GitHub, tener que conectar una herramienta de CI/CD externa añade fricción: cuentas separadas, permisos, configuración. GitHub Actions resuelve eso poniendo la automatización **dentro del propio GitHub**: defines el pipeline en una carpeta del repo y funciona, sin instalar ni configurar servidores aparte.

> Si vienes de otros sistemas de CI: piensa en GitHub Actions como "un pipeline que ya vive donde está tu código, sin tener que enchufar nada externo".

## ¿Cuándo y para qué se usa?

En cualquier proyecto alojado en GitHub que quiera automatizar tareas:

- Ejecutar los tests de una **app de tareas** en cada pull request, para no fusionar código roto.
- Compilar y publicar una **tienda online** automáticamente al subir a `main`.
- Regenerar y desplegar un **blog** estático cada vez que se actualiza un artículo.
- Tareas que no son despliegue: etiquetar issues, enviar avisos, ejecutar scripts a una hora fija.

## Lo mínimo que necesitas saber

**1. Dónde viven los workflows**

Los pipelines (aquí llamados *workflows*) son archivos `.yml` dentro de `.github/workflows/`. Puedes tener varios.

```
.github/
└── workflows/
    └── ci.yml
```

**2. Disparadores (`on`)**

Defines qué evento lanza el workflow.

```yaml
on:
  push:
    branches: [main]
  pull_request:        # también en cada pull request
```

**3. Jobs y runners**

Cada *job* se ejecuta en una máquina virtual limpia (un *runner*) que tú eliges (Linux, Windows o macOS).

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4    # descarga tu código
      - run: npm install
      - run: npm test
```

**4. Actions reutilizables (`uses`)**

Una *action* es un paso empaquetado que alguien ya escribió, para no reinventar la rueda. Se usan con `uses:`. Por ejemplo `actions/checkout` descarga tu repositorio y `actions/setup-node` instala Node.js.

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
```

**5. Ejemplo completo mínimo**

```yaml
name: CI
on:
  push:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm install
      - run: npm test
```

**6. Secrets**

Las contraseñas y tokens nunca se escriben en el YAML. Se guardan en los *Secrets* del repositorio y se leen con `${{ secrets.NOMBRE }}`.

```yaml
- run: ./deploy.sh
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
```

## Lo que NO hace

- **No es exclusivo de despliegue** — sirve para cualquier automatización, no solo CI/CD.
- **No funciona fuera de GitHub** — está atado a repositorios de GitHub.
- **No es ilimitado gratis** — los minutos de ejecución tienen una cuota según el plan (los repos públicos suelen tener mucho margen).
- **No adivina tu proceso** — tú escribes cada paso del workflow.

## Buenas prácticas avanzadas

- **Ancla las actions de terceros a un commit SHA, no a un tag** — `actions/checkout@v4` parece fijo, pero un tag de Git se puede mover: si comprometen la cuenta del autor, `v4` puede pasar a apuntar a código malicioso que corre con acceso a tus secrets. Anclar al hash completo (`uses: actions/checkout@8f4b7f8...`) hace el paso inmutable. Es el ataque de cadena de suministro clásico en Actions, y la mayoría no se protege.
- **Recorta los permisos del `GITHUB_TOKEN`** — cada workflow recibe un token automático que, según la configuración del repo, puede tener permiso de escritura sobre casi todo. Declara arriba del workflow el mínimo necesario (`permissions: { contents: read }`) y amplía solo en el job que lo necesite: si un paso comprometido roba el token, el daño queda acotado.
- **Cancela ejecuciones obsoletas con `concurrency`** — sin esto, cada push a una pull request encola un workflow nuevo mientras los anteriores siguen corriendo, quemando minutos en validar código que ya nadie va a fusionar.

  ```yaml
  concurrency:
    group: ci-${{ github.ref }}
    cancel-in-progress: true
  ```
- **Trata `pull_request_target` como material radiactivo** — a diferencia de `pull_request`, este disparador ejecuta con acceso a los secrets del repositorio... sobre eventos que puede provocar cualquiera desde un fork. Si además haces checkout del código del fork, un desconocido puede exfiltrar tus secrets con solo abrir una PR. Úsalo solo si entiendes exactamente por qué lo necesitas, y nunca ejecutes código del fork dentro de él.
- **No copies YAML entre repositorios: usa `workflow_call`** — cuando el mismo pipeline se repite en varios repos, conviértelo en un *reusable workflow* que los demás invocan con `uses:`. Los workflows clonados divergen en silencio y arreglar un fallo pasa a ser una cacería por N repositorios.

---

*En resumen: GitHub Actions es CI/CD que ya vive dentro de tu repositorio de GitHub — defines workflows en YAML y se ejecutan solos ante cada cambio.*
