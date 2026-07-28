# Organizaciones de GitHub

## ¿Qué es?

Una organización de GitHub es una cuenta compartida que posee repositorios en nombre de un grupo de personas, en lugar de en nombre de un individuo. Los repositorios pertenecen a la organización, y las personas entran y salen sin que la propiedad del código cambie de manos.

## ¿Por qué existe?

En una cuenta personal hay **una sola persona propietaria** y es inamovible: tú. Puedes invitar a colaboradores y darles permisos amplios, pero nadie más que tú puede borrar el repositorio, cambiar su visibilidad, transferirlo o gestionar la facturación. Eso convierte a esa persona en un punto único de fallo: si se va de vacaciones, deja la empresa o pierde el acceso a su segundo factor, el código sigue ahí pero nadie puede administrarlo.

La organización rompe ese acoplamiento entre "quién escribió el código" y "quién manda sobre el código". La propiedad pasa a ser una entidad, y sobre esa entidad puede haber **varias personas con rol de propietaria**. Además aparece la maquinaria que un equipo necesita y una cuenta personal no tiene: equipos con permisos heredados, políticas de seguridad aplicadas a todos los repositorios a la vez, secretos compartidos y un registro de auditoría.

> Si ya conoces la administración de sistemas, piensa en la diferencia como la que hay entre un fichero cuyo dueño es un usuario y un fichero cuyo dueño es un grupo: el segundo sobrevive a que el usuario desaparezca.

## ¿Cuándo y para qué se usa?

En cuanto el código deja de ser un proyecto personal. Los tres detonantes habituales:

- **Trabaja más de una persona en él.** No por los permisos de escritura (eso se resuelve con colaboradores), sino porque hace falta que más de una persona pueda administrar.
- **Es de un cliente o de una empresa.** El código de una tienda online que se factura a un cliente no debería vivir en la cuenta personal de quien lo empezó. Si esa persona se va, hay que negociar una transferencia en lugar de simplemente revocarle el acceso.
- **Necesitas políticas.** Exigir revisión antes de fusionar, obligar al segundo factor, tener un registro de quién cambió qué en la configuración. Casi todo eso solo existe a nivel de organización, o solo se desbloquea en repositorios privados cuando la organización tiene un plan de pago.

Un ejemplo típico: una consultora con cinco proyectos de cinco clientes distintos. Con cuentas personales tendría cinco repositorios repartidos entre las cuentas de quien arrancó cada proyecto, cinco configuraciones distintas y ninguna forma de aplicar una regla común. Con una organización tiene un único sitio donde definir "en todos los repositorios, `main` está protegida y hace falta una aprobación".

## Cuenta personal frente a organización

Lo que de verdad cambia:

| | Cuenta personal | Organización |
|---|---|---|
| Propiedad | Una persona, inamovible | La entidad; **varias personas con rol Owner** |
| Administración | Solo la propietaria (los colaboradores con rol Admin llegan casi, pero no del todo) | Todos los Owners, por igual |
| Agrupar personas | No existe | **Teams**, con permisos heredados y anidables |
| Permiso por defecto | No hay: se concede repo a repo | **Base permissions** para toda la organización |
| Secretos compartidos | No: se repiten en cada repositorio | Secretos y variables **a nivel de organización** |
| Políticas globales | No | 2FA obligatorio, rulesets de organización, políticas de Actions |
| Registro de auditoría | No | Sí |
| Facturación | Personal | De la organización, por licencias |

El matiz importante, porque es la pregunta que casi todo el mundo hace primero: en una cuenta personal **puedes** dar rol Admin a otra persona sobre un repositorio, y esa persona podrá gestionar secretos, reglas de rama y colaboradores. Lo que no podrá nunca es borrar el repositorio, cambiar su visibilidad, transferirlo ni tocar la facturación. Esas cuatro cosas se quedan con la propietaria. No hay "dos propietarias" en una cuenta personal; hay una propietaria y N administradores.

## Anatomía de una organización

Una organización contiene:

- **Repositorios**, con visibilidad pública, privada o *internal* (esta última, visible para cualquier miembro de la organización, solo existe en los planes Enterprise).
- **Miembros**, cada uno con un rol de organización (Owner, Member, y algunos roles especializados).
- **Colaboradores externos** (*outside collaborators*): personas con acceso a repositorios concretos que no son miembros de la organización. Útil para un freelance puntual.
- **Teams**: grupos de miembros a los que se concede acceso en bloque.
- **Packages** (imágenes Docker en `ghcr.io`, paquetes npm o NuGet) publicados bajo el nombre de la organización.
- **Projects**, **discussions**, y la configuración transversal: secretos, variables, rulesets, políticas de Actions, seguridad.

Por encima puede existir además una **enterprise account**, que agrupa varias organizaciones bajo una misma facturación y unas políticas comunes. Solo tiene sentido a partir de cierto tamaño; una empresa pequeña vive perfectamente con una sola organización.

