# Contribuir un fix a un módulo de la OCA

## ¿Qué es?

La OCA (*Odoo Community Association*) es una organización sin ánimo de lucro que mantiene miles de módulos de Odoo que no forman parte del núcleo oficial, agrupados en decenas de repositorios públicos en `github.com/OCA`. Contribuir un fix a uno de esos módulos sigue el mismo mecanismo de Pull Request que el núcleo de Odoo, pero con sus propias herramientas de calidad y su propio proceso de aceptación.

## ¿Por qué existe?

El fabricante de Odoo mantiene el núcleo y los módulos que considera parte del producto oficial, pero un ERP necesita cientos de piezas más específicas —una integración con un banco de un país concreto, un informe sectorial, una regla de negocio de un tipo de empresa muy particular— que no tiene sentido que mantenga el propio fabricante. La OCA existe para que esas piezas tengan un hogar compartido y mantenido en comunidad, en vez de que cada empresa reinvente su propia versión del mismo módulo por su cuenta y sin coordinarse con nadie.

Esa naturaleza de comunidad, en vez de fabricante único, cambia el proceso de contribución en un punto importante: quien revisa y acepta los cambios no es el equipo de una empresa, sino los propios *mantenedores* del repositorio —normalmente personas de varias empresas distintas que usan ese módulo y se han comprometido a cuidarlo—.

> Si ya conoces cómo funciona un paquete de npm o una librería de NuGet mantenida por voluntarios, la OCA es ese mismo modelo aplicado a módulos de Odoo: código abierto, mantenido por quien lo usa, con reglas de calidad más estrictas que un proyecto interno precisamente porque nadie lo revisa "de oficio".

## ¿Cuándo y para qué se usa?

Se usa cuando el módulo con el bug no es propio ni es del núcleo de Odoo, sino uno de terceros instalado desde alguno de los repositorios de `github.com/OCA` (localizaciones fiscales de un país, conectores, utilidades de gestión de proyectos o de recursos humanos que Odoo no trae de serie...). Si el módulo lo mantiene la propia empresa que lo usa, esto no aplica: se corrige directamente en su propio repositorio, sin Pull Request externo.

---

## Cómo está organizado el catálogo de la OCA

La OCA no tiene un único repositorio con todos los módulos: los agrupa por **temática** en repositorios independientes (contabilidad, gestión de proyectos, recursos humanos, comercio electrónico...), y cada uno de esos repositorios tiene, a su vez, **una rama por versión de Odoo** (`16.0`, `17.0`, `18.0`...), igual que el núcleo. El bug que se quiere arreglar vive en el repositorio de esa temática, en la rama de la versión en uso.

Cada repositorio publica quién lo mantiene: una lista de personas (los *maintainers*) con permiso para aprobar y fusionar cambios. Antes de empezar, merece la pena mirar esa lista y el ritmo de actividad del repositorio (Pull Requests recientes, tiempo medio hasta que se cierran): un repositorio con mantenedores activos responde en días; uno abandonado puede no responder nunca, por bien hecho que esté el fix.

## Fork, rama y entorno

El mecanismo de partida es idéntico al del núcleo de Odoo: fork del repositorio concreto, remoto `upstream` apuntando al original de la OCA, y rama nueva partiendo de la rama de versión correspondiente.

```bash
git clone https://github.com/<usuario>/<repositorio-oca>.git
cd <repositorio-oca>
git remote add upstream https://github.com/OCA/<repositorio-oca>.git
git fetch upstream
git checkout -b 18.0-fix-nombre-descriptivo upstream/18.0
```

## Los linters no son opcionales

Aquí es donde el proceso de la OCA es notablemente más estricto que el del núcleo de Odoo: todos los repositorios llevan configurados unos *hooks* de `pre-commit` que reformatean y comprueban el código automáticamente antes de cada commit, y ese mismo conjunto de comprobaciones se repite en la integración continua — si no pasan en la máquina propia, tampoco pasarán en el Pull Request.

```bash
pip install pre-commit
pre-commit install           # se ejecuta a partir de ahora en cada "git commit"
pre-commit run --all-files   # para comprobar el repositorio entero de una vez
```

Entre las comprobaciones habituales: formateo automático de Python (`black`), orden de imports (`isort`), un linter propio de Odoo (`pylint-odoo`) que detecta errores específicos del framework —un modelo sin `_description`, un campo sin `string` traducible, una consulta SQL directa donde el ORM ya resuelve el problema—, y formateo de las vistas XML. La primera vez que se instalan los hooks en un repositorio grande, es normal que reformatee ficheros enteros que nadie ha tocado: eso es intencionado, para que todo el repositorio hable el mismo estilo.

## Convención de commits y Pull Request

El formato de commit es el mismo prefijo entre corchetes que usa el núcleo (`[FIX] <modulo>: <descripcion>`), y cada repositorio trae su propia plantilla de descripción de Pull Request —aparece ya rellena al abrirlo desde GitHub— pidiendo, como mínimo, qué problema resuelve y cómo se ha probado.

