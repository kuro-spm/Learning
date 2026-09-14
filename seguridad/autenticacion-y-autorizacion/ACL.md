# ACL (listas de control de acceso)

## ¿Qué es?

Una **ACL** (*Access Control List*, lista de control de acceso) es una lista de permisos que se cuelga **de cada recurso** e indica qué sujetos (usuarios o grupos) pueden hacer qué sobre ese recurso concreto. Cada línea de esa lista es una **ACE** (*Access Control Entry*): una tripleta *sujeto → permiso → permitir/denegar*.

## ¿Por qué existe?

Es la forma más directa de responder a la pregunta "¿quién puede tocar **este** recurso?": pegar la respuesta al recurso mismo. Un fichero, una carpeta, un objeto de un bucket o una fila de la base de datos llevan adosada su propia lista de "estas personas sí, estas no, y con qué permisos".

Es justo lo contrario del enfoque de [RBAC](RBAC-y-Claims.md). RBAC parte del **sujeto**: agrupa permisos en roles y asigna roles a las personas ("es Administrador, luego puede hacer X en todo el sistema"). ACL parte del **recurso**: cada recurso mantiene su propia lista de quién entra. Uno organiza por función; el otro, por objeto. Y son complementarios, no excluyentes: ACL responde barato a "¿quién puede tocar este recurso?" (miras su lista) pero caro a "¿qué puede tocar esta persona?" (tendrías que recorrer **todos** los recursos); RBAC es justo al revés. Por eso lo normal es combinarlos: RBAC para lo grueso por función, ACL para el "compartido con estas personas concretas".

> Piensa en el panel de "Compartir" de un documento en la nube: añades personas concretas una a una y a cada una le das "puede ver" o "puede editar". Esa lista de nombres pegada al documento **es** su ACL, y cada fila que añades es una ACE.

## ¿Cuándo y para qué se usa?

ACL brilla cuando los permisos son **por instancia** y se comparten de forma ad hoc, persona a persona:

- **Un sistema de ficheros**: cada carpeta y cada archivo llevan su propia lista de quién puede leer, escribir o ejecutar (las ACL de NTFS en Windows, o las POSIX en Linux).
- **Un documento compartido**: un informe que solo ven las tres personas a las que se lo has compartido explícitamente, cada una con su nivel (ver / comentar / editar).
- **Un bucket de almacenamiento en la nube**: cada objeto puede tener una ACL que dice qué cuentas concretas lo descargan.
- **La ficha de un producto en el catálogo de una tienda online**: normalmente la gestiona cualquier `Empleado` por rol, pero si además necesitas dar acceso puntual a una persona externa (un proveedor que solo debe editar la ficha de sus propios productos), eso ya no lo expresa un rol: es una ACE sobre ese recurso concreto.

La señal de que quieres ACL y no roles: cuando el permiso depende del **recurso en sí** ("estas personas concretas sobre este documento concreto"), no de la función que alguien ocupa en la organización.

## La unidad es la ACE: sujeto + permiso + tipo

Cada entrada de la lista responde tres cosas: *quién*, *qué puede hacer* y *si es un permiso o una prohibición*.

```jsonc
// Una ACE conceptual
{
  "sujeto": "grupo:contabilidad",   // usuario o grupo
  "permiso": "read",                // read | write | delete | ...
  "tipo": "allow"                   // allow | deny
}
```

Y la ACL es la lista completa de esas entradas, pegada a un recurso:

```jsonc
// ACL del recurso /informes/ventas-2026.pdf
{
  "recurso": "/informes/ventas-2026.pdf",
  "acl": [
    { "sujeto": "usuario:ana",        "permiso": "write", "tipo": "allow" },
    { "sujeto": "grupo:direccion",    "permiso": "read",  "tipo": "allow" },
    { "sujeto": "grupo:becarios",     "permiso": "read",  "tipo": "deny"  }
  ]
}
```

