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

*En resumen: configurar Git es presentarte una sola vez para que tu nombre quede firmado en cada cambio que hagas.*
