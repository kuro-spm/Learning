# Repositorio

## ¿Qué es?

Un repositorio (o *repo*) es una carpeta que Git vigila. Contiene tus archivos y, dentro de una subcarpeta oculta llamada `.git`, todo el historial de cambios del proyecto.

## ¿Por qué existe?

Git no vigila todo tu disco duro: necesita saber qué carpeta debe seguir. Convertir una carpeta en repositorio es declarar "a partir de aquí, quiero registrar el historial de estos archivos". Toda la magia (commits, ramas, historial) vive dentro de la carpeta `.git`; si la borras, pierdes el historial pero conservas los archivos actuales.

> Si ya conoces las bases de datos, piensa en la carpeta `.git` como una pequeña base de datos que guarda cada versión de tu proyecto.

## ¿Cuándo y para qué se usa?

- Cuando **empiezas un proyecto desde cero** y quieres versionarlo → `git init`.
- Cuando quieres **trabajar sobre un proyecto que ya existe** en internet (por ejemplo, en GitHub) → `git clone`.

## Lo mínimo que necesitas saber

**1. Crear un repositorio nuevo**

```bash
mkdir mi-blog
cd mi-blog
git init
# Initialized empty Git repository in .../mi-blog/.git/
```

A partir de ahí, esa carpeta tiene historial.

**2. Clonar un repositorio existente**

```bash
git clone https://github.com/usuario/tienda-online.git
```

Esto descarga el proyecto completo **con todo su historial** y lo deja listo para trabajar.

**3. La carpeta `.git`**

Está oculta dentro del repo. No la toques a mano: ahí vive todo el historial. Borrarla convierte la carpeta en un proyecto normal sin Git.

**4. Comprobar el estado**

```bash
git status
```

Es el comando que más usarás: te dice en qué rama estás, qué archivos has cambiado y qué está preparado para confirmarse.

## Lo que NO hace

- **`git init` no sube nada a internet** — solo crea el historial local. Conectar con un servidor es un paso aparte (ver [Remotos](Remotos.md)).
- **No versiona carpetas anidadas automáticamente como repos separados** — un repo cubre su carpeta y subcarpetas, salvo casos avanzados.
- **No empieza a guardar cambios solo** — tras `init` aún tienes que hacer `add` y `commit`.

## Buenas prácticas avanzadas

- **Varias carpetas de trabajo sobre un mismo repo con `git worktree`** — en vez de clonar dos veces o hacer malabares con stash para revisar otra rama, `git worktree add ../mi-blog-hotfix hotfix-precios` crea una segunda carpeta con esa rama desplegada, compartiendo el mismo `.git`. Puedes tener la función a medias en una carpeta y el arreglo urgente en otra, sin tocarte nada.
- **Clona ligero cuando el historial completo sobra** — en repositorios enormes, `git clone --filter=blob:none` descarga los archivos solo cuando los necesitas, y `--depth 1` trae únicamente la última versión. Es lo que usan los pipelines de CI para no descargar años de historial en cada ejecución.
- **El repo es 100 % autocontenido y portable** — todo (historial, ramas, configuración local) vive en `.git`; copiar la carpeta a otro disco u ordenador te da un repositorio idéntico y funcional. La otra cara: los remotos, hooks y configuración de `.git/config` viajan con la copia, pero nada de eso se sube nunca con `push` — lo que está solo en tu `.git/config` es invisible para el resto del equipo.

---

*En resumen: un repositorio es una carpeta con memoria — el momento en que le dices a Git "empieza a recordar lo que pasa aquí".*
