# Carpeta Compartida (SMB)

## ¿Qué es?

Una **carpeta compartida** es una carpeta de un equipo que se pone a disposición de otros equipos de la red para que puedan abrir, leer y guardar archivos en ella como si fuera una carpeta propia. La tecnología más habitual para esto en redes con Windows se llama **SMB** (*Server Message Block*).

## ¿Por qué existe?

Pasar archivos de un equipo a otro con un pendrive o por correo es lento e incómodo, sobre todo si varias personas trabajan con los mismos documentos. La carpeta compartida resuelve esto: el archivo vive en un único sitio y todos acceden a él directamente por la red, sin copias sueltas.

> Piensa en ella como un Google Drive privado de tu red local: una carpeta común a la que todos los equipos autorizados pueden entrar.

## ¿Cuándo y para qué se usa?

- **Documentos comunes de un equipo de trabajo**: plantillas, facturas, recursos compartidos.
- **Una unidad de red** que aparece en tu explorador como si fuera otro disco (`Z:`), pero que en realidad está en otro equipo o servidor.
- **Centralizar archivos** para que las copias de seguridad se hagan en un solo punto.
- **Compartir una carpeta entre tu portátil y un servidor o un [NAS](NAS.md)**.

## Lo mínimo que necesitas saber

**1. Compartir una carpeta (en el equipo que la ofrece)**

Clic derecho sobre la carpeta → *Propiedades → Compartir*, eliges quién puede acceder y con qué permisos (solo lectura o lectura y escritura).

**2. Acceder a la carpeta desde otro equipo**

Se usa una ruta especial con doble barra invertida, el nombre o IP del equipo y el nombre del recurso:

```text
\\192.168.1.50\documentos
\\servidor-oficina\plantillas
```

**3. Conectarla como unidad de red**

Puedes asignarle una letra para que aparezca como un disco más:

```bash
# Windows
net use Z: \\192.168.1.50\documentos

# Linux (montar un recurso SMB)
sudo mount -t cifs //192.168.1.50/documentos /mnt/documentos -o username=juan
```

**4. Los permisos mandan**

Llegar a la carpeta no significa poder tocar todo: cada usuario tiene unos permisos (leer, escribir, nada). Si no puedes guardar un archivo, casi siempre es cuestión de permisos, no de conexión.

**5. Puerto 445**

SMB usa el puerto **445**. Es un dato útil cuando algo no conecta y hay que revisar el cortafuegos.

## Lo que NO hace

- **No sincroniza copias** — no hay una copia en tu equipo que se actualice sola; trabajas directamente sobre el archivo remoto. Si pierdes la conexión, pierdes el acceso.
- **No es seguro exponerlo a internet** — SMB está pensado para la red local; para acceder desde fuera se hace a través de una [VPN](VPN.md).
- **No controla versiones** — si alguien sobrescribe un archivo, no hay historial salvo que lo aporte una copia de seguridad aparte.

---

*En resumen: una carpeta compartida pone archivos en un punto común de la red para que varios equipos trabajen sobre ellos sin copias sueltas ni pendrives.*
