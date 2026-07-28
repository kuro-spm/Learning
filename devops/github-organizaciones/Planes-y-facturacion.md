# Planes y facturación

## ¿Qué es?

El plan de una organización de GitHub es su nivel de suscripción (Free, Team o Enterprise) y determina qué funciones de colaboración y de gobierno están disponibles, especialmente en los **repositorios privados**.

## ¿Por qué existe?

GitHub es gratis para código abierto casi sin límites: en un repositorio público tienes protección de ramas, revisiones obligatorias, entornos y automatización sin pagar nada. El modelo de negocio se apoya en otra parte: **en los repositorios privados, buena parte del gobierno está detrás del plan de pago**.

Esto tiene una consecuencia práctica que sorprende a mucha gente y que conviene saber antes de diseñar el flujo de trabajo: puedes montar una topología de ramas impecable, escribir el pipeline perfecto y documentar que "`main` está protegida"… y descubrir que en tu repositorio privado con plan Free **no existe** la opción de protegerla. La regla que creías tener es una convención que todo el mundo puede saltarse sin querer.

> Si ya conoces las licencias de bases de datos comerciales, el patrón es el mismo: el motor funciona igual, pero la réplica, la auditoría y el cifrado están en la edición de pago.

## ¿Cuándo y para qué se usa?

Elegir plan es una decisión de dos momentos:

- **Al crear la organización**, para no diseñar el flujo de trabajo sobre funciones que no vas a tener.
- **Cuando una regla de proceso se vuelve crítica.** El detonante habitual es un incidente: alguien fusiona a la rama de producción sin querer, o un despliegue sale sin revisar. Ahí es cuando se descubre que la protección que faltaba costaba unos pocos euros al mes por persona.

## Los tres planes

### Free

Gratis, con repositorios públicos y privados ilimitados y miembros ilimitados. Incluye lo esencial para trabajar: issues, pull requests, GitHub Actions con una cuota mensual de minutos para repositorios privados, GitHub Packages con cuota de almacenamiento, y la posibilidad de **exigir el segundo factor** a todos los miembros.

Lo que típicamente **no** tienes en repositorios *privados*: protección de ramas y rulesets, revisores obligatorios, *code owners* como revisión requerida, y las reglas de protección de entornos (revisores obligatorios antes de desplegar). En repositorios *públicos*, en cambio, casi todo eso sí está disponible.

### Team

De pago por persona y mes. Es el salto que compra el **gobierno en repositorios privados**: protección de ramas y rulesets, revisiones obligatorias, `CODEOWNERS` como revisión requerida, entornos con reglas de protección, más minutos de Actions y más almacenamiento de paquetes.

Para la mayoría de las empresas pequeñas y consultoras, Team es el plan correcto y el salto Free → Team es el único que de verdad se nota.

### Enterprise

De pago por persona y mes, bastante más caro. Añade lo que necesita una organización grande: **SAML single sign-on** y aprovisionamiento automático de usuarios con SCIM, API del registro de auditoría, repositorios *internal*, roles de repositorio y de organización personalizados, agrupación de varias organizaciones bajo una *enterprise account*, y la opción de contratar GitHub Advanced Security.

Si no tienes un proveedor de identidad corporativo al que conectar el SSO, probablemente no lo necesitas todavía.

## Cómo se factura

La facturación de una organización de pago es **por asiento** (*seat*), y esto es lo que más se malinterpreta:

- Cada **miembro** de la organización consume un asiento.
- Cada **colaborador externo con acceso a un repositorio privado** consume también un asiento. No es gratis por no ser miembro.
- Los asientos se cobran por mes; añadir a alguien a mitad de mes se prorratea, y quitarlo libera el asiento para el ciclo siguiente.
- Actions y Packages se facturan **aparte**, por consumo: los minutos y el almacenamiento incluidos en el plan son una cuota, y a partir de ahí se paga por uso. Ojo con los *runners* de Windows y macOS, que consumen minutos a un múltiplo del coste de Linux.

Un detalle que ahorra dinero real: los minutos de Actions **solo se cuentan en repositorios privados**. En repositorios públicos, los runners alojados por GitHub son gratuitos e ilimitados.

Puedes ver el estado de consumo desde la CLI:

```bash
# Minutos de Actions consumidos este ciclo por la organización
gh api /orgs/acme-store/settings/billing/actions
```

Devuelve algo parecido a esto, donde `included_minutes` es la cuota del plan y `total_paid_minutes_used` lo que ya se está facturando aparte:

```json
{
  "total_minutes_used": 1840,
  "total_paid_minutes_used": 0,
  "included_minutes": 3000,
  "minutes_used_breakdown": { "UBUNTU": 1720, "WINDOWS": 120 }
}
```

## Cómo saber qué desbloquea cada plan, de verdad

