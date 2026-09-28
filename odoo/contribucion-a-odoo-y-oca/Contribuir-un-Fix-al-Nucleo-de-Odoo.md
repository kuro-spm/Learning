# Contribuir un fix al núcleo de Odoo

## ¿Qué es?

Es el procedimiento para que una corrección de un bug del propio código de Odoo (el que se instala desde `github.com/odoo/odoo`) deje de vivir solo en tu servidor y pase a formar parte del proyecto oficial: un Pull Request revisado y aceptado por el equipo de Odoo.

## ¿Por qué existe?

Cuando encuentras un bug en el código propio de tu aplicación, lo arreglas y ya está: el cambio vive en tu repositorio y decides tú cuándo se despliega. Un bug en el código de Odoo es distinto en dos aspectos importantes.

El primero: ese fichero no es tuyo. Cientos de miles de instalaciones de Odoo en el mundo comparten exactamente el mismo código, así que un cambio directo sobre él no es "arreglar tu aplicación": es proponer un cambio a un proyecto compartido, y alguien con autoridad sobre ese proyecto tiene que aceptarlo antes de que forme parte de él.

El segundo: si simplemente editas el fichero en tu propio servidor, la próxima vez que actualices Odoo (una actualización de seguridad, una migración de versión) tu cambio desaparece sin avisar, porque una actualización reemplaza esos ficheros por los oficiales. Hace falta un mecanismo que sobreviva a las actualizaciones, y ese mecanismo es contribuir el fix aguas arriba.

> Si ya sabes cómo funciona una *pull request* de código abierto en general, este documento es la versión concreta de ese proceso aplicada a Odoo: qué rama usar, qué convenciones sigue el equipo y qué pasa después de que te acepten el cambio.

## ¿Cuándo y para qué se usa?

Se usa cuando el bug está en un módulo que Odoo distribuye de serie (los que viven en la carpeta `addons/` del propio proyecto, no en un módulo instalado aparte) y se quiere que la corrección quede disponible para cualquiera que use esa versión de Odoo, no solo para una instancia concreta. Casos típicos: un cálculo con las unidades equivocadas, una condición que no cubre un caso límite, un mensaje de error mal formado.

No se usa para arreglar un comportamiento que una empresa quiere distinto al de serie —eso es una personalización, y va en un módulo propio— ni para módulos de terceros que no sean del propio Odoo. Para esos existe un proceso parecido pero con sus propias reglas: ver [contribuir un fix a un módulo de la OCA](Contribuir-un-Fix-a-un-Modulo-OCA.md).

---

## Antes de escribir código: comprobar que hace falta

Dos búsquedas rápidas ahorran el resto del trabajo:

1. **El historial del fichero en GitHub**, filtrado a la rama de la versión en uso (`https://github.com/odoo/odoo/commits/<version>.0/ruta/al/fichero.py`). Si alguien ya lo arregló en un commit posterior al que está instalado, no hace falta ningún Pull Request: basta con aplicar ese commit (`git cherry-pick`) mientras llega la siguiente actualización oficial.
2. **Los *issues* y Pull Requests abiertos** con palabras clave del bug. Si ya hay uno describiendo lo mismo, lo suyo es sumarse a esa conversación en vez de duplicar el trabajo.

## Preparar el entorno: fork, remoto y rama

El desarrollo no se hace sobre el checkout que corre en producción: se hace sobre una copia propia del repositorio oficial.

```bash
# En GitHub: botón "Fork" de https://github.com/odoo/odoo hacia la cuenta propia
git clone https://github.com/<usuario>/odoo.git
cd odoo
git remote add upstream https://github.com/odoo/odoo.git
git fetch upstream
```

El segundo remoto (`upstream`) es el que permite comparar la copia propia con la oficial y traer sus cambios más adelante.

Odoo mantiene una rama por versión mayor: `17.0`, `18.0`, `19.0`... y una rama `master` que es la **próxima** versión, todavía en desarrollo. La corrección tiene que partir de la rama de la versión que realmente se usa: un Pull Request contra `master` para arreglar algo visto en producción con la versión 18 no ayuda a nadie que use la 18.

```bash
git checkout -b 18.0-fix-nombre-descriptivo-usuario upstream/18.0
```

El nombre de rama con el prefijo de versión y el sufijo del nombre de usuario es la convención que sigue el propio equipo de Odoo: deja claro, con solo mirar el nombre, contra qué versión va el cambio y de quién es.

## Escribir el fix

El cambio en sí sigue las mismas reglas que cualquier código nuevo: mínimo, centrado en el problema, sin tocar nada que no sea necesario para arreglarlo. Un Pull Request que corrige tres líneas y de paso reformatea el fichero entero es mucho más difícil de revisar —y de aceptar— que uno que toca solo lo imprescindible.

Odoo publica sus propias guías de estilo (convenciones de nombres, cómo estructurar un método, qué patrones evitar), distintas de un PEP8 genérico de Python en algunos puntos. Conviene leerlas antes de escribir el cambio, no después: un PR correcto en lógica pero que no sigue la convención del proyecto genera una ronda extra de comentarios pidiendo el mismo cambio de estilo.

## El test que demuestra el bug

Un fix sin test que lo cubra es una promesa, no una demostración. Si el módulo ya tiene tests para la función que se toca, conviene revisarlos primero: es habitual que el bug exista precisamente porque **el test existente no cubre el caso que falla** —prueba solo el camino feliz, o un caso demasiado amplio como para distinguir el código correcto del incorrecto—. En ese caso no basta con añadir un test nuevo: hay que entender por qué el que ya había no lo detectó, para no dejar el mismo punto ciego para el siguiente bug parecido.

