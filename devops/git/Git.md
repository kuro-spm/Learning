# Git

## ¿Qué es?

Git es un **sistema de control de versiones distribuido**: una herramienta que guarda el historial de cambios de un proyecto y permite volver a cualquier punto anterior, trabajar en varias cosas en paralelo y combinar el trabajo de varias personas sin pisarse.

## ¿Por qué existe?

Sin control de versiones, guardar el progreso acaba siendo copiar carpetas con nombres como `proyecto_final`, `proyecto_final_v2`, `proyecto_final_BUENO_ahora_si`. Pierdes el rastro de qué cambió, cuándo y por qué, y juntar el trabajo de dos personas se convierte en un caos de copiar y pegar.

Git resuelve justo eso: registra cada conjunto de cambios como una "foto" del proyecto (un *commit*) con su autor, fecha y descripción, y te deja moverte por esa línea de tiempo cuando quieras.

> Piensa en Git como el historial de versiones de un documento en la nube, pero para todo un proyecto y con mucho más control: puedes ver qué línea exacta cambió en cada versión, quién la escribió y por qué.

## ¿Cuándo y para qué se usa?

Se usa en casi cualquier proyecto de software, desde un script personal hasta una aplicación con cientos de personas trabajando a la vez. Por ejemplo:

- En un **blog personal**, para guardar el historial y recuperar una versión anterior si algo se rompe.
- En una **tienda online** desarrollada por un equipo, para que varias personas trabajen sobre el mismo código sin sobrescribirse entre ellas.
- En cualquier proyecto, para responder preguntas como "¿cuándo dejó de funcionar esto?" mirando el historial.

## Lo mínimo que necesitas saber

**1. Es distribuido**

Cada copia del proyecto (cada *clon*) contiene el historial completo. No dependes de un servidor central para ver versiones antiguas o seguir trabajando sin conexión.

**2. Los tres estados de un archivo**

Un archivo en Git pasa por tres estados:

- **Modificado**: lo has cambiado pero aún no lo has marcado para guardar.
- **Preparado (*staged*)**: lo has marcado para incluirlo en el próximo commit.
- **Confirmado (*committed*)**: ya está guardado en el historial.

**3. El flujo básico de trabajo**

```bash
# editas archivos...
git add archivo.txt     # preparas los cambios
git commit -m "Mensaje" # los confirmas en el historial
```

**4. Git no es GitHub**

Git es la herramienta que se ejecuta en tu ordenador. GitHub, GitLab o Bitbucket son servicios *en línea* que alojan repositorios Git para compartirlos y colaborar. Puedes usar Git sin ninguno de ellos.

## Lo que NO hace

- **No es una copia de seguridad automática en la nube** — guarda historial localmente; subirlo a un servidor es un paso aparte.
- **No resuelve los conflictos por ti** — cuando dos personas cambian la misma línea, te avisa, pero decides tú.
- **No gestiona bien archivos binarios enormes** — está pensado para texto (código); para vídeos o datasets grandes existen otras herramientas.
- **No es GitHub** — es fácil confundirlos, pero son cosas distintas.

## Buenas prácticas avanzadas

- **Piensa en fotos, no en diferencias** — cada commit guarda una instantánea completa del proyecto, no un parche con "lo que cambió". Los diffs que ves los calcula Git al vuelo comparando fotos. Con este modelo mental, comandos como `merge`, `rebase` o `cherry-pick` dejan de ser magia: todos consisten en mover o combinar instantáneas.
- **Interroga al historial con el "pickaxe"** — `git log -S "calcularIva"` encuentra los commits donde ese texto apareció o desapareció del código, y `git log -p -- archivo` te enseña la evolución de un archivo cambio a cambio. Es la forma experta de responder "¿cuándo y por qué se tocó esto?" sin leer commits al azar.
- **Git casi nunca borra de verdad** — los commits "perdidos" tras un error siguen en la base de datos interna durante semanas antes de que Git los limpie. Antes de dar nada por perdido, recuerda que existe una red de seguridad (ver [Deshacer cambios](Deshacer-cambios.md)); quien domina Git trabaja sin miedo porque sabe que casi todo se recupera.
- **Crea alias para lo que escribes veinte veces al día** — `git config --global alias.st "status -sb"` o `alias.lg "log --oneline --graph --all"` convierten comandos largos en dos letras. El formato `--oneline --graph` en particular te da una vista del árbol de ramas que cambia por completo cómo entiendes tu repositorio.

---

*En resumen: Git es la máquina del tiempo de tu proyecto — registra cada cambio para que nunca pierdas trabajo ni el rastro de cómo llegaste hasta aquí.*