Y aquí viene el consejo más útil de esta ficha: **no te fíes de ninguna tabla, incluida la de arriba**. GitHub mueve funciones entre planes con cierta frecuencia, casi siempre para abajo (cosas que eran de Enterprise pasan a Team, cosas de Team pasan a Free). Una tabla escrita hoy envejece en meses.

Las dos fuentes fiables:

1. **La página de precios oficial**: [github.com/pricing](https://github.com/pricing), que compara los planes lado a lado.
2. **La propia página de documentación de cada función.** Toda página de `docs.github.com` que describe una función lleva, arriba, una línea del tipo *"Available with GitHub Team and GitHub Enterprise Cloud"*. Esa línea es la verdad para esa función concreta, en el momento en que la lees.

Y la comprobación definitiva, que no falla nunca: **abre la pantalla de configuración y mira si la opción está**. Si en *Settings → Rules → Rulesets* de tu repositorio privado no puedes crear una regla, no la tienes, digan lo que digan las tablas.

## Qué hacer si estás en Free y necesitas gobierno ya

Tres salidas, en orden de calidad:

1. **Pasar a Team.** Es la solución real. Con un equipo de dos o tres personas, el coste mensual es del orden de una comida.
2. **Hacer público el repositorio**, si el código lo permite. En público tienes protección de ramas gratis. Rara vez es opción con código de cliente.
3. **Implementar la regla en el pipeline.** Es un parche, pero es un parche que funciona. Si no puedes impedir que alguien haga push a la rama de producción, al menos puedes hacer que el workflow lo detecte y falle:

```yaml
# .github/workflows/guard.yml — a "main" solo se llega desde "staging"
name: Guard de rama
on:
  push:
    branches: [main]

jobs:
  verificar-procedencia:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # necesario: sin historial completo no se puede comparar
      - name: El commit debe existir en staging
        run: |
          git fetch origin staging
          if ! git merge-base --is-ancestor HEAD origin/staging; then
            echo "::error::Este commit no viene de staging. Revierte y promociona por el flujo correcto."
            exit 1
          fi
```

`git merge-base --is-ancestor HEAD origin/staging` devuelve código 0 si el commit actual es antecesor de `staging`, es decir, si `staging` ya lo contiene. Si alguien empuja directamente a `main`, la condición falla y el workflow marca la ejecución en rojo con un mensaje explícito.

La diferencia con la protección nativa es clave y hay que tenerla clara: esto **avisa después**, no **impide antes**. El commit ya está en la rama; lo que consigues es que nadie pueda alegar que no lo sabía, y que quede registrado. Es una red, no una puerta.

## Buenas prácticas avanzadas

- **Decide el plan antes de diseñar la topología de ramas.** Si vas a documentar en un ADR que "la rama de producción está protegida y solo recibe merges desde integración", asegúrate primero de que tu plan permite protegerla. Documentar una protección que no existe es peor que no tenerla: crea una falsa sensación de seguridad en todo el equipo.
- **Audita los asientos cada trimestre, empezando por los colaboradores externos.** Es la fuga de dinero más común y, a la vez, un riesgo de seguridad: gente que acabó su trabajo hace meses y sigue con acceso de escritura. `gh api /orgs/ORG/outside_collaborators` te da la lista en un segundo.
- **Mueve a Linux todo lo que puedas en Actions.** Los runners de Windows y macOS consumen minutos a un múltiplo del coste de los de Linux. Un job de tests que corre en Windows por inercia y no por necesidad puede multiplicar tu factura de Actions varias veces.
- **Cachea dependencias antes de subir de plan por falta de minutos.** Cuando la cuota se agota, el reflejo es pagar más minutos. Casi siempre es más barato añadir `actions/cache` (o el `cache:` que ya traen `setup-node`, `setup-dotnet` y compañía) y recortar el tiempo de cada ejecución a la mitad.
- **Revisa cada año si algo que pagas ya bajó a tu plan.** Como el movimiento de funciones entre planes suele ser hacia abajo, es posible que estés en Enterprise por una única función que ahora está en Team. Es una revisión de diez minutos que puede recortar la factura de forma sustancial.

## Recursos didácticos

- [github.com/pricing](https://github.com/pricing) — la comparativa oficial, siempre actualizada.
- [Calculadora de precios de GitHub](https://github.com/pricing/calculator) — permite simular coste de asientos, minutos de Actions y almacenamiento antes de comprometerse.
- [Documentación de facturación](https://docs.github.com/billing) — cómo se prorratean los asientos y cómo poner límites de gasto para no llevarte sorpresas con Actions.

---

*En resumen: en repositorios públicos GitHub te lo da casi todo gratis; en privados, el plan es lo que separa una regla de verdad de una convención con buena fe.*