## Sin CLA, pero con licencia explícita en cada módulo

A diferencia del núcleo de Odoo, la OCA no exige firmar ningún Contributor License Agreement por separado: cada módulo declara su propia licencia (habitualmente AGPL-3 o LGPL-3) en el propio manifest, y al abrir un Pull Request se acepta contribuir bajo esa misma licencia. Es un requisito más ligero, pero no inexistente: conviene revisar la licencia del módulo concreto antes de asumir que su código se puede reutilizar en otro sitio con condiciones distintas.

## Traducciones: un flujo aparte

Si el fix toca texto visible para el usuario (una etiqueta de campo, un mensaje de error), no hace falta traducirlo uno mismo: la OCA centraliza las traducciones de todos sus módulos en una instancia de **Weblate**, una herramienta web donde cualquier persona de la comunidad puede traducir cadenas a su idioma de forma independiente del código. Basta con que la cadena nueva esté en inglés y correctamente marcada como traducible; la traducción a otros idiomas llega después, por esa vía, sin bloquear el Pull Request.

## El bot de fusión y la revisión

La revisión la hacen los mantenedores del repositorio, no un equipo interno de un fabricante, y suele ser bastante más rápida y abierta que la del núcleo — es habitual que cualquier persona de la comunidad comente sugerencias, no solo los mantenedores oficiales. Pero fusionar el cambio no es un botón de GitHub: se hace comentando una orden a un bot propio de la OCA (del tipo `/ocabot merge patch`, indicando si el cambio es un `patch`, `minor` o `major` según su alcance), y solo una persona con permiso de mantenedor puede escribir esa orden con efecto. El bot se encarga de fusionar, incrementar la versión del módulo en su manifest y publicar el cambio.

## Manteniendo al día un módulo OCA que ya se usa

Si un proyecto depende de un módulo de la OCA como parte de sus módulos instalados —clonado directamente, con un submódulo de git, o con una herramienta de agregación de repositorios—, el fix que se acaba de proponer no llega a la instalación propia en cuanto se fusiona, por la misma razón que en el núcleo: hace falta traer esa actualización a la copia local. La diferencia práctica con el núcleo es que aquí suele controlarse uno mismo cuándo se actualiza esa dependencia, así que conviene decidir con calma si se actualiza solo el commit con el fix propio (más seguro, cambio mínimo) o toda la rama del repositorio OCA (trae también otros cambios de otras personas desde la última actualización).

## Buenas prácticas avanzadas

- **Instala los hooks de `pre-commit` antes de escribir una sola línea, no al final.** Si se dejan para el último momento, el primer `git commit` puede reformatear archivos enteros y mezclar cambios de estilo ajenos con el fix real, lo que complica la revisión.
- **Revisa la lista de mantenedores activos antes de invertir tiempo.** Un repositorio de la OCA sin actividad reciente en Pull Requests puede tardar meses o no responder nunca, por correcto que sea el fix; en ese caso, a veces es mejor abrir primero un *issue* preguntando si el repositorio sigue mantenido.
- **No mezcles nunca el *bump* de versión del manifest con el propio cambio.** El bot de fusión de la OCA lo hace automáticamente a partir de lo que se indique en el comando de fusión (`patch`/`minor`/`major`); subir la versión a mano en el Pull Request suele generar conflictos con lo que hace el bot al fusionar.
- **Un Pull Request por módulo, aunque el repositorio contenga varios.** Los repositorios de la OCA agrupan muchos módulos independientes; mezclar cambios de dos módulos distintos en el mismo Pull Request obliga a que lo apruebe alguien con autoridad sobre ambos, cuando lo normal es que cada módulo tenga sus propios mantenedores.

## Documentación oficial

- [Guía de contribución de la OCA](https://github.com/OCA/maintainer-tools/wiki/Contributing) — el proceso completo: convención de commits, plantilla de Pull Request y cómo funciona el bot de fusión.
- [`pylint-odoo`](https://github.com/OCA/pylint-odoo) — el repositorio del linter específico de Odoo, con la lista completa de reglas que comprueba y por qué existe cada una.
- [Organización OCA en GitHub](https://github.com/OCA) — el catálogo completo de repositorios por temática, punto de partida para localizar en cuál vive el módulo que se quiere corregir.

## Recursos didácticos

- [Runboat](https://runboat.odoo-community.org/) — a partir de cualquier Pull Request abierto en un repositorio de la OCA, levanta una instancia de Odoo real con ese cambio ya instalado, sin tocar ningún servidor propio: la forma más rápida de comprobar que el fix funciona de verdad.

---

*En resumen: en la OCA el Pull Request es el mismo mecanismo que en el núcleo de Odoo, pero quien decide es la comunidad de mantenedores del repositorio, no un fabricante — y los linters automáticos existen precisamente porque nadie más va a revisar el estilo por ti.*
