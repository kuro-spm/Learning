# Backups y restauración

**¿Qué es?** Un backup (copia de seguridad) es una foto completa de la base de datos de Odoo más su *filestore* (los archivos adjuntos: imágenes, PDFs, documentos). Restaurar es volver a cargar esa foto para dejar el sistema como estaba en el momento en que se hizo. Es tu red de seguridad: si algo sale mal, vuelves atrás.

---

## ¿Por qué existe?

Por muy bien que planifiques, una prueba puede salir mal, un módulo puede corromper datos o alguien puede borrar algo por error. Sin una copia reciente, esos datos se pierden para siempre. El backup existe para que "deshacer" sea siempre posible, aunque el daño ya esté hecho.

> Es el equivalente a un punto de guardado en un videojuego: si la siguiente pantalla te mata, recargas la partida y no pierdes horas de progreso.

---

## ¿Cuándo y para qué se usa?

- **Siempre, justo antes** de instalar un módulo, actualizar o hacer un cambio importante en producción.
- De forma **automática y periódica** (diaria), para tener siempre una copia reciente aunque nadie se acuerde de hacerla.
- Para **crear un entorno de pruebas**: un backup de producción es el punto de partida de la copia que luego neutralizas.

---

## Lo mínimo que necesitas saber

**1. Un backup completo son dos cosas**

- La **base de datos** (PostgreSQL): registros, configuración, todo el "texto".
- El **filestore**: los ficheros adjuntos, que no viven en la base de datos sino en el disco.

Un backup que solo incluye la base de datos deja fuera las imágenes y documentos. El backup oficial de Odoo (formato `.zip`) incluye ambos.

**2. Cómo se hace y se restaura**

Desde el gestor de bases de datos (`/web/database/manager`) hay botones **Backup** y **Restore**. Por línea de comandos:

```bash
# Copia de la base de datos
pg_dump -Fc produccion -f produccion_2026-07-01.dump

# Restaurar esa copia en una base de datos nueva
createdb odoo_restaurada
pg_restore -d odoo_restaurada produccion_2026-07-01.dump
```

**3. Un backup solo sirve si sabes que se puede restaurar**

Una copia que nunca has probado a restaurar es una copia en la que no puedes confiar. De vez en cuando, restaura un backup en una base de datos de prueba para comprobar que realmente funciona.

---

## Lo que NO hace

- **No se hace solo** salvo que configures copias automáticas — en una instalación propia, la periodicidad es responsabilidad tuya.
- **No incluye el filestore si haces solo un `pg_dump`** — recuerda copiar también los archivos adjuntos.
- **No es lo mismo que un entorno de staging** — el backup sirve para recuperar; staging, para ensayar.

---

*En resumen: un backup es tu punto de guardado antes de cualquier cambio arriesgado; incluye base de datos y filestore, se prueba de vez en cuando restaurándolo, y convierte cualquier error en algo reversible.*