El test ideal falla con el código de antes y pasa con el código de después. Ejecutarlo contra la rama propia antes de tocar nada, comprobar que efectivamente falla, y solo entonces aplicar el fix, es la forma de estar seguro de que el test prueba algo real y no un caso que siempre iba a pasar.

## El commit

Odoo usa un prefijo de tipo entre corchetes al principio del mensaje, seguido del módulo afectado y una descripción breve en inglés:

```
[FIX] <modulo>: <que corrige, en pocas palabras>
```

Un único commit lógico por Pull Request. Si hace falta corregir algo antes de que lo revisen, `git commit --amend` sobre el mismo commit; no se van acumulando commits de tipo "arreglo typo" o "de verdad ahora sí".

## Firmar el acuerdo de contribución (CLA)

Antes de que el Pull Request pueda aceptarse, Odoo exige firmar un **Contributor License Agreement** (CLA) individual: un documento breve donde se aceptan los términos bajo los que la contribución pasa a formar parte del proyecto. Se firma una sola vez, no por cada Pull Request, en la propia web de Odoo. Un bot automático comprueba la firma en cuanto se abre el PR y lo bloquea con un mensaje si falta.

## Abrir el Pull Request

```bash
git push origin 18.0-fix-nombre-descriptivo-usuario
```

Desde GitHub, el Pull Request se abre **contra la misma rama de versión** de la que se partió (`odoo:18.0`, no `odoo:master`). La descripción debe explicar tres cosas para quien lo revise sin haber visto el bug en persona: qué falla, cómo se reproduce (con datos genéricos, nunca datos reales de un cliente) y qué corrige el cambio.

## Integración continua y revisión

Al abrir el Pull Request, un sistema de integración continua ejecuta automáticamente la batería de tests del módulo —y de los módulos que dependen de él— contra la rama propuesta. Un test en rojo bloquea la revisión hasta que se corrige.

Superada esa comprobación automática, la revisión pasa a manos de una persona del equipo de Odoo. Al ser código nuclear —lo que usan todas las instalaciones del mundo, no un módulo de la comunidad—, esta revisión la hace el propio fabricante, no una comunidad de mantenedores abierta, y puede tardar: no hay un plazo garantizado, y es habitual recibir peticiones de cambio antes de la aceptación final.

## Después de que lo acepten

Aceptar el Pull Request no lo instala en ningún servidor. El cambio queda en la rama de la versión dentro del repositorio oficial; llega a una instalación real cuando esa instalación **actualiza su copia del código** desde ese punto en adelante, algo que normalmente ocurre con la siguiente actualización de mantenimiento que publica el fabricante, no al momento.

Mientras esa actualización no ha llegado al servidor propio, el bug sigue presente allí. La forma práctica de no depender del calendario de publicación de Odoo es aplicar la corrección uno mismo en su propia instalación, sin tocar el código que no es propio: un módulo aparte que hereda el modelo afectado y redefine únicamente el método corregido. El Pull Request y ese módulo propio no son alternativas: el primero arregla el problema para todo el mundo a medio plazo, el segundo protege la instalación ya, desde hoy.

## Buenas prácticas avanzadas

- **Un Pull Request, un problema.** Mezclar la corrección de un bug con una mejora de estilo del código de alrededor, aunque sea tentador viendo el fichero abierto, multiplica el trabajo de quien revisa y el riesgo de que rechacen el conjunto por una parte que no tiene nada que ver con el bug.
- **El test que demuestra el punto ciego vale más que el fix.** Cuando el bug existe porque un test anterior no lo detectaba, documentar exactamente qué caso no cubría —y por qué— ayuda tanto o más que la corrección en sí: evita que el mismo tipo de bug reaparezca en otro método parecido.
- **Revisa el histórico de la rama antes de abrir el PR, no solo al principio.** Entre que se empieza el fix y se termina pueden pasar semanas; si alguien ha tocado el mismo fichero mientras tanto, un `git rebase upstream/18.0` con los conflictos resueltos a mano evita un PR que ya no aplica limpio sobre la rama actual.
- **"Funciona en mi servidor" no basta como prueba.** La revisión humana espera el test automatizado como evidencia; una captura de pantalla del bug arreglado en una instancia concreta no sustituye un test que se ejecute en la integración continua de cualquiera.

## Documentación oficial

- [Guía de contribución de desarrollo de Odoo](https://www.odoo.com/documentation/18.0/contributing/development.html) — el punto de partida oficial: guías de estilo, estructura de commits y proceso de Pull Request, con el detalle que este documento resume.
- [Repositorio `odoo/odoo` en GitHub](https://github.com/odoo/odoo) — el código fuente y el histórico de commits y Pull Requests reales, útil para ver ejemplos aceptados de primera mano.

## Recursos didácticos

- [Runbot de Odoo](https://runbot.odoo.com/) — el panel público donde se ve en vivo el estado de la integración continua de los Pull Requests abiertos contra `odoo/odoo`, incluidos los propios mientras se revisan.

---

*En resumen: un fix al núcleo de Odoo no es solo código correcto, es código correcto además demostrado con un test, propuesto en la rama de la versión adecuada y con la paciencia de esperar una revisión humana — y mientras tanto, el propio servidor sigue necesitando su propio parche.*
