# Odoo

## ¿Qué es?

Odoo es un **ERP** (sistema de gestión empresarial) de código abierto y modular: una única plataforma que reúne aplicaciones para ventas, facturación, inventario, contabilidad, RRHH, CRM y muchas más. Está escrito en Python, guarda los datos en una base de datos PostgreSQL y se usa desde el navegador.

## ¿Por qué existe?

Una empresa necesita muchas herramientas distintas: una para vender, otra para facturar, otra para el almacén, otra para las nóminas... Si cada una es un programa separado, los datos no hablan entre sí y hay que copiarlos a mano de una a otra.

Odoo resuelve esto poniendo **todas esas aplicaciones sobre la misma base de datos**. Cuando vendes un producto, el stock baja solo y la factura se genera sola, porque ventas, inventario y contabilidad comparten la misma información.

> Si ya conoces el mundo web, piensa en Odoo como "un WordPress de la gestión empresarial": un núcleo pequeño al que le enchufas módulos (*apps*) para añadir funciones.

## ¿Cuándo y para qué se usa?

Aparece siempre que una organización quiere centralizar su operativa en un solo sitio. Ejemplos genéricos:

- Una **tienda online** que gestiona catálogo, pedidos, stock y facturas desde un mismo lugar.
- Una **empresa de servicios** que lleva proyectos, partes de horas y facturación de clientes.
- Un **taller o fábrica** que controla órdenes de trabajo, materiales y compras a proveedores.

En todos los casos, la ventaja es la misma: un único dato compartido en lugar de islas de información desconectadas.

## Lo mínimo que necesitas saber

**1. La arquitectura en tres piezas**

```
Navegador  ──►  Servidor Odoo (Python)  ──►  Base de datos (PostgreSQL)
```

Tú usas Odoo desde el navegador. El servidor contiene la lógica (Python) y guarda todo en PostgreSQL. Nunca deberías tocar PostgreSQL "a mano": Odoo espera ser el único que escribe en su base de datos.

**2. Todo son módulos (apps)**

La funcionalidad se instala como módulos independientes: `sale` (ventas), `stock` (inventario), `account` (contabilidad), `hr` (recursos humanos)... Activas solo los que necesitas.

**3. Modelos y campos (la forma de los datos)**

Cada tipo de dato es un *modelo*, que por dentro es una tabla. Por ejemplo, un cliente es un registro del modelo `res.partner`; un pedido, del modelo `sale.order`. Cada modelo tiene campos (nombre, correo, importe...).

**4. Community vs Enterprise**

Hay dos ediciones: **Community** (gratuita y open source) y **Enterprise** (de pago, con módulos extra y soporte). Además, cada año sale una versión nueva (Odoo 16, 17, 18...).

**5. Se automatiza y se integra por API**

Además de la web, Odoo expone una API (XML-RPC / JSON-RPC) para que otros programas lean o escriban datos. Así se conecta, por ejemplo, con una web externa o con un sistema de terceros.

```python
# Leer clientes desde fuera de Odoo mediante la API
import xmlrpc.client
models = xmlrpc.client.ServerProxy("https://miempresa.odoo.com/xmlrpc/2/object")
clientes = models.execute_kw(db, uid, password,
    "res.partner", "search_read",
    [[["customer_rank", ">", 0]]], {"fields": ["name", "email"]})
```

## Lo que NO hace

- **No es solo una web** — es un sistema de gestión completo; la web es solo la puerta de entrada.
- **No se personaliza tocando PostgreSQL** — los cambios se hacen desde la interfaz o desarrollando módulos, nunca con `UPDATE` directos a las tablas.
- **No es "instalar y olvidarse"** — cada versión anual puede cambiar cosas, y actualizar de una a otra requiere planificación.

---

*En resumen: Odoo es una caja de aplicaciones de gestión que comparten una misma base de datos, de modo que la información fluye sola entre ventas, almacén, contabilidad y todo lo demás.*
