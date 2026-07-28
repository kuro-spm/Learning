# Reescritura al formato de guía completa

Documento de trabajo: plan, checklist y estado de la conversión de todas las colecciones del repositorio del formato antiguo (introducciones breves) al **formato de guía completa**.

No es una ficha de tutorial. Vive en la raíz porque tiene que estar versionado y ser fácil de encontrar entre sesiones. Cuando la conversión termine, este fichero se puede borrar.

**Última actualización:** 28 de julio de 2026.

---

## 1. Qué cambia y por qué

El repositorio se escribió como colección de **introducciones breves**: fichas de 60-120 líneas con un índice de secciones cerrado, pensadas para leerse en pocos minutos y quedarse con "lo mínimo para no perderse".

La decisión nueva es que sean **guías completas**: la ficha sigue empezando por una introducción accesible, pero a partir de ahí desarrolla el tema con la profundidad que pida, con el objetivo de que quien la lea acabe sabiendo usar la tecnología sin tener que buscar lo importante en otro sitio.

Los tres cambios concretos:

| | Antes | Ahora |
|---|---|---|
| Extensión | 60-120 líneas | La que pida el tema (en la práctica, 300-500) |
| Índice de secciones | Cerrado, sin inventar secciones | Apertura y cierre fijos, desarrollo libre |
| Ejemplos de código | Snippet suelto | Ejemplo **guiado**: introducción → código → qué hace y qué devuelve |

La skill `crear-tutorial` (en `~/.claude/skills/crear-tutorial/SKILL.md`) ya está actualizada con estas reglas y es la referencia normativa. Este documento no la sustituye: organiza su aplicación al contenido que ya existía.

---

## 2. El contrato de formato

Esqueleto obligatorio. Apertura y cierre fijos, desarrollo intermedio libre.

```markdown
# <Título>

## ¿Qué es?                            ← FIJA. Una o dos frases.
## ¿Por qué existe?                    ← FIJA. El problema que resuelve + blockquote de analogía.
## ¿Cuándo y para qué se usa?           ← Recomendada, no obligatoria.

## <las secciones que pida el tema>     ← LIBRES. Tantas como haga falta,
## <...>                                   ordenadas como progresión de aprendizaje.

## Buenas prácticas avanzadas           ← FIJA. 3-6 puntos, específicos y accionables.
## Documentación oficial                ← FIJA desde el 28/07/2026. Fuentes canónicas.
## Recursos didácticos                  ← FIJA. Omitible solo si no hay nada que valga la pena.

---

*En resumen: <una frase memorable>.*  ← FIJA.
```

> **`Documentación oficial` es una sección nueva** (28/07/2026), añadida porque los enlaces canónicos quedaban mezclados con los didácticos y pasaban desapercibidos. La división: `Documentación oficial` son las fuentes canónicas (documentación del proyecto, especificación o RFC, *cheat sheet* normativa, repositorio), de 1 a 4 enlaces y cada uno con una línea que diga qué parte merece la pena y cuándo ir ahí; `Recursos didácticos` es todo lo que ayuda a aprender pero no es la fuente. Un enlace no va en las dos.
>
> **Decisión explícita: no hay retrofit.** La sección se aplica al contenido escrito de aquí en adelante. Las fichas ya convertidas se quedan sin ella; están listadas en la tabla de la sección 7 y se puede hacer una pasada más adelante si se quiere, pero no es un requisito para cerrar la conversión.

### Reglas duras

- **Español** en la prosa. Términos técnicos consolidados, comandos y código en su idioma original.
- **Audiencia: backend junior-medio.** Programa a diario y puede conocer partes del stack, pero es nuevo en la tecnología concreta. No se explica qué es una API; sí se definen los términos propios antes de usarlos.
- **Género neutro** para el lector; masculino genérico solo si el neutro queda forzado.
- **Autónoma.** Sin referencias a proyectos, módulos ni dominios privados. Escenarios genéricos y reconocibles.
- **Todo párrafo que hable de código lleva su ejemplo guiado.** Regla práctica: si un párrafo habla de código y no hay snippet cerca, falta el snippet.
- **Muestra la salida real** cuando aporte (respuesta JSON, log de consola, mensaje de error concreto) y usa contrastes ✅/❌.
- **Prohibido el relleno:** repetir lo ya dicho, parafrasear la documentación oficial, enumerar la API entera sin explicar nada.
- **Sin banners de metacomentario** tipo `> 🧭 Tutorial recomendado de forma proactiva…`. Son contexto del repositorio, no de la tecnología.
- **Nombres de ejemplo genéricos y coherentes** dentro de la misma ficha y, a ser posible, de la misma colección. Nunca `foo`/`bar`.

