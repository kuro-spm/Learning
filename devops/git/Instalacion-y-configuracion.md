**¿Qué es?** El primer paso antes de usar Git: instalarlo y decirle quién eres. Esa identidad (nombre y correo) se adjunta a cada commit que hagas, para que el historial sepa de quién es cada cambio.

---

## ¿Por qué existe?

Git necesita saber el autor de cada cambio. Si no lo configuras, los commits quedan sin firmar o Git se niega a crearlos. Es una configuración que haces **una sola vez** por ordenador y te olvidas.

---

## ¿Cuándo y para qué se usa?

Justo después de instalar Git en un equipo nuevo, antes del primer commit. También cuando quieras cambiar tu nombre o correo, o ajustar preferencias globales como el editor de texto.

---

## Lo mínimo que necesitas saber

**1. Comprobar que Git está instalado**

```bash
git --version
# git version 2.45.0
```

Si no aparece una versión, descárgalo desde [git-scm.com](https://git-scm.com).

**2. Configurar tu identidad (una vez por ordenador)**

```bash
git config --global user.name "Ana García"
git config --global user.email "ana@ejemplo.com"
```

La opción `--global` aplica la configuración a todos tus proyectos. Sin ella, solo afecta al repositorio actual.

**3. Ver la configuración actual**

```bash
git config --list
```

**4. Cambiar el nombre de la rama por defecto**

```bash
git config --global init.defaultBranch main
```

Así los repositorios nuevos empezarán con la rama `main` en lugar del antiguo `master`.

---

## Lo que NO hace

- **No verifica tu identidad** — el correo es informativo; cualquiera puede escribir cualquier nombre. La autenticación real la gestionan servicios como GitHub.
- **No es lo mismo que iniciar sesión en GitHub** — esto solo configura Git localmente.

---

## Buenas prácticas avanzadas

- **Identidades distintas por carpeta con `includeIf`** — si usas un correo para el trabajo y otro para tus proyectos personales, no dependas de acordarte de configurarlo repo a repo. En tu `.gitconfig` global:

  ```gitconfig
  [includeIf "gitdir:~/trabajo/"]
      path = ~/trabajo/.gitconfig
  ```

  Todo repo bajo `~/trabajo/` usará automáticamente el correo definido en ese archivo. Se acabaron los commits personales firmados con el correo de la empresa (que, una vez subidos, ya no se arreglan).
- **En Windows, decide cómo se tratan los finales de línea** — Windows termina las líneas con CRLF y Linux/macOS con LF; sin configurarlo, aparecen diffs donde "cambia todo el archivo" sin que nadie tocara nada. `git config --global core.autocrlf true` en Windows (o, mejor aún, un archivo `.gitattributes` en el repo con `* text=auto`) elimina esa clase entera de ruido en equipos mixtos.
- **`push.autoSetupRemote` elimina el error más repetido de Git** — el famoso "fatal: The current branch has no upstream branch" en el primer push de cada rama. Con `git config --global push.autoSetupRemote true`, `git push` a secas crea y vincula la rama remota él solo, y no vuelves a copiar el comando que Git te sugiere.

---

*En resumen: configurar Git es presentarte una sola vez para que tu nombre quede firmado en cada cambio que hagas.*
