# Rulesets y protección de ramas

## ¿Qué es?

Las reglas que GitHub aplica del lado del servidor sobre las ramas y los tags de un repositorio: quién puede empujar, qué comprobaciones tienen que pasar antes de fusionar y qué operaciones están prohibidas. Los **rulesets** son el mecanismo moderno, y pueden definirse a nivel de repositorio o de organización; la **protección de ramas clásica** (*branch protection rules*) es el mecanismo anterior, todavía en uso.

## ¿Por qué existe?

Un flujo de trabajo documentado es una convención, no un control. Puedes escribir en un documento de arquitectura que "la rama de producción solo recibe fusiones desde integración, previa revisión", y ese documento no impide absolutamente nada: cualquiera con permiso de escritura puede hacer `git push origin main` a las once de la noche, o forzar un `push --force` que reescribe el historial y borra el trabajo de otra persona.

Las reglas mueven la convención al servidor. Con `main` protegida, ese push no falla en la conciencia de nadie: falla en el `git push`, con un mensaje que explica por qué. La diferencia entre las dos situaciones es la diferencia entre un proceso que se cumple y uno que se cumple casi siempre.

> Si ya conoces las restricciones de una base de datos, es la misma idea: puedes validar en la aplicación que un pedido siempre tenga cliente, pero hasta que no pones la clave foránea, tarde o temprano aparece una fila huérfana.

## ¿Cuándo y para qué se usa?

En cuanto la rama principal tiene un significado: se despliega, se factura, o simplemente es la que tiene que estar siempre verde. Los casos concretos:

- Impedir el push directo a la rama principal y obligar a pasar por pull request.
- Exigir que el CI pase en verde antes de poder fusionar.
- Exigir una o más aprobaciones.
- Prohibir el `force push` y el borrado de la rama.
- Proteger los tags de versión (`v1.2.3`) para que nadie los mueva después de publicar.

## Clásico frente a rulesets

Las dos cosas coexisten en la interfaz y conviene saber cuál usar:

| | Protección clásica | Rulesets |
|---|---|---|
| Dónde se define | Solo en el repositorio | Repositorio **y organización** |
| Reglas que coinciden | Se aplican todas, sin visibilidad de cuál | Se **acumulan**, con vista de cuáles aplican |
| Excepciones | Solo "incluir administradores" o no | **Lista de actores** que pueden saltarse la regla |
| Qué protege | Ramas | Ramas, **tags** y **pushes** |
| Estado | Soportado, sin evolución | El mecanismo actual |

Para un repositorio nuevo, **usa rulesets**. La ventaja decisiva no es ninguna regla concreta: es que un ruleset de organización aplica a todos los repositorios que encajen en un patrón, incluidos los que se creen mañana. Es la diferencia entre configurar la protección quince veces y configurarla una.

Los rulesets también tienen un estado de **evaluación** (`evaluate`): la regla se comprueba y se registra, pero no bloquea. Es la forma sensata de introducir una regla estricta en un equipo grande: primero mides a quién habría bloqueado, y solo después la activas.

## Las reglas que importan

### Pull request obligatorio

Impide el push directo: para que un commit llegue a la rama tiene que ir por un pull request. Ajustes que lleva dentro:

- **Número de aprobaciones** (una es lo razonable; dos si es código crítico y hay gente suficiente).
- **Descartar aprobaciones obsoletas** al llegar nuevos commits. Sin esto, alguien aprueba, el autor empuja un cambio completamente distinto y la aprobación sigue contando. Actívalo.
- **Requerir revisión de los code owners**, que es lo que convierte `CODEOWNERS` de sugerencia en requisito.
- **Requerir que las conversaciones estén resueltas** antes de fusionar, para que un comentario que señala un problema no se quede colgando.

### Comprobaciones de estado obligatorias

Los *status checks* son los jobs del CI. Elegir los que tienen que estar en verde para poder fusionar es la regla que más valor aporta por sí sola: garantiza que nada entra en la rama principal sin compilar y sin pasar los tests.

