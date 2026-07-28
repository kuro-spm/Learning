# Por qué los secretos no van a git

## ¿Qué es?

Un **secreto** es cualquier dato que otorga acceso o capacidad: una clave de API, la contraseña de una base de datos, una clave de firma de tokens, una cadena de conexión con credenciales dentro. Esta ficha trata de una única regla y de sus consecuencias prácticas: **ese dato no entra nunca en el control de versiones**.

## ¿Por qué existe?

Porque git no está diseñado para olvidar. Su trabajo es exactamente el contrario: conservar cada versión de cada fichero para siempre y replicarla en cada copia del repositorio. Eso es una virtud para el código y un desastre para una contraseña, porque un secreto añadido "solo un momento" queda grabado en un objeto inmutable que viaja a todos los clones, forks, backups y cachés de CI que existan o vayan a existir.

> Si ya conoces los *append-only logs*, piensa en git como en uno de ellos. Un `git rm` no borra: añade un commit que dice "aquí ya no está". El contenido anterior sigue almacenado y sigue siendo recuperable con su SHA. Es como tachar un dato con rotulador en una fotocopia: lo tapas en tu hoja, pero el original está archivado y hay cien copias repartidas.

> Regla rápida: si con ese dato alguien puede **gastar dinero, leer datos o suplantarte**, es un secreto.

## ¿Cuándo y para qué se usa?

Siempre, desde el primer commit de cualquier repositorio. No es una práctica que se adopte "cuando el proyecto se ponga serio", porque el coste de aplicarla el primer día son cinco minutos y el coste de aplicarla el día 400 es reescribir el historial y rotar credenciales en producción.

El ejemplo que recorre la ficha es una tienda online con la API en `src/Tienda.Api` y dos secretos reales: la clave de la pasarela de pagos (`Pasarela:ApiKey`, contra `api.pasarela.ejemplo.com`) y la cadena de conexión de la base de datos (`ConnectionStrings:TiendaDb`).

---

## Secreto frente a configuración

La regla es fácil de enunciar y difícil de aplicar, porque el 90 % del trabajo consiste en decidir de qué lado cae cada valor. La distinción no es "¿es un dato técnico?" sino: **¿este valor identifica *qué* usas, o demuestra *quién* eres?**

| Valor | Veredicto | Por qué |
|---|---|---|
| `https://api.pasarela.ejemplo.com` (URL del endpoint) | Configuración | Es público: cualquiera puede leerlo en la documentación del proveedor |
| `Production`, `Staging` (nombre del entorno) | Configuración | Describe dónde corre el proceso, no autoriza nada |
| `5432`, `8080` (número de puerto) | Configuración | Saberlo no da acceso; el acceso lo controla el firewall |
| `ClientId` de OAuth | Configuración | Está pensado para viajar en la URL de autorización, a la vista del navegador |
| `ClientSecret` de OAuth | **Secreto** | Es la mitad que prueba que eres tu aplicación y no otra |
| `Server=db;Database=TiendaDb;User Id=api;Password=...` | **Secreto** | Lleva la contraseña dentro: el conjunto es tan sensible como su parte más sensible |
| ID de un bucket de almacenamiento | Configuración | Es un nombre; el acceso lo decide la política del bucket |
| Clave de firma de JWT (HMAC) | **Secreto** | Con ella se fabrican tokens válidos para cualquier usuario, incluido un administrador |
| Clave **pública** de un par asimétrico | Configuración | Existe para repartirla; sirve para verificar, no para firmar |
| Clave **privada** de un par asimétrico | **Secreto** | Es la que firma y la que descifra |

Dos observaciones que se sacan de esta tabla y que valen más que la tabla:

- **La sensibilidad de un valor compuesto es la de su componente más sensible.** Una cadena de conexión es configuración hasta que le metes la contraseña; a partir de ahí es un secreto entero, no "casi todo configuración".
- **Ante la duda, secreto.** El coste de tratar como secreto algo que no lo era es una línea más en el `.gitignore` y una variable más que rellenar. El coste del error contrario es rotar credenciales de producción con el equipo mirando.

## Por qué git es una trampa

