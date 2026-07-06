# Submódulos

## ¿Qué es?

Un submódulo es un repositorio de Git **dentro de otro repositorio**. El proyecto principal no copia el código del otro: solo guarda una referencia que apunta a un commit concreto de ese repositorio externo.

## ¿Por qué existe?

A veces tu proyecto necesita usar otro proyecto que también está en Git y que evoluciona por su cuenta: una biblioteca de componentes compartida entre varias aplicaciones, por ejemplo. Podrías copiar y pegar su código, pero entonces perderías su historial y sería un infierno mantenerlo al día. Los submódulos te dejan **incluir ese repositorio tal cual**, fijado a una versión concreta, sin mezclar los dos historiales.

> Imagina un libro que cita otro libro. En lugar de copiar capítulos enteros, pone una nota: "ver *Tal libro*, edición de 2024". El submódulo es esa referencia precisa a una versión exacta de otro proyecto.

## ¿Cuándo y para qué se usa?

- Para **reutilizar una biblioteca propia** en varios proyectos manteniéndola como repositorio independiente.
- Para **incluir una dependencia** que quieres tener fijada a un commit exacto y bajo tu control.

Es una herramienta potente pero con fama de incómoda; úsala solo cuando de verdad necesitas dos repositorios separados pero conectados.

## Lo mínimo que necesitas saber

**1. Añadir un submódulo**

```bash
git submodule add https://github.com/usuario/ui-compartida.git librerias/ui
```

Esto crea la carpeta `librerias/ui` con el otro repo dentro y un archivo `.gitmodules` que registra la conexión.

**2. Clonar un proyecto que tiene submódulos**

Un `git clone` normal deja las carpetas de los submódulos **vacías**. Para traerlos:

```bash
git clone --recurse-submodules https://github.com/usuario/mi-app.git
```

Si ya clonaste sin esa opción:

```bash
git submodule update --init --recursive
```

**3. Actualizar un submódulo a su última versión**

```bash
cd librerias/ui
git pull origin main
cd ../..
git add librerias/ui
git commit -m "Actualiza la librería de UI a la última versión"
```

El proyecto principal guarda *a qué commit* apunta el submódulo, así que actualizarlo es un cambio que también se commitea.

## Lo que NO hace

- **No copia el código al historial del proyecto principal** — solo guarda la referencia a un commit del otro repo.
- **No se actualiza solo** — el submódulo queda fijado a un commit hasta que tú decides moverlo.
- **No es la única forma de reutilizar código** — muchas veces un gestor de paquetes (npm, NuGet...) es más sencillo. Plantéate si de verdad necesitas un submódulo.

## Buenas prácticas avanzadas

- **El error que rompe el proyecto a todo el equipo: referenciar un commit sin subir** — si commiteas dentro del submódulo, actualizas la referencia en el proyecto principal y haces push solo del principal, el resto del equipo apuntará a un commit que **no existe** en el remoto del submódulo: su `submodule update` fallará. Protégete con `git push --recurse-submodules=on-demand` (sube primero los submódulos pendientes) o al menos `=check`, que aborta el push si detecta el problema.
- **`git config submodule.recurse true` para que dejen de "aparecer como modificados"** — el síntoma clásico: cambias de rama o haces pull, y el submódulo figura como cambiado sin que nadie lo tocara. Es que su carpeta se quedó en el commit antiguo. Con `submodule.recurse` activado, `pull`, `switch` y compañía actualizan los submódulos solos, eliminando el 90 % de la fama de incómodos que tienen.
- **Haz legible el diff de submódulos** — un `git diff` normal solo muestra dos hashes crípticos cuando cambia la referencia. Con `git config --global diff.submodule log`, verás en su lugar la lista de commits del submódulo que entran o salen — imprescindible para revisar qué estás actualizando exactamente antes de commitear el cambio de puntero.

---

*En resumen: un submódulo es una referencia precisa a otro repositorio dentro del tuyo — reutilizas código ajeno fijado a una versión exacta, sin mezclar los dos historiales.*
