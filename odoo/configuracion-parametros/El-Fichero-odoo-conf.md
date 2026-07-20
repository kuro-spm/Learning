# El fichero odoo.conf

## ¿Qué es?

`odoo.conf` es el fichero de configuración del **servidor** de Odoo: un archivo de texto en formato `ini` donde se declara todo lo que Odoo necesita saber antes de arrancar (dónde está la base de datos, en qué carpetas buscar los módulos, en qué puerto escuchar, cuántos procesos levantar…). Odoo lo lee **una sola vez, al iniciarse**.

## ¿Por qué existe?

Odoo tiene que conocer ciertos datos antes de poder funcionar, y esos datos dependen de la máquina en la que corre, no de la lógica de negocio: la contraseña de PostgreSQL en este servidor, la ruta de los addons en este disco, cuántos workers aguanta esta CPU. Meter eso en el código sería absurdo, porque cambia de un servidor a otro. `odoo.conf` es el sitio donde vive esa configuración de infraestructura, separada del programa.

> Si vienes de .NET, piensa en `odoo.conf` como el `appsettings.json` (más el fichero de arranque del servicio): parámetros de entorno que la aplicación lee al iniciarse, no lógica.

## ¿Cuándo y para qué se usa?

Se toca al **montar o mantener un servidor** de Odoo: la primera instalación, cuando se añade una carpeta de módulos nueva, cuando hay que apuntar a otra base de datos, o cuando se ajusta el rendimiento (número de procesos y límites de memoria). Un desarrollador de módulos lo edita poco; quien administra el despliegue, a menudo.

## Lo mínimo que necesitas saber

**1. Estructura básica**

Es un `ini` con una sección `[options]` y pares `clave = valor`:

```ini
[options]
admin_passwd = una-contraseña-larga-y-secreta
db_host = localhost
db_port = 5432
db_user = odoo
db_password = ****
addons_path = /opt/odoo/odoo/addons,/opt/odoo/custom-addons
http_port = 8069
```

**2. Los parámetros que más se tocan**

- `addons_path` — lista de carpetas (separadas por comas) donde Odoo busca módulos. Si un módulo no aparece en Odoo, casi siempre es que su carpeta no está aquí.
- `db_host` / `db_user` / `db_password` — cómo conectar con PostgreSQL.
- `admin_passwd` — la *master password* que protege el gestor de bases de datos (crear, duplicar, borrar BD).
- `http_port` — el puerto donde escucha (8069 por defecto).
- `workers` — número de procesos para atender peticiones (0 = modo mono-proceso, solo para desarrollo).
- `dbfilter` — restringe qué bases de datos se muestran según el dominio de acceso.

**3. Cómo se le indica a Odoo qué fichero usar**

```bash
odoo -c /etc/odoo/odoo.conf
```

Cualquier parámetro del fichero se puede sobreescribir en la línea de comandos (`--http-port=8070`), lo que resulta útil para levantar un entorno de test con una variante puntual.

**4. Leerlo desde Python (solo lectura)**

Dentro de un módulo, se puede consultar la configuración del servidor así:

```python
from odoo.tools import config

data_dir = config.get("data_dir")
esta_en_test = config.get("test_enable")
```

## Lo que NO hace

- **No se edita desde la interfaz** — es un fichero del sistema; para cambiarlo hay que entrar al servidor.
- **No se recarga en caliente** — cualquier cambio exige **reiniciar** el servicio de Odoo.
- **No guarda configuración de negocio** — plazos de pago, claves de una integración o textos por defecto no van aquí; para eso están los parámetros del sistema y los ajustes.
- **No es por compañía ni por usuario** — es global a toda la instancia del servidor.

## Buenas prácticas avanzadas

- **`admin_passwd` fuerte y `list_db = False` en producción** — la *master password* permite borrar cualquier base de datos desde el navegador. Ponla larga y, en producción, desactiva el listado de BD (`list_db = False`) para que ni siquiera aparezca el gestor.
- **No versiones el fichero con credenciales dentro** — el `odoo.conf` real vive en el servidor y queda fuera del repositorio; en el repo se guarda como mucho una *plantilla* con los valores sensibles en blanco. Así las contraseñas no acaban en el historial de Git.
- **`workers` se calcula, no se improvisa** — una regla de partida habitual es `(nº de CPUs * 2) + 1`. Con `workers = 0` Odoo funciona en un solo proceso: vale para desarrollar, pero en producción deja el servidor incapaz de atender peticiones en paralelo o de ejecutar los crons largos sin bloquearse.
- **`dbfilter` evita el zoo de bases de datos** — en un servidor con varias BD, un `dbfilter = ^%d$` (o basado en el subdominio) hace que cada dominio muestre solo la suya, en lugar de ofrecer al usuario una lista donde podría entrar en la que no toca.

## Recursos didácticos

- Documentación oficial de la configuración de despliegue de Odoo: <https://www.odoo.com/documentation/18.0/administration/on_premise/deploy.html>

---

*En resumen: `odoo.conf` es la configuración de infraestructura que el servidor lee al arrancar —dónde está todo y cuánto puede tirar—, separada del código y de la configuración de negocio.*
