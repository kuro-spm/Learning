# Fusión y conflictos (merge)

## ¿Qué es?

Fusionar (*merge*) es combinar los cambios de una rama en otra. Un **conflicto** ocurre cuando Git no puede combinarlos solo porque dos ramas modificaron la misma parte de un archivo de formas distintas.

## ¿Por qué existe?

Las ramas sirven para trabajar por separado, pero en algún momento ese trabajo tiene que volver a juntarse. La fusión toma lo que hiciste en una rama y lo incorpora a otra (normalmente a `main`).

La mayoría de las veces Git combina todo automáticamente. Pero si tú cambiaste la línea 10 de un archivo y otra persona también, Git no puede adivinar cuál es la buena: ahí surge el conflicto, y decides tú.

> Imagina dos personas editando el mismo párrafo de un documento compartido. Si tocan párrafos distintos, se mezcla solo. Si tocan exactamente la misma frase, alguien tiene que decidir con qué versión quedarse. Eso es un conflicto.

## ¿Cuándo y para qué se usa?

Cuando terminas una rama (por ejemplo, `funcion-login`) y quieres llevar sus cambios a la rama principal. Te sitúas en la rama que **recibe** los cambios y fusionas la otra dentro.

## Lo mínimo que necesitas saber

**1. Fusionar una rama en la actual**

```bash
git switch main          # sitúate en la rama que recibe
git merge funcion-login  # trae los cambios de la otra
```

**2. Cuando hay conflicto, Git te avisa**

```bash
# CONFLICT (content): Merge conflict in login.html
```

El archivo en conflicto mostrará ambas versiones marcadas así:

```
<<<<<<< HEAD
<h1>Iniciar sesión</h1>
=======
<h1>Acceder a tu cuenta</h1>
>>>>>>> funcion-login
```

**3. Resolver el conflicto a mano**

Edita el archivo y déjalo como debe quedar, borrando las marcas `<<<<<<<`, `=======` y `>>>>>>>`. Luego confirma la resolución:

```bash
git add login.html
git commit            # cierra la fusión
```

**4. Comprobar qué está en conflicto**

```bash
git status
```

Te lista los archivos pendientes de resolver.

## Lo que NO hace

- **No decide por ti en un conflicto** — Git marca el problema, pero la elección es humana.
- **No siempre genera conflictos** — son la excepción, no la norma; la mayoría de fusiones son automáticas.
- **No borra la rama fusionada** — sigue ahí hasta que la elimines tú (`git branch -d`).

---

*En resumen: fusionar es reunir el trabajo separado de las ramas, y un conflicto es solo Git pidiéndote que decidas tú cuando dos cambios chocan en la misma línea.*