Aquí es donde la intuición falla. Vamos a demostrarlo con comandos sobre un caso concreto: alguien comiteó `appsettings.json` con la clave de la pasarela dentro, se dio cuenta a los diez minutos y la borró en el commit siguiente. Para esa persona el problema está resuelto. No lo está.

**1. El historial es para siempre.** Este comando muestra todos los cambios que ha sufrido el fichero, versión a versión, y la salida contiene literalmente la clave que "ya se borró":

```bash
git log -p --follow -- src/Tienda.Api/appsettings.json
```

```diff
commit 4b1c9de7a2f05c8e1d3b6a9f0c7e2d5a8b4f1c3e
    Configura la pasarela de pagos

+  "Pasarela": {
+    "ApiKey": "sk_live_9f2a41c8e7b0d5a3"
+  },
```

Ese `+` sigue ahí en cada clon del repositorio. Y no hace falta ni leer el diff: si conoces el SHA, pides el fichero completo tal y como estaba en ese momento.

```bash
git show 4b1c9de:src/Tienda.Api/appsettings.json
```

```json
{
  "ConnectionStrings": {
    "TiendaDb": "Server=db;Database=TiendaDb;User Id=api;Password=Verano2026!"
  },
  "Pasarela": { "ApiKey": "sk_live_9f2a41c8e7b0d5a3" }
}
```

Peor aún: no hace falta saber ni el SHA ni el fichero. Esto rastrea **todo** el historial de **todas** las ramas buscando un prefijo de clave:

```bash
git rev-list --all | xargs git grep -n 'sk_live_'
```

```
4b1c9de:src/Tienda.Api/appsettings.json:6:    "ApiKey": "sk_live_9f2a41c8e7b0d5a3"
a70f2b1:src/Tienda.Api/appsettings.json:6:    "ApiKey": "sk_live_9f2a41c8e7b0d5a3"
c93e5d4:tests/Tienda.Api.Tests/appsettings.Test.json:4:    "ApiKey": "sk_live_9f2a41c8e7b0d5a3"
```

Tres commits, dos ficheros, y el tercero es una copia en los tests que nadie recordaba. Este comando tarda segundos en un repositorio mediano y no requiere ningún permiso especial: lo ejecuta quien tenga el repo. Eso incluye a un atacante que consiguió un clon.

**2. El repositorio se copia mucho más de lo que parece.** Un secreto en git no está "en un sitio", está a la vez en los clones locales de todo el equipo (incluida la persona que se fue y no borró la carpeta), en los forks —que son copias independientes fuera de tu control administrativo—, en el *workspace* y la caché de cada ejecución de CI, en los backups del servidor de git y en los índices de búsqueda de código de la plataforma. Borrar el secreto del repositorio original no toca ninguna de esas copias.

**3. "Privado hoy" no es "privado siempre".** Un repositorio privado protege mientras se cumplan a la vez varias condiciones: que nadie lo pase a público al reorganizar la organización, que los permisos de equipos y colaboradores externos estén bien, que ninguna cuenta con acceso se vea comprometida, que la plataforma no sufra una brecha y que el acceso amplio dado a un contratista "solo para esta semana" se retire de verdad. Meter un secreto en git es apostar a que las cinco se cumplen durante toda la vida del proyecto.

## El coste real de una fuga

Hay proyectos de investigación que suben claves señuelo a repositorios públicos y miden cuánto tardan en recibir el primer intento de uso. El orden de magnitud que reportan no son días ni horas: son **minutos**, a veces menos de uno. No es sorprendente, porque la API de eventos públicos de GitHub permite reaccionar a cada push casi en tiempo real y los prefijos que identifican las claves de los proveedores grandes son documentación pública.

Lo que pasa después depende de qué abría la clave. **Un servicio de pago por uso** (LLMs, SMS, cómputo en la nube) se convierte en factura: consumo automatizado y en paralelo hasta el límite de gasto configurado, y el detalle que hace daño es que si nunca fijaste un límite, el techo lo pone el proveedor y está muy por encima de lo que habrías autorizado. **La base de datos** se convierte en exfiltración o en extorsión: hay campañas que copian la base, la borran y dejan una tabla con instrucciones de pago. Y **la clave de firma de JWT** es la peor y la que menos alarma provoca, porque no hay factura — con ella se fabrican tokens válidos con cualquier `sub` y cualquier rol, y no hay nada anómalo que detectar en los logs: las peticiones llegan perfectamente autenticadas.