El ajuste **"requerir que la rama esté actualizada"** merece un párrafo aparte, porque tiene una consecuencia que sorprende. Sin él, es perfectamente posible que dos pull requests pasen el CI por separado y, al fusionarse ambos, la rama principal quede roja: cada uno se probó contra un estado anterior que no incluía al otro. Con él, hay que actualizar la rama antes de fusionar, y el CI se ejecuta contra el resultado real. El precio es que en un repositorio con mucha actividad genera una carrera de actualizaciones constante; ahí es donde entran las colas de fusión, que resuelven el problema automatizando el reordenamiento.

Un detalle operativo importante: **una comprobación obligatoria que nunca se ejecuta bloquea el pull request para siempre**. Si exiges el check `tests-backend` y el workflow tiene un filtro de rutas que lo salta cuando el PR solo toca el frontend, ese PR queda esperando indefinidamente una comprobación que no va a llegar. La solución es un job "de resumen" que siempre se ejecuta y decide:

```yaml
# Este job SIEMPRE corre; es el que se marca como obligatorio
resumen:
  if: always()
  needs: [tests-backend, tests-frontend]
  runs-on: ubuntu-latest
  steps:
    - name: Fallar si alguna dependencia falló
      if: contains(needs.*.result, 'failure')
      run: exit 1
```

Se exige `resumen` en el ruleset, no los jobs individuales. Los jobs saltados por filtros de ruta devuelven `skipped`, que no es `failure`, así que el resumen pasa; si alguno falla de verdad, el resumen falla.

### Historial lineal

Obliga a que las fusiones no introduzcan commits de merge: hay que usar *squash* o *rebase*. Produce un historial recto, mucho más fácil de leer y de bisecar. Es una decisión de estilo, pero si la tomas, esta regla es lo que la hace real.

### Bloquear force push y borrado

Deberían estar siempre activas en cualquier rama compartida. Un `push --force` sobre una rama que otros han clonado destruye trabajo y produce una tarde entera de confusión; el borrado accidental de la rama principal es raro pero espectacular.

### Firma de commits obligatoria

Exige que los commits estén firmados criptográficamente (GPG, SSH o S/MIME). Sube el listón de la trazabilidad —garantiza que quien dice haber hecho un commit lo hizo— a cambio de que todo el equipo tenga que configurar la firma. En un equipo pequeño y de confianza es opcional; en código sensible o con contribuciones externas, vale la pena.

### Reglas de tag

Muchas veces olvidadas y muy útiles si el despliegue se dispara con un tag. Si publicar la versión `v2.1.0` lanza el pipeline de producción, que nadie pueda mover ni borrar ese tag es tan importante como proteger la rama: un tag movido significa que la etiqueta que dice `v2.1.0` apunta ahora a otro código, y todo tu registro de qué hay desplegado pasa a ser mentira.

## Un ruleset de organización

Aquí está el valor real. Un solo ruleset que protege la rama principal de todos los repositorios:

```bash
gh api -X POST /orgs/acme-store/rulesets --input - <<'JSON'
{
  "name": "Proteger la rama principal",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name":        { "include": ["refs/heads/main"], "exclude": [] },
    "repository_name": { "include": ["~ALL"], "exclude": ["sandbox-*"] }
  },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "required_review_thread_resolution": true
      }
    }
  ],
  "bypass_actors": [
    { "actor_type": "OrganizationAdmin", "bypass_mode": "pull_request" }
  ]
}
JSON
```

Vale la pena leer tres piezas de ese JSON:

- `repository_name.include: ["~ALL"]` con `exclude: ["sandbox-*"]` aplica a todos los repositorios presentes y futuros, salvo los de pruebas. Un repositorio creado el mes que viene nace protegido.
- `non_fast_forward` es el nombre interno de "bloquear force push", y `deletion` de "bloquear borrado".
- `bypass_actors` con `bypass_mode: "pull_request"` significa que los Owners pueden saltarse la regla **solo a través de un pull request**, no con un push directo. Es más restrictivo que el `"always"` habitual y bastante más sensato.

