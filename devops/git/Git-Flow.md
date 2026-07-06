# Git Flow

## ¿Qué es?

Git Flow es un **modelo de ramas**: un conjunto de reglas sobre qué ramas existen, para qué sirve cada una y cómo fluye el trabajo entre ellas. No es un comando de Git, sino una convención que un equipo decide seguir.

## ¿Por qué existe?

Git te deja crear ramas libremente, pero esa libertad en equipo puede volverse un caos: ¿dónde va el trabajo en curso?, ¿dónde lo que ya es estable?, ¿cómo metemos un arreglo urgente sin frenar lo demás? Git Flow responde a todo eso con una estructura fija de ramas con roles claros, para que todo el equipo trabaje igual y sin sorpresas.

> Es como las normas de circulación: Git te da las calles (las ramas), pero Git Flow define los carriles y las señales para que nadie choque.

## ¿Cuándo y para qué se usa?

Encaja bien en proyectos con **versiones publicadas periódicamente** y varias personas trabajando a la vez (por ejemplo, una aplicación de escritorio que saca actualizaciones cada pocas semanas). Para proyectos pequeños o de despliegue continuo suele ser demasiado: ahí se prefieren modelos más simples (ver *Lo que NO hace*).

## Lo mínimo que necesitas saber

**1. Las dos ramas permanentes**

- **`main`**: siempre contiene la versión estable, la que está en producción.
- **`develop`**: la rama de integración donde se va juntando el trabajo en curso para la próxima versión.

**2. Las ramas temporales**

| Rama | Sale de | Vuelve a | Para qué |
|---|---|---|---|
| `feature/*` | `develop` | `develop` | Desarrollar una función nueva. |
| `release/*` | `develop` | `main` y `develop` | Preparar y pulir una versión antes de publicarla. |
| `hotfix/*` | `main` | `main` y `develop` | Arreglar un error urgente en producción. |

**3. El flujo de una función**

```bash
git switch develop
git switch -c feature/carrito-de-compra   # empiezas la función
# ...commits...
git switch develop
git merge feature/carrito-de-compra        # la integras al terminar
```

**4. Por qué los hotfix son especiales**

Salen directamente de `main` (lo que está en producción) para arreglar algo urgente sin arrastrar trabajo a medias de `develop`, y luego se aplican a ambas para que el arreglo no se pierda.

## Lo que NO hace

- **No es un comando de Git** — es una convención. Existe una extensión `git flow` que automatiza los pasos, pero puedes seguir el modelo solo con `branch`, `switch` y `merge`.
- **No es la mejor opción siempre** — para despliegue continuo es habitual usar modelos más ligeros como *GitHub Flow* (una sola rama principal + ramas cortas de función) o *trunk-based*.
- **No sustituye a la revisión de código** — se combina con [pull requests](Pull-requests.md), no los reemplaza.

## Buenas prácticas avanzadas

- **El *back-merge* a `develop` es el paso que todo el mundo olvida** — cada `hotfix/*` y cada `release/*` deben fusionarse en `main` **y también en `develop`**. Si el segundo merge se omite, el arreglo vive solo en producción y la siguiente release lo pisa: el bug "arreglado" reaparece. Es el fallo más común en equipos que usan Git Flow; algunos lo blindan con un check de CI que compara si `main` contiene commits ausentes en `develop`.
- **Etiqueta cada merge a `main`** — en Git Flow, `main` solo recibe releases y hotfixes, así que cada merge merece su tag (`v2.1.0`, `v2.1.1`). Sin esa disciplina pierdes la gracia del modelo: poder responder "¿qué hay exactamente en producción?" y volver a cualquier versión publicada al instante (ver [Tags y versiones](Tags-y-versiones.md)).
- **En una rama `release/*` solo entran correcciones** — su propósito es congelar el contenido de la versión mientras se pule. Colar "una funcioncita más" porque la rama sigue abierta invalida las pruebas hechas hasta entonces y alarga la release indefinidamente. Lo nuevo espera en `develop` a la siguiente versión; si no puede esperar, el problema es de planificación, no de Git.
- **Vigila la vida de las ramas `feature/*`** — el talón de Aquiles de Git Flow son las ramas de función que viven semanas alejándose de `develop`: la fusión final se convierte en una batalla de conflictos. Los equipos que lo dominan trocean el trabajo para fusionar cada pocos días y, mientras una feature siga abierta, la actualizan desde `develop` con regularidad en lugar de esperar al final.

---

*En resumen: Git Flow son las normas de circulación de las ramas de un equipo — aporta orden cuando hay versiones y varias personas, a cambio de cierta ceremonia que no todos los proyectos necesitan.*
