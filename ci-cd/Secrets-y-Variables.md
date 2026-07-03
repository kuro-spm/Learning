# Secrets y variables de entorno

**¿Qué es?** Son los mecanismos para pasarle a un pipeline información que cambia según el entorno (*variables de entorno*) o que es sensible y no debe verse (*secrets*): contraseñas, tokens de API, claves de acceso, etc.

---

## ¿Por qué existe?

Un pipeline necesita datos para hacer su trabajo: la URL del servidor, el token para publicar, la contraseña de la base de datos. Pero **escribir esos datos directamente en el archivo del pipeline es un error grave**: ese archivo está en Git, lo ve todo el equipo y queda en el historial para siempre. Los secrets y las variables permiten usar esos datos sin escribirlos en el código.

> Es la diferencia entre dejar la llave de casa pegada con celo en la puerta, o guardarla en una caja fuerte y pedirla solo cuando hace falta.

---

## ¿Cuándo y para qué se usa?

Siempre que el pipeline necesite algo que no debería estar escrito en el repositorio:

- El token para desplegar una **tienda online** en su servidor.
- La cadena de conexión a la base de datos de una **app de tareas**.
- La clave de API de un servicio externo (correo, pagos, notificaciones) en un **blog**.

---

## Lo mínimo que necesitas saber

**1. Variable de entorno vs. secret**

Una **variable** guarda un valor no sensible (la rama, una URL pública). Un **secret** guarda algo confidencial y la herramienta lo oculta incluso en los logs.

**2. Se guardan fuera del código**

Los secrets se configuran en la propia plataforma de CI/CD (ajustes del repositorio o del proyecto), nunca en el archivo del pipeline.

**3. Se leen como variables**

Dentro del pipeline se usan por su nombre, sin que el valor real aparezca nunca.

```yaml
# GitHub Actions
- run: ./deploy.sh
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
```

```yaml
# GitLab CI: la variable se define en Settings y se usa directamente
script:
  - curl -H "Authorization: $API_TOKEN" https://api.ejemplo.com/deploy
```

**4. La regla de oro**

Si algo es una contraseña, un token o una clave, **nunca** va en un archivo del repositorio (ni siquiera "temporalmente"). Va siempre como secret.

---

## Lo que NO hace

- **No cifran tu aplicación** — protegen los datos sensibles *dentro del pipeline*, no el código que despliegas.
- **No se imprimen en los logs** — un secret bien configurado aparece enmascarado (`***`) si alguien intenta mostrarlo.
- **No sustituyen a un gestor de secretos completo** — para entornos grandes existen herramientas dedicadas (como Vault), pero para empezar, los secrets de la plataforma bastan.

---

## Buenas prácticas avanzadas

- **Un secret filtrado se rota, no se borra** — si un token llegó a un commit, sigue en el historial de Git para siempre aunque lo elimines del archivo: la única respuesta correcta es invalidarlo y generar uno nuevo. Por eso los equipos expertos eligen credenciales fáciles de rotar (tokens revocables, claves por servicio) en lugar de una contraseña maestra que da pánico cambiar.
- **El enmascarado es cosmético, no una barrera** — la plataforma oculta el valor exacto en los logs, pero cualquier transformación se le escapa: un `echo $TOKEN | base64` lo imprime a la vista. Y un secret pasado como argumento de línea de comandos puede acabar en la lista de procesos o en mensajes de error; pásalo siempre como variable de entorno o por fichero.
- **Delimita cada secret a su entorno y sus ramas** — el token de producción no debe existir en los pipelines que cualquiera dispara desde una rama o un fork: usa los mecanismos de la plataforma (*environments* en GitHub, variables *protected* en GitLab) para que solo los workflows de despliegue real puedan leerlo. Un secret visible desde cualquier PR es un secret público con pasos intermedios.

---

*En resumen: los secrets y las variables de entorno son la caja fuerte del pipeline — le dan acceso a contraseñas y tokens sin que esos datos lleguen jamás a quedar escritos en tu repositorio.*
