# Entornos: desarrollo, staging y producción

## ¿Qué es?

Un **entorno** es una copia independiente del sistema donde trabajar sin afectar a las demás. En Odoo se suele hablar de tres: **producción** (el que usan las personas reales cada día), **staging** (una copia casi idéntica para ensayar cambios) y **desarrollo** (donde se programa y se rompe sin miedo). Cada uno tiene su propia base de datos.

## ¿Por qué existe?

Si solo existiera producción, cualquier prueba —instalar un módulo, cambiar una configuración, importar datos— se haría directamente sobre los datos reales de la empresa. Un error ahí puede facturar mal, borrar clientes o dejar el sistema caído mientras la gente intenta trabajar.

La solución es **ensayar primero en un sitio que no importa**. Los entornos separados dan un lugar seguro donde equivocarse: si algo explota en staging, no pasa nada; lo arreglas y solo entonces lo llevas a producción.

> Es la misma idea que ensayar una obra de teatro antes del estreno: nadie prueba la función por primera vez delante del público que ha pagado la entrada.

## ¿Cuándo y para qué se usa?

- Antes de **actualizar Odoo** a una versión nueva: se prueba en staging para ver qué se rompe.
- Antes de **instalar o modificar un módulo**: se comprueba en una copia que todo sigue funcionando.
- Para **formar** a personas nuevas o **hacer demostraciones** sin tocar datos reales.
- Para **desarrollar** funcionalidad a medida sin que los datos de prueba contaminen la contabilidad real.

## Lo mínimo que necesitas saber

**1. Los tres entornos y su propósito**

| Entorno | Para qué | ¿Datos reales? |
|---|---|---|
| Desarrollo | Programar y experimentar | No (datos de prueba) |
| Staging | Ensayar cambios antes del estreno | Copia de los reales, neutralizada |
| Producción | El uso diario real | Sí, los de verdad |

**2. El flujo va siempre en una dirección**

```
Desarrollo  ──►  Staging  ──►  Producción
   (creas)      (ensayas)       (publicas)
```

Los cambios suben de desarrollo a producción, nunca al revés. Lo que sí baja (copiándose) son los datos: producción → staging, para ensayar sobre información realista.

**3. Staging usa datos copiados y "neutralizados"**

Para que las pruebas sean realistas, staging parte de una copia de los datos de producción. Pero esa copia se **neutraliza**: se desactivan los correos, los pagos y las conexiones externas para que un ensayo no mande facturas de verdad a clientes de verdad (ver [Duplicar y neutralizar la base de datos](Duplicar-y-Neutralizar-Base-de-Datos.md)).

**4. En Odoo.com y Odoo.sh esto viene de serie**

Si usas la versión alojada por Odoo (**Odoo.sh**), crear un entorno de staging es un botón: clona producción y la neutraliza automáticamente. En una instalación propia (*on-premise*) tendrás que montarlo tú duplicando la base de datos.

## Lo que NO hace

- **Un entorno separado no se sincroniza solo** — si cambias algo en staging, no aparece en producción; hay que volver a aplicarlo.
- **Staging no sustituye a las copias de seguridad** — es para ensayar, no para recuperar datos perdidos (eso son los [backups](Backups-y-Restauracion.md)).
- **Tener staging no te exime de cuidado en producción** — sigue siendo el único sitio con datos reales.

## Buenas prácticas avanzadas

- **Paridad exacta o el ensayo no vale** — staging debe correr la misma versión de Odoo (rama y revisión), los mismos módulos en las mismas versiones y, a ser posible, las mismas versiones de PostgreSQL y Python que producción. Un ensayo sobre un staging "parecido" da una falsa seguridad: el fallo aparecerá justo en la diferencia que no replicaste.
- **Refresca staging antes de cada ensayo importante** — un staging clonado hace meses ya no se parece a producción: datos y configuración han divergido. El hábito experto es re-clonar producción justo antes de ensayar algo serio, no reutilizar un staging viejo "porque ya está montado".
- **Ojo con lo que la copia sigue compartiendo con producción** — al clonar, parámetros como `web.base.url` siguen apuntando a la URL real, y las credenciales de servicios externos son las mismas. Si staging usa la misma API key que producción para un transportista o una pasarela de pago, tus pruebas pueden tocar datos reales aunque la base de datos sea una copia: usa credenciales *sandbox* en staging.
- **Convierte el ensayo en un guion** — mientras pruebas en staging, anota cada paso, su orden y cuánto tarda. Ese guion (*runbook*) es lo que ejecutarás en producción: el día del cambio real no se improvisa, se repite lo ensayado.
- **Staging tiene datos reales: protégelo como a producción** — la copia neutralizada no envía correos, pero sigue conteniendo nombres, correos y facturas de clientes de verdad. Restringe quién accede y no lo dejes en una URL pública sin autenticación; a efectos de privacidad (RGPD), staging es tan sensible como producción.

---

*En resumen: separar desarrollo, staging y producción te da un lugar donde equivocarte sin consecuencias; los cambios suben hacia producción y los datos bajan (neutralizados) hacia staging, de modo que nunca ensayas por primera vez sobre los datos reales.*