Para comprobar qué reglas aplican de verdad a una rama concreta, sumando repositorio y organización:

```bash
gh api /repos/acme-store/api-pedidos/rules/branches/main
```

Devuelve la lista efectiva. Es la respuesta a "¿está protegida esta rama?" sin tener que deducirla de dos pantallas de configuración distintas.

## Proteger una topología de varias ramas

Un caso realista: un flujo `persona → develop → staging → main`, donde `main` solo debe recibir fusiones desde `staging`. Las reglas cubren gran parte, pero **no existe una regla nativa de "solo desde esta rama de origen"** para fusiones normales (los *deployment branches* de un entorno restringen desde dónde se despliega, no desde dónde se fusiona).

El reparto que funciona:

- En `main`: pull request obligatorio con aprobación, checks obligatorios, sin force push, sin borrado.
- En `staging` y `develop`: checks obligatorios, sin force push.
- La regla "solo desde `staging`" se implementa en el CI, comprobando la procedencia del commit (el patrón está desarrollado en la ficha de [Planes y facturación](Planes-y-facturacion.md)).
- El despliegue se protege con un **entorno** con *deployment branches* limitados y revisores obligatorios, que es donde sí hay control nativo.

Y una advertencia sobre lo anterior: la comprobación en CI **detecta**, no **impide**. El commit ya está en la rama cuando el workflow falla. Sirve para que nadie pueda alegar desconocimiento y para que quede registrado, no para cerrar la puerta. No la documentes como si fuera una protección.

## Buenas prácticas avanzadas

- **Marca como obligatorio un job de resumen, no los jobs individuales.** Es el error que convierte los filtros de rutas de un monorepo en pull requests bloqueados para siempre esperando un check que nunca se ejecutará. Un job con `if: always()` que agrega los resultados de los demás resuelve el problema de raíz y además hace que añadir jobs nuevos no obligue a tocar el ruleset.
- **Activa "descartar aprobaciones obsoletas".** Sin ella, la revisión es teatro: se aprueba un cambio de una línea y después se empuja lo que sea, con la aprobación intacta. Es la forma más silenciosa de que código sin revisar entre en la rama principal por la puerta principal.
- **Usa el modo `evaluate` para introducir reglas en un equipo grande.** Activar una regla estricta de golpe bloquea a gente a mitad de tarea y genera la petición inmediata de desactivarla. Con `evaluate` mides una semana quién habría sido bloqueado y por qué, lo comentas, y activas después con el equipo enterado.
- **Prefiere `bypass_mode: pull_request` a `always` para los Owners.** Un Owner necesita poder desatascar una emergencia, pero no necesita hacerlo con un push directo a las tres de la mañana sin traza. Con `pull_request` la excepción existe, es rápida, y deja constancia revisable.
- **Protege los tags si el despliegue se dispara con un tag.** Es el punto ciego clásico: se blinda `main` con cinco reglas y cualquiera puede mover `v2.1.0` a otro commit, lo que significa desplegar código arbitrario a producción sin tocar ninguna rama protegida. Una regla de tag de dos clics cierra ese hueco.
- **Comprueba la protección con la API, no con la interfaz.** Cuando conviven rulesets de organización, rulesets de repositorio y reglas clásicas, ninguna pantalla te muestra el resultado combinado. `gh api /repos/ORG/REPO/rules/branches/main` sí, y es la única respuesta fiable a la pregunta "¿qué protege ahora mismo esta rama?".

## Recursos didácticos

- [Sobre los rulesets](https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets) — modelo completo, tipos de regla y cómo se acumulan.
- [Reglas disponibles](https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets) — la lista exhaustiva, con qué requiere cada una.
- [Colas de fusión](https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue) — el siguiente paso cuando "requerir rama actualizada" se vuelve insostenible por volumen.

---

*En resumen: un flujo documentado es una convención; un ruleset es un control, y en la organización se configura una vez para todos los repositorios que existen y los que existirán.*
