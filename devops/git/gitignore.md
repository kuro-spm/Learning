**¿Qué es?** Un archivo de texto llamado `.gitignore` donde listas los archivos y carpetas que quieres que Git **ignore**, es decir, que no rastree ni incluya en el historial.

---

## ¿Por qué existe?

No todo lo que hay en la carpeta de un proyecto debe versionarse. Hay archivos que se generan solos (dependencias descargadas, resultados de compilación), archivos enormes o, peor aún, secretos como contraseñas y claves de API. Subirlos al historial ensucia el repositorio o crea un riesgo de seguridad. `.gitignore` le dice a Git "estos ni los mires".

> Es como la lista de "no molestar": Git hace como si esos archivos no existieran.

---

## ¿Cuándo y para qué se usa?

Se crea al principio del proyecto y se va ajustando. Casos típicos: carpetas de dependencias (`node_modules/`), archivos de configuración local con credenciales (`.env`), o archivos temporales del sistema operativo.

---

## Lo mínimo que necesitas saber

**1. Crear el archivo**

Crea un archivo llamado `.gitignore` en la raíz del proyecto. Cada línea es un patrón a ignorar:

```gitignore
# Dependencias
node_modules/

# Variables de entorno con secretos
.env

# Archivos del sistema
.DS_Store

# Todo lo que acabe en .log
*.log
```

**2. El asterisco es un comodín**

`*.log` ignora cualquier archivo con esa extensión. `temp/` ignora una carpeta entera.

**3. Solo afecta a archivos aún no rastreados**

`.gitignore` ignora archivos que Git todavía no controla. Si un archivo ya está en el historial, añadirlo a `.gitignore` no lo elimina; primero hay que dejar de rastrearlo:

```bash
git rm --cached .env
```

**4. El propio `.gitignore` sí se versiona**

Suele commitearse junto al proyecto, para que todo el equipo ignore lo mismo.

---

## Lo que NO hace

- **No borra archivos** — solo evita que Git los rastree; siguen en tu disco.
- **No oculta lo que ya está en el historial** — si un secreto ya se subió, ignorarlo después no lo elimina del pasado.
- **No es retroactivo** — ver el punto 3.

---

## Buenas prácticas avanzadas

- **`git check-ignore -v` responde al "¿por qué no aparece este archivo?"** — cuando un archivo se ignora y no sabes por qué (¿el `.gitignore` del proyecto?, ¿uno de una subcarpeta?, ¿tu configuración global?), `git check-ignore -v ruta/al/archivo` te dice el archivo y la línea exactos del patrón responsable. Ahorra los diez minutos de buscar a ojo entre patrones.
- **No se puede "des-ignorar" dentro de una carpeta ignorada** — la negación `!` tiene una trampa: si ignoras `logs/`, un `!logs/importante.log` **no funciona**, porque Git ni siquiera entra en carpetas excluidas. El truco es ignorar el contenido en vez de la carpeta: `logs/*` seguido de `!logs/importante.log` sí funciona. Es el motivo n.º 1 por el que "la negación no me va".
- **Tus manías van en tu ignore global, no en el del proyecto** — los archivos de *tu* entorno (carpetas de tu editor, `.DS_Store`, notas personales) no pintan nada en el `.gitignore` compartido de cada repo. Configura una vez `git config --global core.excludesFile ~/.gitignore` y lístalos ahí; para exclusiones puntuales de un solo repo sin ensuciar a nadie, existe además `.git/info/exclude`.

---

*En resumen: `.gitignore` es la lista de "esto no lo guardes" — mantiene fuera del historial lo que se genera solo, lo que pesa demasiado y, sobre todo, lo que nunca debería hacerse público.*
