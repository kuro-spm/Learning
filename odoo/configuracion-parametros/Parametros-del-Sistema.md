# Parámetros del Sistema (ir.config_parameter)

## ¿Qué es?

Los parámetros del sistema son un almacén de **clave-valor** guardado en la base de datos, en el modelo `ir.config_parameter`. Cada parámetro es una pareja de textos (`clave` → `valor`) que cualquier parte del código puede leer o escribir en tiempo de ejecución, sin tocar el servidor ni reiniciar nada.

## ¿Por qué existe?

Muchos valores no son código ni infraestructura, pero tampoco deberían estar clavados en el programa: la URL de una API externa, un límite configurable, la clave de un servicio, un interruptor para activar o desactivar una funcionalidad. Si se escriben directamente en el `.py`, cambiar uno obliga a modificar código y volver a desplegar. `ir.config_parameter` es el sitio donde poner esos valores para poder cambiarlos en caliente.

> Piensa en él como una tabla de dos columnas —`clave` y `valor`— compartida por todo Odoo: una especie de diccionario global persistente que sobrevive a los reinicios.

## ¿Cuándo y para qué se usa?

Cada vez que un módulo necesita un valor **global a la base de datos** que pueda cambiar sin redeploy: la dirección de correo a la que enviar un informe, el *token* de una integración, el número de días de un plazo, o un *feature flag* del tipo "activar el nuevo cálculo". Odoo mismo lo usa: `web.base.url` (la URL base de la instancia) es un parámetro del sistema.

## Lo mínimo que necesitas saber

**1. Leer un parámetro**

```python
ICP = self.env["ir.config_parameter"].sudo()
url_api = ICP.get_param("mi_modulo.url_api", default="https://api.ejemplo.com")
```

Si la clave no existe, devuelve el `default` (o `False` si no se indica).

**2. Escribir un parámetro**

```python
self.env["ir.config_parameter"].sudo().set_param("mi_modulo.url_api", "https://nueva-api.com")
```

**3. Dónde verlos y editarlos a mano**

Con el modo desarrollador activo: *Ajustes → Técnico → Parámetros del sistema*. Ahí aparece la lista completa y se pueden crear o editar directamente.

**4. Definir un valor inicial por XML**

Un módulo puede dejar un parámetro creado al instalarse, con un fichero de datos:

```xml
<record id="param_url_api" model="ir.config_parameter">
    <field name="key">mi_modulo.url_api</field>
    <field name="value">https://api.ejemplo.com</field>
</record>
```

## Lo que NO hace

- **No guarda tipos** — todo se almacena como texto. Un `"5"` o un `"True"` hay que **castearlo** al leerlo (`int(...)`, comparar con `"True"`, etc.).
- **No es por usuario ni por compañía** — el valor es único para toda la base de datos. Para valores por compañía, mira [Configuración por compañía](Configuracion-por-Compania.md).
- **No cifra nada** — el valor se guarda en claro en la BD. Un secreto muy sensible no debería vivir aquí sin más protección.
- **No tiene interfaz amable por sí solo** — la lista técnica es para administradores; para exponerlo a usuarios de negocio se usa la pantalla de [Ajustes](Ajustes.md).

## Buenas prácticas avanzadas

- **Prefija siempre la clave con el nombre del módulo** — `mi_modulo.url_api`, no `url_api`. El almacén es global y compartido; sin prefijo, dos módulos pueden pisarse la misma clave sin darse cuenta.
- **Usa `sudo()` casi siempre** — el acceso de lectura/escritura a `ir.config_parameter` está restringido a administradores. Un usuario normal que dispare un código que lea el parámetro provocaría un error de permisos si no se usa `sudo()`.
- **Castea de forma defensiva** — como todo es texto, lee con un valor por defecto y conviértelo con cuidado: `int(ICP.get_param("mi_modulo.dias", "0"))`. Así un parámetro vacío o mal escrito no revienta el proceso.
- **No metas secretos de alto riesgo en claro** — para claves de API muy sensibles, valora cifrarlas o guardarlas fuera de la BD (variables de entorno leídas desde `odoo.conf`), porque cualquiera con acceso a la base de datos las vería.

---

*En resumen: `ir.config_parameter` es el diccionario global y persistente de Odoo para valores que quieres cambiar en caliente sin tocar código —recuerda prefijar la clave, usar `sudo()` y castear, porque todo se guarda como texto.*
