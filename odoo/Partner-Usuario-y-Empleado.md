# Partner, Usuario y Empleado en Odoo

En Odoo hay tres modelos que representan "personas" y es fácil confundirlos, porque
a menudo hablan de **la misma persona real**... pero desde tres puntos de vista
distintos. Entender la diferencia es clave: muchas decisiones de desarrollo
dependen de *sobre qué modelo* estás trabajando.

Los tres son `res.partner`, `res.users` y `hr.employee`. Vamos de lo más general a
lo más específico.

---

## 1. `res.partner` — el contacto (la base de todo)

Es la entidad **fundamental** para cualquiera a quien Odoo necesite conocer:

- Clientes y proveedores.
- Empresas y personas de contacto dentro de esas empresas.
- Direcciones de envío o de factura.

Tiene nombre, email, teléfono, dirección, NIF... Es simplemente "una ficha de
contacto". **No** implica poder entrar en Odoo, ni trabajar en la empresa.

> 🔑 Idea clave: casi todo lo demás en Odoo, cuando se refiere a una persona o
> empresa, **acaba apuntando a un `res.partner`**. Es el ladrillo base.

---

## 2. `res.users` — la cuenta de acceso (quién entra y qué puede hacer)

Un `res.users` es alguien que **puede autenticarse** (login) en Odoo. Lo define:

- **Login y contraseña.**
- Sobre todo, sus **grupos y permisos**: qué ve y qué puede tocar en el sistema.

Detalle técnico importante: `res.users` **hereda de `res.partner`** por delegación
(en el código, `_inherits = {'res.partner': 'partner_id'}`). Esto significa que:

- **Cada usuario tiene siempre un partner detrás.**
- Campos como `user.name` o `user.email` en realidad viven en su partner; el
  usuario solo los "toma prestados".

En resumen: **usuario = un partner + capacidad de login + permisos.**

> Ejemplo: cuando en una tarea de proyecto asignas a alguien
> (`project.task.user_ids`), estás asignando **usuarios** (`res.users`), porque solo
> tiene sentido asignar trabajo a quien puede entrar en el sistema.

---

## 3. `hr.employee` — el empleado (la ficha laboral)

Un `hr.employee` es la ficha de **Recursos Humanos** de alguien que trabaja en la
empresa. Vive en la app de *Empleados* e incluye datos laborales:

- Departamento, puesto, responsable jerárquico.
- Email de trabajo, jornada, contrato...

Opcionalmente enlaza con un usuario (`employee.user_id`) y con un partner de
contacto de trabajo.

> Ejemplo: los **partes de horas** (`account.analytic.line`) y la sincronización con
> **Toggl** se apoyan en `hr.employee`, porque van de *quién ficha horas como
> trabajador*, no de quién se loguea.

---

## Cómo se relacionan

```
res.partner  (contacto: nombre, email, dirección)
   ▲                        ▲
   │ _inherits              │ (contacto de trabajo)
   │                        │
res.users  ◄──────────  hr.employee.user_id   (enlace OPCIONAL)
(login + permisos)       (ficha laboral)
```

---

## Lo más importante: NO es una relación 1:1

No des por hecho que una persona tiene los tres. Cada combinación es posible:

| Caso real | partner | user | employee |
|-----------|:---:|:---:|:---:|
| Un cliente | ✅ | ❌ | ❌ |
| Un consultor externo que solo entra a Odoo | ✅ | ✅ | ❌ |
| Un trabajador que ficha horas pero no entra a Odoo | ✅ | ❌ | ✅ |
| Un empleado de plantilla que trabaja y entra al sistema | ✅ | ✅ | ✅ |

Esto tiene consecuencias prácticas: **el puente `hr.employee.user_id` solo existe si
la persona tiene las dos caras.** Si construyes algo que salte de empleado a usuario
(o al revés), tienes que contemplar que ese enlace puede estar vacío.

---

## Regla mental rápida

- ¿Es un **contacto** cualquiera (cliente, proveedor, dirección)? → `res.partner`.
- ¿**Entra** en Odoo y tiene **permisos**? ¿Se le **asigna trabajo**? → `res.users`.
- ¿Es la **ficha laboral** de RRHH? ¿**Ficha horas**? → `hr.employee`.

---

## Caso real en Maccion

Esta distinción fue justo el centro de una decisión de diseño: el **token de Toggl**
vive en `hr.employee` (fichar horas) y los **filtros de personas de Scrum** filtran
por `res.users` (asignación de tareas). Son la misma gente, pero modelos distintos —
por eso no conviene meterlos en una sola lista. (Ver la propuesta
`menu-config-toggl-scrum.md` en el repo de Odoo.)
