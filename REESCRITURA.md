# Reescritura al formato de guía completa

Documento de trabajo: plan, checklist y estado de la conversión de todas las colecciones del repositorio del formato antiguo (introducciones breves) al **formato de guía completa**.

No es una ficha de tutorial. Vive en la raíz porque tiene que estar versionado y ser fácil de encontrar entre sesiones. Cuando la conversión termine, este fichero se puede borrar.

**Última actualización:** 27 de julio de 2026.

---

## 1. Qué cambia y por qué

El repositorio se escribió como colección de **introducciones breves**: fichas de 60-120 líneas con un índice de secciones cerrado, pensadas para leerse en pocos minutos y quedarse con "lo mínimo para no perderse".

La decisión nueva es que sean **guías completas**: la ficha sigue empezando por una introducción accesible, pero a partir de ahí desarrolla el tema con la profundidad que pida, con el objetivo de que quien la lea acabe sabiendo usar la tecnología sin tener que buscar lo importante en otro sitio.

Los tres cambios concretos:

| | Antes | Ahora |
|---|---|---|
| Extensión | 60-120 líneas | La que pida el tema (en la práctica, 200-350) |
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
## Recursos didácticos                  ← FIJA. Omitible solo si no hay nada que valga la pena.

---

*En resumen: <una frase memorable>.*  ← FIJA.
```

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
- **Esperar la notificación de fin.** Consultar el tamaño del fichero y deducir que ha acabado produce commits de estados intermedios. Ya pasó una vez y hubo que arreglarlo con dos commits de seguimiento.
- **Contar con errores 529 del API.** En la primera tanda, 6 de 8 agentes terminaron con `529 Overloaded`, todos en el paso final de recortar longitud, después de haber producido un documento completo y coherente. Ante un fallo: verificar estructura (secciones fijas, delimitadores de código pares, cierre presente) antes de asumir que hay que rehacerlo. Uno de los ocho murió antes de escribir nada y se reescribió a mano.

---

## 7. Estado

**Convertidas: 36 fichas en 7 colecciones. Pendientes: 213 fichas en 27 colecciones.**

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

### Pendiente

Orden propuesto: por valor de uso y por dependencias entre colecciones. Las pequeñas primero dentro de cada categoría, porque cerrar una colección entera permite comitear.

| # | Colección | Fichas | Avisos |
|---:|---|---:|---|
| 1 | `bases-de-datos/acceso-a-datos-dotnet` | 2 | ⚠️ Contiene `Dapper.md`, que la skill cita como referencia de tono. Al convertirlo, actualizar esa nota en `SKILL.md`. |
| 2 | `bases-de-datos/caching` | 4 | 5 ficheros con banner 🧭. |
| 3 | `bases-de-datos/migraciones-de-esquema` | 5 | 6 ficheros con banner 🧭. |
| 4 | `seguridad/gestion-de-secretos-en-desarrollo` | 3 | Enlazada desde varias fichas ya convertidas. |
| 5 | `seguridad/algoritmos-de-hash` | 6 | |
| 6 | `seguridad/autenticacion-y-autorizacion` | 10 | 7 ficheros con banner 🧭. |
| 7 | `devops/ci-cd` | 12 | Enlazada desde `docker` y `despliegue-en-vps`. |
| 8 | `devops/git` | 20 | |
| 9 | `arquitectura-de-software/clean-architecture` | 9 | |
| 10 | `arquitectura-de-software/patrones-de-diseno` | 11 | 9 ficheros con banner 🧭. Carpeta renombrada de `patrones-de-diseño` el 27/07/2026. |
| 11 | `arquitectura-de-software/tipos-de-apis` | 13 | |
| 12 | `arquitectura-de-software/multi-tenancy` | 7 | |
| 13 | `lenguajes/csharp-dotnet` (+ 2 subcarpetas) | 8 | Tiene subcarpetas con README propio. |
| 14 | `testing/testing-dotnet` | 9 | Se cruza con `docker/Testcontainers.md`, ya convertida. |
| 15 | `testing/e2e` | 1 | |
| 16 | `desarrollo-web/asp-net-core` | 9 | |
| 17 | `desarrollo-web/de-wpf-a-web` | 16 | |
| 18 | `desarrollo-web/frontend-react` | 28 | ⚠️ La más grande. La skill cita su `README.md` como modelo de índice y `clsx.md` como ejemplo. |
| 19 | `redes/redes-y-acceso-remoto` | 11 | `SSH.md` y `VPN.md` se cruzan con `despliegue-en-vps`, ya convertida. |
| 20 | `odoo/fundamentos` | 4 | |
| 21 | `odoo/busqueda-y-filtros` | 4 | |
| 22 | `odoo/pruebas-seguras` | 5 | |
| 23 | `odoo/configuracion-parametros` | 6 | |
| 24 | `ia/context-engineering` | 9 | |
| 25 | `herramientas/correo-transaccional` | 1 | |

---

## 8. Asuntos abiertos

- **`ia/ingenieria-con-llms/` sin identificar.** Apareció durante la sesión del 27/07/2026 con 9 fichas (~2 500 líneas), ya en formato nuevo. No la encargó ninguno de los subagentes lanzados en esa sesión, que trabajaban en tracing, OpenTelemetry, Docker, ghcr, Testcontainers, PostgreSQL, MongoDB y credenciales. **Está sin comitear**, a la espera de identificar su origen. Si es contenido válido, hay que darle README-índice, enlazarla desde `ia/README.md` y revisar si se solapa con `ia/context-engineering`.
- **Referencias de tono en la skill.** `SKILL.md` cita `bases-de-datos/acceso-a-datos-dotnet/Dapper.md` como referencia de tono y `desarrollo-web/frontend-react/README.md` como modelo de índice. Ambas están en formato antiguo. Cuando se conviertan, conviene apuntar esas referencias a fichas ya convertidas (por ejemplo `devops/despliegue-en-vps/UFW.md` y su README) y quitar la nota de «calibra el tono, no la extensión».
- **Fin de línea.** El repositorio convierte LF a CRLF al indexar, así que tras comitear los ficheros pueden reaparecer como modificados. Es ruido, no un cambio real: se confirma con `git diff --ignore-all-space`.
- **Al terminar la conversión**, revisar el README raíz: ya no describe la colección como «guías introductorias», pero conviene una lectura final de coherencia.