### Lo que aporta valor de verdad

Lo que diferencia una ficha convertida buena de una simplemente más larga:

- Una **tabla de decisión** para las disyuntivas reales del tema ("¿cola o topic?", "¿incrustar o referenciar?").
- Una sección de **errores frecuentes** con formato síntoma → causa.
- El **cálculo o la salida concreta** que hace tangible un peligro abstracto (la explosión de cardinalidad con números, la media que esconde el p99).
- **Cuándo NO usar** la tecnología.
- Los puntos de «Buenas prácticas avanzadas» pasan el criterio: *¿esto lo sabe solo quien de verdad domina el tema?*

---

## 3. Convenciones transversales

**Ejemplo conductor por colección.** Todas las fichas de una colección comparten escenario y nombres, para que las piezas encajen entre documentos. El que se está usando en `devops/`:

> Tienda online. Pedido **#4711**. Servicios `pedidos`, `pagos`, `inventario`, `emails`. VPS en `203.0.113.10`, dominio `tienda.ejemplo.com`, usuario de despliegue `deployer`.

**READMEs-índice.** Los escribe siempre la persona (o el hilo principal), nunca un subagente: son lo que da coherencia a la colección. Además del orden de lectura, funcionan muy bien con tablas operativas al final (comandos de verificación, dónde se aplica cada cambio, los errores que se pagan caros).

**Enlaces.** Toda ruta relativa debe resolver, mayúsculas incluidas. Verificar antes de comitear.

**Un commit por colección**, con el detalle de qué cobertura se ha añadido, no solo "reescribe X".

---

## 4. Procedimiento por colección

Checklist repetible. Marca una copia de esto por cada colección que abordes.

- [ ] **1. Leer la colección entera** (todas las fichas y su README) para conservar sus decisiones de contenido y su analogía si es buena.
- [ ] **2. Decidir el ejemplo conductor** y anotarlo, para que todas las fichas lo compartan.
- [ ] **3. Esbozar el índice de secciones** de cada ficha antes de redactar. Si un tema es inabarcable, plantear partirlo en varias fichas enlazadas.
- [ ] **4. Repartir en subagentes** (ver sección 6) con el brief completo: contrato de formato + ejemplo conductor + contenido a cubrir + enlaces a verificar.
- [ ] **5. Escribir el README-índice** a mano mientras trabajan los agentes.
- [ ] **6. Esperar la notificación de fin de cada agente.** No mirar el tamaño del fichero y asumir que ha terminado.
- [ ] **7. Verificar** (comandos en la sección 5): estructura, enlaces, banners, fugas de datos privados.
- [ ] **8. Leer una ficha completa** de las que escribió un agente. La verificación automática comprueba forma, no calidad.
- [ ] **9. Actualizar los índices superiores:** README de la categoría y README raíz si la descripción se queda corta.
- [ ] **10. Commit y push** de la colección.
- [ ] **11. Actualizar la tabla de estado** de este documento.

---

## 5. Comandos de verificación

Estructura de todas las fichas de una colección:

```bash
cd ~/GitRepos/Learning
COL=devops/observabilidad
for f in $COL/*.md; do b=$(basename "$f"); [ "$b" = "README.md" ] && continue
  printf "%-40s q:%s p:%s bp:%s r:%s res:%s  %s líneas\n" "$b" \
    "$(grep -c '^## ¿Qué es?$' "$f")" "$(grep -c '^## ¿Por qué existe?$' "$f")" \
    "$(grep -c '^## Buenas prácticas avanzadas$' "$f")" "$(grep -c '^## Recursos didácticos$' "$f")" \
    "$(grep -c '^\*En resumen:' "$f")" "$(wc -l < "$f")"
done
```

Todos los enlaces relativos del repositorio:

```bash
total=0; roto=0
for f in $(find . -name "*.md" -not -path "./.git/*"); do d=$(dirname "$f")
  while read -r l; do [ -z "$l" ] && continue; total=$((total+1))
    [ -e "$d/$l" ] || { roto=$((roto+1)); echo "ROTO ${f#./} -> $l"; }
  done < <(grep -oE '\]\([^)]+\)' "$f" | sed -E 's/^\]\(//; s/\)$//' | grep -vE '^(https?:|#|mailto:)' | sed 's/#.*//')
done
echo "$total enlaces, $roto rotos"
```