La conclusión operativa es incómoda: **el tiempo de reacción disponible es menor que el tiempo que tardas en darte cuenta.** Por eso "lo borré en el commit siguiente" no es una mitigación, y por eso lo primero del procedimiento de respuesta es rotar.

## La idea de fondo: dominios de confianza separados

Todo lo anterior se resume en un principio que permite decidir bien en los casos que ninguna tabla cubre. El código y los secretos pertenecen a **dominios de confianza distintos**:

| | Código | Secreto |
|---|---|---|
| Dónde vive | En git | Fuera de git |
| Quién lo lee | Todo el equipo, CI, forks, futuros contratados | Solo el proceso que lo necesita |
| Cuántas copias hay | Docenas, incontroladas | Una por entorno, controlada |
| Cómo se cambia | Un commit y una revisión | Un cambio en el almacén y un reinicio |

El objetivo no es esconder el secreto dentro del código —ofuscarlo, codificarlo en base64, partirlo en dos constantes— porque eso no cambia de dominio nada. El objetivo es **sacarlo del código**, de modo que quien lee el repositorio no obtenga el secreto y quien roba una copia del repositorio tampoco.

Cómo se materializa depende del entorno: en desarrollo, con [User-secrets en .NET](User-Secrets-En-Dotnet.md), que guarda el fichero en tu perfil de usuario, fuera del árbol del proyecto; en producción, con variables de entorno inyectadas por la plataforma o con un gestor de secretos. Si necesitas guardar credenciales cifradas en tu propio almacenamiento, eso lo cubre [Cifrado en reposo de credenciales](Cifrado-En-Reposo-De-Credenciales.md). Y ten presente que sacar el secreto de git no basta si luego se filtra en el momento de usarlo, que es el tema de [Credenciales en llamadas salientes](../secretos-en-llamadas-salientes/Credenciales-en-Llamadas-Salientes.md).

## Qué SÍ va al repositorio

La regla no es "el repositorio no sabe nada de los secretos". Es **el repositorio conoce la forma, no el contenido**: quien clona el proyecto tiene que poder saber qué valores hace falta rellenar sin preguntar a nadie. Así queda `src/Tienda.Api/appsettings.json` en las dos versiones:

```json
// ❌ los valores reales, comiteados
{
  "ConnectionStrings": { "TiendaDb": "Server=db;Database=TiendaDb;User Id=api;Password=Verano2026!" },
  "Pasarela": { "BaseUrl": "https://api.pasarela.ejemplo.com", "ApiKey": "sk_live_9f2a41c8e7b0d5a3" }
}
```

```json
// ✅ la estructura declarada, los secretos vacíos
{
  "ConnectionStrings": { "TiendaDb": "" },
  "Pasarela": { "BaseUrl": "https://api.pasarela.ejemplo.com", "ApiKey": "" }
}
```

Fíjate en que `BaseUrl` **sí** se queda con su valor: es configuración. Y en que las claves siguen declaradas aunque estén vacías: eso documenta el contrato de configuración y hace que la aplicación falle con un error claro ("`Pasarela:ApiKey` está vacío") en lugar de con un `NullReferenceException` a las tres pantallas de profundidad.

Si el proyecto usa variables de entorno, el equivalente es un `.env.example` versionado —con los nombres, sin los valores— junto a un `.env` **no** versionado:

```dotenv
# .env.example — este fichero SÍ va a git
TIENDA_CONNECTIONSTRINGS__TIENDADB=
TIENDA_PASARELA__BASEURL=https://api.pasarela.ejemplo.com
TIENDA_PASARELA__APIKEY=
```

Y en el README va la parte que casi siempre falta: **de dónde se saca cada valor**.