## Cómo se nota en el día a día

Lo primero que cambia es el *namespace*. Todo lo que identificaba al repositorio por la persona pasa a identificarlo por la organización:

```bash
# Antes: repositorio en una cuenta personal
https://github.com/ana-dev/api-pedidos

# Después: el mismo repositorio en la organización
https://github.com/acme-store/api-pedidos
```

Eso arrastra los remotos de git, que hay que actualizar en cada clon local:

```bash
git remote set-url origin git@github.com:acme-store/api-pedidos.git
git remote -v
# origin  git@github.com:acme-store/api-pedidos.git (fetch)
# origin  git@github.com:acme-store/api-pedidos.git (push)
```

Y también el namespace de los paquetes. Si el pipeline publica imágenes Docker, la etiqueta cambia:

```yaml
# Antes
tags: ghcr.io/ana-dev/api-pedidos:v1.2.0
# Después
tags: ghcr.io/acme-store/api-pedidos:v1.2.0
```

En un workflow bien escrito esto no se toca a mano, porque se deriva del contexto:

```yaml
tags: ghcr.io/${{ github.repository_owner }}/api-pedidos:${{ github.ref_name }}
```

`github.repository_owner` vale `ana-dev` antes de la migración y `acme-store` después. El workflow sigue funcionando sin cambios: es la razón por la que conviene no escribir el propietario a mano en los YAML.

## Qué no cambia

Conviene decirlo porque genera dudas: **git no se entera de nada**. Los commits, el historial, los SHA, las ramas y los tags son idénticos. Una organización es una capa de permisos, políticas y facturación *alrededor* de los repositorios. Clonar, hacer commit, hacer push y fusionar funciona exactamente igual.

Tampoco cambia la identidad de quien commitea: los commits siguen firmados con el nombre y correo de cada persona, y se atribuyen a su cuenta personal de GitHub. Ser miembro de una organización no crea una segunda cuenta; tu cuenta personal *pertenece* a la organización.

## Cuándo no merece la pena

Ser honesto con esto ahorra trabajo:

- **Proyectos personales, apuntes, dotfiles.** No hay nada que gobernar. Una organización solo añade una capa de configuración que no aporta.
- **Un repositorio público de una sola persona con contribuciones por *fork* y pull request.** El modelo de forks ya resuelve la colaboración sin dar acceso a nadie.
- **Cuando el problema real es otro.** Si lo que necesitas es que alguien pueda revisar y fusionar, un colaborador con permiso de escritura basta. Migrar a una organización tiene sentido cuando el problema es de *administración* y de *continuidad*, no de acceso.

## Buenas prácticas avanzadas

- **Crea la organización antes de necesitarla, no cuando ya duele.** Migrar un repositorio con historial, issues, pull requests, releases, paquetes publicados y un pipeline de despliegue es una tarea con checklist. Migrar uno que acaba de nacer son treinta segundos. Si sospechas que un proyecto va a ser de más de una persona, arráncalo ya en la organización.
- **Nunca escribas el propietario a mano en los workflows.** Usa `${{ github.repository_owner }}` y `${{ github.repository }}` en imágenes, URLs y rutas. Es la diferencia entre que una transferencia sea transparente y que rompa el pipeline de release el día que menos te apetece.
- **Vigila el *bus factor* de los Owners, no solo el de los desarrolladores.** Es el error clásico: una organización con diez miembros y un único Owner tiene exactamente el mismo punto único de fallo que la cuenta personal de la que huiste. Dos Owners es el mínimo; tres si la organización factura a clientes.
- **Los colaboradores externos consumen licencia en los planes de pago.** Si tienen acceso a repositorios privados, cuentan como asiento facturable igual que un miembro. Un freelance que ya acabó su trabajo y sigue en la lista es dinero y superficie de ataque a la vez. Revísala cada trimestre.
- **Decide el nombre pensando en que es para siempre.** El nombre de la organización aparece en todas las URLs, en los remotos de git de cada clon, en las etiquetas de las imágenes Docker y en los identificadores de paquetes. Se puede renombrar y GitHub deja redirecciones, pero las redirecciones se rompen si alguien registra después el nombre viejo. Elige el nombre legal o comercial estable, no el del proyecto de moda.

## Recursos didácticos

- [Documentación oficial de organizaciones](https://docs.github.com/organizations) — la referencia; cada página indica en qué planes está disponible cada función.
- [GitHub Skills](https://skills.github.com/) — cursos interactivos cortos que se ejecutan sobre repositorios reales de tu cuenta.
- La CLI `gh` ([cli.github.com](https://cli.github.com/)) — permite inspeccionar y configurar casi todo lo que se ve en la interfaz web. Explorar con `gh api` es la mejor forma de entender qué objetos existen de verdad por debajo.

---

*En resumen: una organización no cambia tu forma de usar git, cambia quién tiene la llave del armario cuando tú no estás.*