Banners de metacomentario pendientes y bloques de código sin cerrar:

```bash
grep -rl '🧭' --include="*.md" .
for f in $(find . -name "*.md" -not -path "./.git/*"); do
  n=$(grep -c '^```' "$f"); [ $((n % 2)) -ne 0 ] && echo "IMPAR $f ($n)"
done
```

---

## 6. Trabajar con subagentes

Lo que funciona, aprendido en la primera tanda:

- **3-4 agentes en paralelo por colección**, uno por ficha. Más agentes no va más rápido y complica la verificación.
- **El brief tiene que ser autosuficiente:** contrato de formato completo, ejemplo conductor, lista de contenido a cubrir, lista de enlaces a verificar con `ls` antes de usarlos, y una ficha ya convertida como referencia de tono.
- **Indicar explícitamente qué NO debe cubrir**, para que no se solape con la ficha vecina.
- **El README lo escribe el hilo principal.** Un agente no ve la colección completa.
- **Esperar la notificación de fin. No hay atajo.** Consultar el fichero y deducir que ha acabado produce commits de estados intermedios. Pasó en la primera tanda y **volvió a pasar el 28/07/2026** con `algoritmos-de-hash/Hashing-En-CSharp.md`: el documento estaba completo y coherente (todas las secciones fijas, delimitadores pares, cierre presente), se verificó a conciencia y se comiteó — y el agente siguió trabajando después con una pasada de compresión de párrafos. Hubo que arreglarlo con un commit de seguimiento. La lección afinada: **verificar la estructura no distingue «terminado» de «casi terminado»**, porque el agente llega a un documento válido antes de dar el último repaso. Solo la notificación lo distingue.
- **Contar con errores 529 del API.** En la primera tanda, 6 de 8 agentes terminaron con `529 Overloaded`, todos en el paso final de recortar longitud, después de haber producido un documento completo y coherente. Ante un fallo: verificar estructura (secciones fijas, delimitadores de código pares, cierre presente) antes de asumir que hay que rehacerlo. Uno de los ocho murió antes de escribir nada y se reescribió a mano.
- **La extensión se va por encima del rango indicado, y suele estar justificada.** En la segunda tanda se pidieron 250-350 líneas y salieron entre 327 y 519, sin relleno: cuando el brief lista 14-16 bloques de contenido y cada uno pide su snippet o su tabla, la aritmética no da para menos. Dos consecuencias prácticas: **el rango del brief funciona como orientación, no como límite**, y si de verdad quieres una ficha corta hay que recortar la lista de contenido, no repetir el número. No pidas a un agente que recorte al final: es justo donde murieron los de la primera tanda.
- **Avisar al agente de que verá cambios ajenos en `git status`.** Con varios agentes en paralelo, cada uno ve los ficheros de los demás como modificados y lo reporta como anomalía. No es un problema, pero ensucia el informe final.

---

## 7. Estado

**Convertidas: 65 fichas en 13 colecciones. Pendientes: 198 fichas en 21 colecciones.**

Todas las fichas convertidas salvo `Entity-Framework-Core.md` son anteriores a la sección `Documentación oficial` y no la tienen (ver la nota de la sección 2: no hay retrofit).

### Hecho

| Colección | Fichas | Notas |
|---|---:|---|
| `devops/despliegue-en-vps` | 14 | Creada ya en formato nuevo. Define el ejemplo conductor de `devops/`. |
| `devops/mensajeria-asincrona` | 5 | Código de RabbitMQ actualizado a la API asíncrona 7.x. |
| `devops/observabilidad` | 11 | Añadido `¿Qué es?` a `ILogger-T` y `log4net`; y BBPP + Recursos a `log4net`. |
| `devops/docker` | 3 | El README delimita la frontera con `despliegue-en-vps`. |
| `bases-de-datos/mongodb` | 1 | |
| `bases-de-datos/postgresql` | 1 | |
| `seguridad/secretos-en-llamadas-salientes` | 1 | |
| `devops/github-organizaciones` | 5 | Creada ya en formato nuevo, en una sesión aparte. |
| `ia/ingenieria-con-llms` | 9 | Creada ya en formato nuevo. Le falta README-índice (ver sección 8). |
| `bases-de-datos/acceso-a-datos-dotnet` | 3 | Al convertir `Dapper.md` se actualizaron las referencias de tono de `SKILL.md`. `Entity-Framework-Core.md` se **movió aquí** desde `desarrollo-web/de-wpf-a-web` el 28/07/2026: es una tecnología de acceso a datos y la colección no estaba completa sin el ORM. Es la única ficha con la sección `Documentación oficial`. |
| `seguridad/gestion-de-secretos-en-desarrollo` | 3 | |
| `bases-de-datos/caching` | 4 | |
| `bases-de-datos/migraciones-de-esquema` | 5 | |

### Pendiente

Orden propuesto: por valor de uso y por dependencias entre colecciones. Las pequeñas primero dentro de cada categoría, porque cerrar una colección entera permite comitear.

| # | Colección | Fichas | Avisos |
|---:|---|---:|---|
| 1 | `seguridad/algoritmos-de-hash` | 6 | |
| 2 | `seguridad/autenticacion-y-autorizacion` | 10 | 7 ficheros con banner 🧭. |
| 3 | `devops/ci-cd` | 12 | Enlazada desde `docker` y `despliegue-en-vps`. |
| 4 | `devops/git` | 20 | |
| 5 | `arquitectura-de-software/clean-architecture` | 9 | |
| 6 | `arquitectura-de-software/patrones-de-diseno` | 11 | 9 ficheros con banner 🧭. Carpeta renombrada de `patrones-de-diseño` el 27/07/2026. |
| 7 | `arquitectura-de-software/tipos-de-apis` | 13 | |
| 8 | `arquitectura-de-software/multi-tenancy` | 7 | |
| 9 | `lenguajes/csharp-dotnet` (+ 2 subcarpetas) | 8 | Tiene subcarpetas con README propio. |
| 10 | `testing/testing-dotnet` | 9 | Se cruza con `docker/Testcontainers.md`, ya convertida. |
| 11 | `testing/e2e` | 1 | |
| 12 | `desarrollo-web/asp-net-core` | 9 | |
| 13 | `desarrollo-web/de-wpf-a-web` | 15 | `Entity-Framework-Core.md` salió de aquí a `bases-de-datos/acceso-a-datos-dotnet` el 28/07/2026; el README la enlaza en su nueva ubicación. |
| 14 | `desarrollo-web/frontend-react` | 28 | ⚠️ La más grande. La skill citaba su `README.md` como modelo de índice y `clsx.md` como ejemplo; ya no (ver sección 8). |
| 15 | `redes/redes-y-acceso-remoto` | 11 | `SSH.md` y `VPN.md` se cruzan con `despliegue-en-vps`, ya convertida. |
| 16 | `odoo/fundamentos` | 4 | |
| 17 | `odoo/busqueda-y-filtros` | 4 | |
| 18 | `odoo/pruebas-seguras` | 5 | |
| 19 | `odoo/configuracion-parametros` | 6 | |
| 20 | `ia/context-engineering` | 9 | Revisar solapamiento con `ia/ingenieria-con-llms`, ya en formato nuevo. |
| 21 | `herramientas/correo-transaccional` | 1 | |

---

## 8. Asuntos abiertos

- **`ia/ingenieria-con-llms/` ya comiteada** (commit `34709c1`). Queda pendiente: darle README-índice, enlazarla desde `ia/README.md`, revisar si se solapa con `ia/context-engineering` y actualizar la descripción de la categoría en el README raíz.
- **Referencias de tono en la skill: resueltas** el 28/07/2026. `SKILL.md` cita ahora `devops/despliegue-en-vps/UFW.md` (tono), `bases-de-datos/acceso-a-datos-dotnet/Dapper.md` (guía completa con tablas de decisión y errores frecuentes) y `devops/despliegue-en-vps/README.md` (modelo de índice), las tres en formato nuevo. La nota de «calibra el tono, no la extensión» se sustituyó por un aviso de que en el repositorio aún queda contenido en formato antiguo y no sirve como referencia de profundidad. Ese aviso se puede borrar cuando la conversión termine.
- **Fin de línea.** El repositorio convierte LF a CRLF al indexar, así que tras comitear los ficheros pueden reaparecer como modificados. Es ruido, no un cambio real: se confirma con `git diff --ignore-all-space`.
- **Al terminar la conversión**, revisar el README raíz: ya no describe la colección como «guías introductorias», pero conviene una lectura final de coherencia.