```markdown
Copia `.env.example` a `.env` y rellena:
- `TIENDA_CONNECTIONSTRINGS__TIENDADB` — cadena de la base local (ver `docker-compose.yml`).
- `TIENDA_PASARELA__APIKEY` — panel de la pasarela → Claves de API → entorno *sandbox*.
```

Sin esas dos líneas, la persona nueva pasa media mañana adivinando y acaba pidiendo por chat una clave de producción, que es exactamente el incidente que intentamos evitar.

## Red de seguridad, capa por capa

La regla la aplican personas, y las personas se despistan. Estas son las dos capas que convierten el despiste en un aviso en lugar de en una fuga.

### Capa 1: `.gitignore`

Lo importante es ignorar los ficheros que de verdad acaban conteniendo secretos, no llenarlo de patrones genéricos:

```gitignore
# Secretos locales — nunca versionados
.env
appsettings.*.local.json
secrets.json
# Material criptográfico
*.pfx
*.pem
*.key
```

Ahora el aviso que casi nadie tiene interiorizado: **`.gitignore` no protege lo que git ya está siguiendo.** El fichero solo filtra candidatos nuevos; en cuanto un fichero está en el índice, git lo rastrea igual y `.gitignore` es papel mojado para él. Se comprueba y se corrige así:

```bash
git ls-files --error-unmatch .env
# .env        ← está en el índice: se seguirá comiteando pase lo que pase

git rm --cached .env          # lo saca del índice sin borrarlo de tu disco
git commit -m "Deja de versionar .env"
# [master 8e2a1f4] Deja de versionar .env
#  1 file changed, 0 insertions(+), 3 deletions(-)
#  delete mode 100644 .env
```

Y aquí viene la parte que suele malinterpretarse: ese commit **no elimina el secreto del historial**, solo deja de arrastrarlo hacia el futuro. Si el `.env` con valores reales estuvo versionado aunque sea un commit, sigues necesitando el procedimiento de la sección siguiente.

### Capa 2: escaneo de secretos

Un escáner busca patrones de credencial en el árbol de trabajo y en el historial. La herramienta habitual es **gitleaks**, y lo interesante es que revisa todos los commits, no solo el estado actual:

```bash
gitleaks detect --source . --redact --verbose
```

```
Finding:     "ApiKey": "REDACTED"
RuleID:      generic-api-key
Entropy:     4.184884
File:        src/Tienda.Api/appsettings.json
Line:        6
Commit:      4b1c9de7a2f05c8e1d3b6a9f0c7e2d5a8b4f1c3e
Author:      Ana Ruiz
Date:        2026-03-11T09:14:22Z

11:02AM INF 342 commits scanned.
11:02AM WRN leaks found: 1
```

El campo que hay que mirar es `Commit`: dice que el hallazgo está en el historial, no en tu copia actual. `--redact` evita que la clave acabe en el log de CI, que sería duplicar la fuga en otro sitio. El código de salida es distinto de cero cuando encuentra algo, así que sirve directamente como paso de CI que rompe la construcción.

Escanear el historial está bien para auditar; lo que evita el problema es escanear **antes** del commit. `gitleaks protect --staged` mira solo lo que hay en el área de preparación, y eso encaja en un hook:

```bash
# .git/hooks/pre-commit  (chmod +x)
#!/bin/sh
gitleaks protect --staged --redact --verbose || {
  echo "❌ Commit abortado: posible secreto detectado."
  exit 1
}
```

Como `.git/hooks/` no se versiona, en equipo esto se distribuye con `pre-commit` (el gestor de hooks) o con Husky, para que se instale solo al clonar en lugar de depender de que cada persona copie el fichero.

Vale la pena conocer las otras dos piezas del mismo tablero. **trufflehog** hace algo que gitleaks no: además de reconocer el patrón, **verifica** la credencial llamando al proveedor, con lo que distingue "esto parece una clave" de "esto es una clave viva ahora mismo" — decisivo cuando auditas un repositorio viejo lleno de valores de ejemplo. Y el **secret scanning de GitHub** analiza los repositorios por su cuenta con patrones que registran los propios proveedores, y avisa tanto a ti como al proveedor, que puede revocar la clave directamente. Su modo más útil es la **push protection**, que rechaza el push en el servidor:

