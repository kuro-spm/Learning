# Remotos (push, pull, fetch)

## ¿Qué es?

Un *remoto* es una copia del repositorio alojada en otro sitio, normalmente un servidor en internet como GitHub o GitLab. Sincronizar con él te permite compartir tu trabajo y recibir el de los demás.

## ¿Por qué existe?

Git es distribuido: cada quien tiene el repo completo en su ordenador. Pero para colaborar (o tener una copia a salvo fuera de tu máquina) hace falta un punto común al que todos suban y del que todos bajen. Ese punto es el remoto.

> Piensa en el remoto como una carpeta compartida en la nube para tu proyecto: cada persona trabaja en su copia local y luego sube y baja cambios del lugar común.

## ¿Cuándo y para qué se usa?

- Para **publicar** tu proyecto y que otros puedan verlo o colaborar.
- Para **trabajar en equipo**: tú subes tus commits, tus compañeros bajan los suyos.
- Para tener una **copia de seguridad** del historial fuera de tu ordenador.

El remoto por defecto suele llamarse `origin`.

## Lo mínimo que necesitas saber

**1. Conectar tu repo local con un remoto**

```bash
git remote add origin https://github.com/usuario/mi-blog.git
```

(Si clonaste el proyecto con `git clone`, esto ya está hecho.)

**2. Subir tus commits (`push`)**

```bash
git push origin main
```

Envía los commits de tu rama `main` al remoto. La primera vez puedes usar `git push -u origin main` para vincularlas y luego basta con `git push`.

**3. Bajar y combinar los cambios de los demás (`pull`)**

```bash
git pull
```

Trae los commits nuevos del remoto y los fusiona en tu rama actual. Hazlo a menudo para no quedarte desactualizado.

**4. Solo mirar, sin combinar (`fetch`)**

```bash
git fetch
```

Descarga los cambios del remoto pero **no** los fusiona. Útil para revisar qué hay de nuevo antes de integrarlo.

## Lo que NO hace

- **`push` no sube lo que no has commiteado** — solo viajan los commits, no los cambios sueltos.
- **`pull` puede provocar conflictos** — si tú y otra persona tocasteis la misma línea (ver [Fusión y conflictos](Fusion-y-conflictos.md)).
- **El remoto no es Git** — es solo un lugar donde vive una copia; toda la potencia sigue estando en tu Git local.

---

*En resumen: los remotos son el punto de encuentro de tu proyecto — subes (`push`) lo tuyo y bajas (`pull`) lo de los demás para que todas las copias hablen el mismo idioma.*
