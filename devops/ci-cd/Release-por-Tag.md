# Release por tag

## ¿Qué es?

Una estrategia de despliegue en la que **subir código a una rama no publica nada**: el deploy a producción se dispara únicamente cuando alguien crea un *tag* de versión (`v1.2.0`) sobre la rama principal. El tag es el botón de publicar.

## ¿Por qué existe?

La alternativa habitual es desplegar "cuando algo llega a `main`". El problema: un merge puede hacerse por muchos motivos (consolidar trabajo, preparar terreno, arreglar un conflicto) y no todos significan "quiero esto en producción ya". Cuando el merge y el release son la misma cosa, publicar deja de ser una decisión y pasa a ser un efecto secundario.

Separarlos convierte el release en un **acto deliberado**: alguien mira el estado de la rama, decide que ese commit exacto es la versión 1.2.0, y lo etiqueta.

> Piensa en la diferencia entre guardar el borrador de un artículo (merge) y pulsar "Publicar" (tag). Puedes guardar cien veces; publicas cuando tú lo decides.

## ¿Cuándo y para qué se usa?

- Cuando el equipo quiere **control humano** sobre el momento exacto de publicar (por ejemplo, no desplegar un viernes por la tarde).
- Cuando se necesita un **historial de versiones legible**: `git tag` lista todas las releases, y cada deploy corresponde a una versión con nombre.
- Cuando conviene poder **volver atrás con precisión**: re-desplegar la versión anterior es apuntar al tag anterior, sin buscar commits a mano.

Los tags suelen seguir **versionado semántico** (*semver*): `vMAYOR.MENOR.PARCHE` — los básicos de qué es un tag en Git están en [Tags y versiones](../git/Tags-y-versiones.md).

## Lo mínimo que necesitas saber

**1. El workflow se dispara con el tag, no con la rama**

```yaml
# .github/workflows/release.yml
on:
  push:
    tags: ['v*']   # v1.0.0, v2.3.1... cualquier tag que empiece por v
```

**2. Crear una release es crear y subir un tag**

```bash
git checkout main
git tag v1.2.0
git push origin v1.2.0   # <- esto (y solo esto) lanza el deploy
```

**3. La imagen desplegada queda ligada a su versión**

```yaml
- uses: docker/build-push-action@v6
  with:
    push: true
    tags: |
      ghcr.io/mi-org/tienda-backend:v1.2.0   # trazable: imagen <-> versión
      ghcr.io/mi-org/tienda-backend:latest
```

**4. Volver atrás = re-desplegar el tag anterior**

```bash
# el pipeline permite relanzar el deploy de cualquier versión etiquetada
git push origin v1.1.9 --force-with-lease   # o relanzar el workflow del tag v1.1.9
```

## Lo que NO hace

- **No valida el código** — la validación ocurrió antes (gates de PR, suite E2E); el tag asume que etiquetas algo ya verificado. Etiquetar un commit roto lo despliega roto.
- **No impide tags equivocados** — cualquiera con permisos puede etiquetar; suele protegerse limitando quién puede crear tags `v*`.
- **No decide la numeración** — elegir si es `1.3.0` o `2.0.0` (¿rompe compatibilidad?) sigue siendo criterio humano o de herramientas de *semantic release*.

## Buenas prácticas avanzadas

- **Usa tags anotados, no ligeros** — `git tag v1.2.0` crea un tag "ligero": un simple puntero sin autor, fecha ni mensaje. `git tag -a v1.2.0 -m "Checkout con nueva pasarela de pago"` crea un objeto completo con metadatos, que es lo que esperan las herramientas de release (`git describe`, por ejemplo, ignora los ligeros por defecto). Para una versión publicada, el anotado es el correcto siempre.
- **Un tag publicado no se mueve jamás** — técnicamente puedes borrar `v1.2.0` y volver a crearlo sobre otro commit (`git tag -f`), pero entonces existen dos artefactos distintos que dicen ser la misma versión: la imagen que alguien ya descargó y la nueva. Registros, cachés y máquinas de compañeros quedan mintiendo. Si el release salió mal, publica `v1.2.1`; nunca reescribas una versión que ya vio el mundo.
- **Protege el patrón `v*` en la plataforma** — el tag es el botón de producción, así que configúralo como *protected tag* (GitLab) o con *rulesets* (GitHub) para que solo ciertos roles puedan crearlo. Sin esto, cualquier persona con permiso de push puede desplegar a producción sin querer con un simple `git push --tags`.
- **Que el workflow del tag verifique, no confíe** — como el tag no valida nada, un buen pipeline de release comprueba antes de desplegar que el commit etiquetado pasó los checks de CI (o relanza la suite completa). Cuesta unos minutos y elimina la clase entera de accidentes "etiqueté la rama equivocada".
- **Etiqueta la imagen también con el SHA del commit** — además de `tienda-backend:v1.2.0`, publica `tienda-backend:sha-8f4b7f8`. La versión es para humanos; el SHA responde sin ambigüedad la pregunta que llega tarde o temprano: "¿qué código exacto está corriendo ahora mismo en producción?".

---

*En resumen: release por tag separa "integrar código" de "publicar producto" — el deploy deja de ser un efecto secundario y vuelve a ser una decisión.*