```
remote: error: GH013: Repository rule violations found for refs/heads/master.
remote: - GITHUB PUSH PROTECTION
remote:   Resolve the following secret leak before pushing again.
remote:   —— Generic API Key ——
remote:      - commit: 4b1c9de7a2f05c8e1d3b6a9f0c7e2d5a8b4f1c3e
remote:        path: src/Tienda.Api/appsettings.json:6
```

Así el problema se detiene antes de existir en el servidor. Pero ninguna de estas capas es infalible: un secreto sin prefijo reconocible y con entropía baja (una contraseña como `Verano2026!`) pasa por delante de todos los escáneres. Son red de seguridad, no sustituto de la regla.

## Qué hacer cuando un secreto ya tocó git

Este es el procedimiento, y **el orden no es negociable**.

### 1. Rotar primero

Antes de tocar el historial, antes de avisar a nadie, antes de entender cómo pasó: ve al panel del proveedor, **genera una credencial nueva y revoca la antigua explícitamente**. Dos matices que suelen estropear este paso: crear la nueva sin revocar la vieja deja las dos activas y no arregla nada; y en el caso de la base de datos, "rotar" es cambiar la contraseña del usuario, no crear otro usuario y dejar el primero como estaba.

¿Por qué esto va antes que limpiar el historial? Por aritmética de tiempos. Si el push llegó al remoto, hubo una ventana en la que la clave fue legible por cualquiera con acceso, y esa ventana **ya se cerró en el pasado**: reescribir el historial no deshace una lectura que ya ocurrió. Limpiar sin rotar produce un repositorio limpio y una credencial comprometida en producción, que es el peor resultado posible porque además genera la sensación de haberlo resuelto. Rotar sin limpiar deja un repositorio sucio con una credencial ya inservible: feo, pero inofensivo. Dicho de otra forma, **rotar es la mitigación y limpiar el historial es la higiene** — si solo tienes tiempo para una, haz la primera.

### 2. Limpiar el historial

La herramienta recomendada es `git filter-repo`, que sustituye a la vieja `filter-branch` (mucho más lenta y con trampas de comportamiento). Si el secreto está dentro de un fichero que quieres conservar, interesa reemplazar el valor, no eliminar el fichero:

```bash
# 1. Copia de seguridad: filter-repo reescribe de forma destructiva
git clone --mirror git@github.com:ejemplo/tienda.git tienda-backup.git

# 2. Lista de literales a sustituir en todo el historial
cat > reemplazos.txt <<'EOF'
sk_live_9f2a41c8e7b0d5a3==>***ELIMINADO***
Verano2026!==>***ELIMINADO***
EOF

# 3. Reescritura
git filter-repo --replace-text reemplazos.txt
# Parsed 342 commits
# New history written in 1.87 seconds; now repacking/cleaning...
# Rewrote the history; completely finished after 3.42 seconds.
```

Si lo que sobra es el fichero entero (un `.pfx`, un `.env`), la variante es `git filter-repo --path secrets/pasarela.pfx --invert-paths`. En ambos casos, **todos los SHA posteriores al primer commit afectado cambian**, así que hay que forzar el push (`git push --force --all` y `--force --tags`), y eso exige coordinación previa con el equipo y desbloquear temporalmente las ramas protegidas.

Y aquí el punto que confunde a mucha gente: **`git commit --amend` y `git rebase -i` no sirven si ya hiciste push.** Los dos reescriben *tu* historia local, pero el commit original sigue en el servidor como objeto propio, con su SHA, alcanzable aunque ninguna rama lo apunte — `git show 4b1c9de:src/Tienda.Api/appsettings.json` sigue devolviendo el fichero con la clave. Además, `--amend` solo alcanza al último commit y `rebase` solo a lo que le indiques: si el secreto lleva ahí treinta commits, ninguno de los dos lo toca.

### 3. Avisar al equipo y saber qué queda fuera de tu alcance

Reescribir el historial rompe todos los clones existentes. Quien haga `git pull` sobre el historial viejo generará un merge que **reintroduce los commits con el secreto**, y volvéis al punto de partida. El aviso tiene que ser explícito: *no hagáis pull, borrad la copia local y volved a clonar*. Quien tenga trabajo sin subir, que lo guarde con `git format-patch` o `git stash` y lo reaplique después sobre el clon nuevo.