Muchas ACL admiten entradas de **denegación** explícita, no solo de permiso. La regla habitual (por ejemplo en NTFS) es que **un `deny` explícito gana** a cualquier `allow`, y que si ningún ACE coincide con el sujeto, el resultado por defecto es **denegar**. El orden y la prioridad exactos dependen de cada sistema: no los des por supuestos, ya que cómo se resuelven `allow` frente a `deny`, y cómo se hereda una ACL entre carpetas, cambia entre NTFS, POSIX y los buckets de la nube.

## Un caso real en Linux: POSIX ACL

Más allá del clásico dueño/grupo/otros, Linux permite dar permisos a usuarios y grupos nombrados con `setfacl`, y consultarlos con `getfacl`:

```bash
# Dar a la usuaria 'ana' permiso de lectura y escritura sobre un fichero
setfacl -m u:ana:rw informe.pdf

# Ver la ACL efectiva del fichero
getfacl informe.pdf
# user::rw-
# user:ana:rw-      <- ACE nombrada que acabamos de añadir
# group::r--
# mask::rw-
# other::---
```

Las ACL son la implementación clásica del control de acceso **discrecional** (*DAC*): quien posee el recurso decide, a su discreción, a quién se lo abre. Se opone al control **obligatorio** (*MAC*), donde las reglas las fija el sistema y el dueño no puede saltárselas (típico de entornos militares o de alta seguridad).

## Buenas prácticas avanzadas

- **Pon las ACE sobre grupos, no sobre personas** — asigna el permiso a `grupo:contabilidad` y gestiona quién pertenece al grupo; así un alta o una baja se resuelve en un sitio. Si atas cada permiso a individuos, cada cambio de plantilla te obliga a repasar las ACL de todos los recursos.
- **Audita la ACL *efectiva*, no la declarada** — mover o copiar un fichero puede arrastrar ACE heredadas del origen o romper la herencia sin avisar. Comprueba siempre el resultado real con la herramienta del sistema (`getfacl` en Linux, `icacls` en Windows), no lo que crees haber configurado.
- **Cuidado con la `mask` de las POSIX ACL** — `setfacl` mantiene una máscara que acota el máximo de permisos efectivos de los usuarios y grupos nombrados. Puedes dar `rwx` a alguien en una ACE y que la máscara lo recorte a `r--` sin que salte ningún error: el permiso "está", pero no surte efecto.
- **Nunca uses el sujeto comodín por comodidad** — conceder a `Everyone` (NTFS), a `other` (POSIX) o a `AllUsers` (buckets de la nube) es la causa número uno de fugas de datos en almacenamiento en la nube. Concede siempre a sujetos nombrados, aunque sea más trabajo.
- **Traza una frontera clara entre ACL y RBAC** — deja que RBAC decida lo estructural por función y que la ACL decida solo el "compartido con estas personas concretas". Si los dos modelos deciden sobre lo mismo, acabarás con permisos contradictorios imposibles de depurar ("el rol le deja, pero la ACL del recurso se lo niega... ¿o al revés?").

## Documentación oficial

- [`acl(5)` — man page de Linux](https://man7.org/linux/man-pages/man5/acl.5.html) — la especificación formal de las ACL POSIX: cómo se calculan los permisos efectivos y cómo funciona exactamente la `mask`.
- [`icacls` en Windows](https://learn.microsoft.com/windows-server/administration/windows-commands/icacls) — la referencia de la herramienta de línea de comandos para leer y modificar ACL de NTFS, con la sintaxis de cada modificador.

## Recursos didácticos

- En cualquier Linux tienes un laboratorio inmediato: crea un fichero de prueba y juega con `setfacl -m`, `getfacl` y `setfacl -x` para ver cómo cambian las ACE y la `mask` en tiempo real.
- En Windows, `icacls <ruta>` en la terminal muestra la ACL efectiva de un archivo o carpeta y es la forma más rápida de entender la herencia (`(I)` marca las ACE heredadas).

---

*En resumen: una ACL cuelga del recurso la lista de quién puede hacer qué sobre él — el modelo centrado en el objeto, complementario al enfoque centrado en el rol de RBAC.*