Lo que **no** puedes arreglar tú:

- **Los forks son repositorios independientes.** Tu reescritura no los toca. Hay que pedir a cada propietario que rehaga el fork, y si el repositorio era público asume que la lista de forks es incompleta.
- **Los objetos pueden seguir accesibles por SHA en la plataforma**, incluso sin rama que los referencie, hasta que la recolección de basura del servidor los elimine. En GitHub esto no lo controlas desde el cliente: hay que abrir un ticket al soporte para forzar la purga de los objetos huérfanos y de las vistas en caché de los commits. La [guía oficial de GitHub para eliminar datos sensibles](https://docs.github.com/es/authentication/keeping-your-account-secure/removing-sensitive-data-from-a-repository) lo dice con estas palabras y por eso insiste en rotar.
- **Las cachés de CI, los artefactos y los logs de construcción** son almacenamiento aparte. Si algún paso imprimió la configuración, el secreto está en un log que sobrevive a la reescritura. Púrgalos también.

La lectura conjunta de los tres puntos es la misma de siempre: la única acción que de verdad cierra el incidente es la del paso 1.

## Errores frecuentes

| Síntoma / lo que se dice | Causa real |
|---|---|
| "Lo borré en el commit siguiente, ya está" | El commit anterior sigue existiendo y `git show <sha>:fichero` recupera el valor. El borrado añade historia, no la quita |
| "Es un repositorio privado, no pasa nada" | Confunde el control de acceso de hoy con el de toda la vida del proyecto: reorganizaciones, permisos heredados, colaboradores externos, forks y brechas de la plataforma |
| "Solo es la clave de *staging*" | O bien staging apunta a datos reales (habitual), o bien la clave es la misma que en producción porque nadie las separó. Y aunque no lo sea, entrar en staging es el primer paso para entrar en producción |
| "No lo puse en el fichero de configuración, lo puse en una constante del código" | Peor: un `const string ApiKey = "sk_live_..."` está igual de comiteado, no lo tapa ningún `.gitignore`, ningún escáner de ficheros de config lo mira y además va compilado en el binario que distribuyes |
| Añadí `.env` al `.gitignore` y sigue apareciendo en los commits | `.gitignore` no afecta a lo que ya está en el índice. Hace falta `git rm --cached .env` — y el historial anterior sigue conteniendo los valores |
| El escáner no encuentra nada y aun así hubo fuga | Las contraseñas con forma de palabra (`Verano2026!`) no tienen prefijo reconocible ni entropía alta: ningún patrón las detecta |
| "Rotamos la clave y limpiamos el historial, pero siguen entrando" | Queda una copia viva en otro sitio: caché de CI, log de despliegue, fork, o una segunda credencial que se creó sin revocar la primera |

## Cuándo la regla no aplica

- **Un valor no es secreto por parecerlo.** Una cadena larga y aleatoria puede ser perfectamente pública: un `ClientId` de OAuth, un identificador de instalación, una clave pública. Sacarlos del repositorio "por si acaso" añade fricción a cada persona que clona y no aporta nada.
- **Los secretos de juguete, si de verdad lo son.** La contraseña de una base de datos que solo existe dentro del `docker-compose.yml` de desarrollo (`Password=dev`), que no escucha en la red y que se reconstruye vacía en cada arranque, puede vivir en el repositorio: no da acceso a nada. El problema empieza cuando ese mismo fichero se reutiliza para un entorno compartido y nadie cambia el valor.
- **Los secretos cifrados cuya clave maestra no está en el repositorio.** Patrones como SOPS o `git-crypt` versionan el fichero cifrado a propósito. No es una excepción a la regla, es la regla aplicada un nivel más abajo, y depende por completo de que la clave de descifrado viva en otro dominio de confianza — lo desarrolla [Cifrado en reposo de credenciales](Cifrado-En-Reposo-De-Credenciales.md).

## Buenas prácticas avanzadas

- **Trata la primera fuga como un fallo del proceso, no de la persona.** Si un secreto llegó a un commit, es que era posible que llegara: no había hook, ni escaneo en CI, ni push protection. Añade la capa que faltaba en el mismo incidente. Culpar a quien lo comiteó garantiza que la próxima persona lo esconda en lugar de avisar, y un incidente ocultado es infinitamente peor que uno gestionado.
- **Pon un límite de gasto en cada proveedor de pago por uso el día que creas la cuenta.** Es la única medida que acota el daño de una clave filtrada por una vía que no controlas (una captura de pantalla, un log, un fork viejo). Rotar limita el tiempo de exposición; el límite de gasto limita el importe, y es lo que diferencia un susto de una factura que hay que negociar.
- **Fuerza credenciales distintas por entorno, aunque el proveedor te deje reutilizarlas.** Es el requisito que hace posible responder rápido: con una clave por entorno, revocar la de desarrollo es una decisión de treinta segundos que no interrumpe a nadie. Con una clave compartida, cada rotación es un despliegue coordinado, y esa fricción es exactamente lo que hace que la gente aplace la rotación tras un incidente pequeño.
- **Escanea el historial completo una vez, hoy, aunque el equipo lleve años siendo cuidadoso.** `gitleaks detect` sobre todos los commits tarda segundos y el resultado casi nunca es cero: aparecen claves de servicios que ya no usáis, contraseñas de bases de datos de hace tres migraciones, tokens personales de gente que ya no está. Cada hallazgo es una credencial que hay que revocar aunque creas que ya no sirve, porque "creo que está desactivada" no es una comprobación.
- **Haz que el arranque falle cuando falte un secreto, en lugar de tirar de un valor por defecto.** Un `?? "clave-de-desarrollo"` en la lectura de configuración convierte un error de despliegue evidente en un servicio a medio funcionar con credenciales de juguete — y, si es la clave de firma de JWT, en un sistema donde cualquiera puede fabricar tokens porque la clave está en el código fuente. Validar la configuración al arrancar (`ValidateOnStart` en .NET) transforma un problema silencioso de seguridad en un error de despliegue ruidoso y trivial de arreglar.
- **Ten el procedimiento de rotación escrito, con el enlace al panel de cada proveedor.** El momento de averiguar dónde se revoca `Pasarela:ApiKey` no es el momento en que está filtrada. Media página con "entra aquí, genera, despliega, revoca la anterior" convierte veinte minutos de pánico en tres de ejecución.

## Recursos didácticos

- [gitleaks](https://github.com/gitleaks/gitleaks) — el escáner de referencia. Merece la pena leer su fichero de reglas por defecto: te enseña de un tirón qué prefijos identifican las claves de los proveedores grandes, que es justo lo que hace posible el `git grep` de "Por qué git es una trampa". Su compañero [trufflehog](https://github.com/trufflesecurity/trufflehog) añade la verificación en vivo de cada hallazgo.
- [git-filter-repo](https://github.com/newren/git-filter-repo) — la herramienta oficialmente recomendada para reescribir historia. Su `README` incluye una comparación honesta con `filter-branch` y explica por qué el reemplazo de texto es preferible a borrar el fichero cuando el fichero hace falta.
- [Eliminar datos sensibles de un repositorio (GitHub Docs)](https://docs.github.com/es/authentication/keeping-your-account-secure/removing-sensitive-data-from-a-repository) — el procedimiento paso a paso, con la parte que ninguna herramienta local puede resolver: qué hacer con los forks y cómo pedir la purga de los objetos que siguen accesibles por SHA.
- [Secret scanning y push protection de GitHub](https://docs.github.com/es/code-security/secret-scanning/about-secret-scanning) — la lista de patrones soportados y cómo activar el rechazo en el push. Esa lista sirve además como criterio para saber qué credenciales tuyas tienen prefijo detectable y cuáles vas a tener que vigilar a mano.

---

*En resumen: git está diseñado para no olvidar nada, así que un secreto comiteado está comprometido para siempre — el objetivo no es esconderlo en el código sino sacarlo de él, y si ya llegó al historial se rota primero y se limpia después.*
