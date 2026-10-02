# Documentos Word con python-docx

## ¿Qué es?

`python-docx` es una librería de Python para **crear y modificar ficheros `.docx`** (Word) desde código, sin tener Word instalado: se describe el documento con párrafos, tablas e imágenes, y se guarda.

## ¿Por qué existe?

Un `.docx` es un ZIP con varios ficheros XML. Escribirlos a mano es inviable, y automatizar Word mediante COM solo funciona en Windows con Office instalado. `python-docx` ofrece una API de objetos (`Document`, `Paragraph`, `Table`...) que genera el XML por ti y funciona igual en cualquier sistema operativo, incluido un servidor o un pipeline de CI.

> Si ya conoces una librería de PDF o de Excel (`openpyxl`), la idea es la misma: construyes el documento en memoria con una API de objetos y lo vuelcas a disco.

## ¿Cuándo y para qué se usa?

Cuando el **mismo contenido** debe llegar a alguien en formato Word: un informe mensual con tablas generadas desde la base de datos, facturas con el mismo diseño, o el manual de usuario de una tienda online que se mantiene en Markdown (la fuente de verdad) y se entrega también como `.docx`. Regla útil: el fichero fuente se edita, el `.docx` se **regenera**; si alguien retoca el `.docx` a mano, ese cambio se pierde en la siguiente generación.

## Instalación

```bash
pip install python-docx
```

Se instala como `python-docx`, pero se importa como `docx`: `from docx import Document`. Hay un paquete distinto, también llamado `docx`, que no es este y da errores confusos si se instala por error.

## Tu primer documento

Un documento se crea con `Document()`, se rellena con `add_heading`, `add_paragraph` y se guarda con `save`:

```python
from docx import Document

doc = Document()
doc.add_heading('Informe mensual de ventas', level=1)
doc.add_paragraph('Resumen de las ventas de la tienda online durante octubre.')

doc.add_heading('Puntos destacados', level=2)
doc.add_paragraph('Las ventas crecieron un 12 %', style='List Bullet')
doc.add_paragraph('El producto más vendido fue la camiseta básica', style='List Bullet')

doc.add_heading('Pasos siguientes', level=2)
doc.add_paragraph('Revisar el inventario', style='List Number')
doc.add_paragraph('Preparar la campaña de noviembre', style='List Number')

doc.save('informe.docx')
```

Qué hace: `add_heading` con `level=1..9` usa los estilos de título de Word (`level=0` da el estilo `Title`, el de portada); las listas no son una función aparte, son **estilos** de párrafo (`List Bullet`, `List Number`) que ya trae la plantilla por defecto. `save` escribe el fichero; si ya existe, lo sobrescribe sin avisar.

### Texto con formato dentro de un párrafo

Un párrafo se compone de *runs*: tramos de texto con su propio formato. Para mezclar negrita y texto normal se añaden varios runs:

```python
p = doc.add_paragraph('Total del pedido: ')
run = p.add_run('149,90 €')
run.bold = True
run.italic = False
```

`add_paragraph('texto')` es la forma corta de crear un párrafo con un único run.

## Estilos y fuentes

Cada párrafo tiene un **estilo** (`Normal`, `Heading 1`, `List Bullet`...), y los estilos controlan la fuente, el tamaño y el espaciado. La forma ordenada de dar formato a todo el documento es cambiar el estilo una vez, en lugar de formatear párrafo a párrafo:

```python
from docx.shared import Pt, RGBColor

normal = doc.styles['Normal']
normal.font.name = 'Calibri'
normal.font.size = Pt(11)

h1 = doc.styles['Heading 1']
h1.font.name = 'Calibri'
h1.font.size = Pt(20)
h1.font.color.rgb = RGBColor(0x1B, 0x3A, 0x5C)
```

Cuando hace falta formato puntual, se aplica a un run (`run.font.size`, `run.font.color.rgb`, `run.bold`). Para fijar el formato de un párrafo entero (sangría, espacio antes y después) se usa `paragraph.paragraph_format`:

```python
from docx.shared import Pt, Cm

p.paragraph_format.space_after = Pt(6)
p.paragraph_format.left_indent = Cm(0.6)
```

Las medidas se expresan con objetos `Pt`, `Cm`, `Inches` de `docx.shared`, no con números sueltos. Los márgenes se cambian por sección: `doc.sections[0].left_margin = Cm(2.2)`.

## Tablas

`add_table(rows, cols)` crea la tabla; las celdas se rellenan con `cell(fila, columna).text`, y las filas nuevas con `add_row()`:

```python
filas = [
    ('Camiseta básica', 3, '12,00 €'),
    ('Sudadera con capucha', 1, '39,90 €'),
]

tabla = doc.add_table(rows=1, cols=3)
tabla.style = 'Table Grid'          # bordes simples

cabecera = tabla.rows[0].cells
cabecera[0].text, cabecera[1].text, cabecera[2].text = 'Producto', 'Cantidad', 'Precio'

for producto, cantidad, precio in filas:
    celdas = tabla.add_row().cells
    celdas[0].text = producto
    celdas[1].text = str(cantidad)
    celdas[2].text = precio
```

Qué hace: crea una tabla con la fila de cabecera y añade una fila por cada dato. Los valores de las celdas son siempre texto, por lo que los números se convierten con `str()`. `Table Grid` es el estilo de bordes más simple de la plantilla por defecto.

Para dar formato a una celda (negrita en la cabecera, por ejemplo) se trabaja con su párrafo: `celda.paragraphs[0].runs[0].bold = True`. Cosas como el **color de fondo de una celda** (para cabeceras o filas alternas tipo «cebra») no tienen API propia y obligan a tocar el XML de la celda:

```python
from docx.oxml import OxmlElement
from docx.oxml.ns import qn

def fondo_celda(celda, hex_color):
    sombreado = OxmlElement('w:shd')
    sombreado.set(qn('w:val'), 'clear')
    sombreado.set(qn('w:fill'), hex_color)   # p. ej. 'EEEEF0'
    celda._tc.get_or_add_tcPr().append(sombreado)
```

`_tc` es un atributo interno (por eso empieza por guion bajo): funciona y es el camino habitual, pero es parte de la implementación, no de la API pública, y podría cambiar entre versiones.

## Imágenes con ancho y pie de foto

`add_picture` añade una imagen. Se indica el **ancho** (o el alto) y la librería mantiene la proporción; si se omiten los dos, usa el tamaño original del fichero, que suele desbordar la página:

```python
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.shared import Cm

parrafo = doc.add_paragraph()
parrafo.alignment = WD_ALIGN_PARAGRAPH.CENTER
parrafo.add_run().add_picture('capturas/catalogo.jpg', width=Cm(14))

pie = doc.add_paragraph('Figura 1. Catálogo de productos', style='Caption')
pie.alignment = WD_ALIGN_PARAGRAPH.CENTER
```

Qué hace: una imagen vive dentro de un *run*, por eso se crea un párrafo, se centra y se le añade el run con la imagen. `doc.add_picture(ruta, width=...)` es el atajo, pero coloca la imagen en un párrafo propio que no puedes centrar tan cómodamente. El pie de foto es simplemente otro párrafo, aquí con el estilo `Caption` de la plantilla por defecto.

El ancho útil depende de los márgenes: en una página A4 (21 cm) con 2,2 cm de margen a cada lado quedan unos 16,6 cm; un ancho de 14-16 cm es razonable. Para no ampliar imágenes pequeñas, comprueba el ancho natural antes (o genera las capturas ya a un tamaño adecuado).

## Cabecera, pie y portada

Cada sección del documento tiene una cabecera y un pie, accesibles desde `doc.sections[0]`. Se rellenan como cualquier párrafo:

```python
seccion = doc.sections[0]
seccion.header.paragraphs[0].text = 'Manual de la tienda online'
seccion.footer.paragraphs[0].text = 'Documento interno'
seccion.different_first_page_header_footer = True   # la portada queda sin cabecera ni pie
```

Con `different_first_page_header_footer = True` la primera página usa su propia cabecera y pie (`first_page_header`, `first_page_footer`), que están vacíos por defecto: es la forma estándar de tener una portada limpia.

El **número de página** es un caso especial: no es texto, es un *campo* de Word, y `python-docx` no tiene API para él. Se inserta con XML:

```python
def numero_de_pagina(parrafo):
    run = parrafo.add_run()
    for tipo, texto in (('begin', None), (None, 'PAGE'), ('end', None)):
        if tipo:
            el = OxmlElement('w:fldChar')
            el.set(qn('w:fldCharType'), tipo)
        else:
            el = OxmlElement('w:instrText')
            el.set(qn('xml:space'), 'preserve')
            el.text = texto
        run._r.append(el)

numero_de_pagina(seccion.footer.paragraphs[0])
```

La **portada** es un conjunto de párrafos con tamaño de letra grande (título, subtítulo, fecha) seguido de un salto de página con `doc.add_page_break()`. Con `level=0`, `add_heading` usa el estilo `Title`:

```python
doc.add_heading('Manual de la tienda online', level=0)
doc.add_paragraph('Versión 1.0 · Octubre 2026')
doc.add_page_break()
```

## Convertir Markdown a docx

Con las piezas anteriores se puede escribir un conversor sencillo de un subconjunto de Markdown. La idea es leer **línea a línea** y decidir qué bloque empieza con cada una. No hace falta una librería de parseo si el Markdown que se usa es disciplinado (títulos, párrafos, listas, tablas, imágenes):

```python
import os
import re
from docx import Document
from docx.shared import Cm

IMG = re.compile(r'^!\[([^\]]*)\]\(([^)]+)\)\s*$')

def markdown_a_docx(origen, destino):
    base = os.path.dirname(os.path.abspath(origen))
    doc = Document()
    with open(origen, encoding='utf-8') as f:
        for linea in f:
            linea = linea.rstrip('\n')
            if not linea.strip():
                continue
            if m := re.match(r'^(#{1,4})\s+(.*)$', linea):
                doc.add_heading(m.group(2), level=len(m.group(1)))
            elif m := re.match(r'^[-*]\s+(.*)$', linea):
                doc.add_paragraph(m.group(1), style='List Bullet')
            elif m := re.match(r'^\d+\.\s+(.*)$', linea):
                doc.add_paragraph(m.group(1), style='List Number')
            elif m := IMG.match(linea):
                poner_imagen(doc, m.group(1), os.path.join(base, m.group(2)))
            else:
                doc.add_paragraph(linea)
    doc.save(destino)
```

Qué hace: cada línea se clasifica por su inicio (`#`, `-`, `1.`, `![`) y se traduce a la llamada equivalente; todo lo demás es un párrafo. Es suficiente para un manual sencillo. Las rutas de las imágenes se resuelven **relativas al `.md`**, no al directorio desde el que se lanza el script.

Quedan fuera cosas que casi siempre acaban haciendo falta y que conviene resolver en este orden:

1. **Negrita, cursiva y código dentro del texto.** Se parte la línea con una expresión regular (`(\*\*.+?\*\*|`.+?`|\*.+?\*)`) y se crea un *run* por tramo, con `bold`, `italic` o fuente monoespaciada según el marcador.
2. **Párrafos que ocupan varias líneas.** Si la prosa se parte en varias líneas, hay que acumularlas en un búfer y volcarlas como un solo párrafo al llegar una línea vacía u otro bloque; si no, cada línea sería un párrafo.
3. **Tablas.** Se detectan líneas que empiezan por `|` con una fila separadora (`|---|---|`); se acumulan las filas y se crea la tabla con `add_table`.
4. **Citas (`>`)**, que se pueden aproximar con un párrafo con sangría y estilo `Intense Quote` o `Quote` si la plantilla los incluye.

Para convertir Markdown de verdad en casos complejos hay herramientas hechas: ver «Alternativas».

## Fallos frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `FileNotFoundError` al añadir una imagen | La ruta es relativa a otra carpeta (la del script, no la del `.md`). | Resolver con `os.path.join(carpeta_del_md, ruta)` y comprobar `os.path.isfile` antes. |
| `UnrecognizedImageError` | Formato no soportado: `python-docx` admite PNG, JPEG, GIF, BMP y TIFF, **no** WebP ni SVG. | Convertir antes la imagen (p. ej. con Pillow). |
| La imagen se sale de la página | No se indicó `width`, y se usó el tamaño original. | Indicar siempre `width` (o `height`). |
| `KeyError: "no style with name 'X'"` | El estilo no existe en la plantilla por defecto (o está en otro idioma). | Usar nombres en inglés de la plantilla por defecto; o crear el estilo con `doc.styles.add_style`. |
| Páginas en blanco o títulos huérfanos al final de página | Un `add_page_break` seguido de un título que ya tiene `page_break_before`. | Un solo mecanismo de salto por título; `paragraph_format.keep_with_next = True` para que el título no quede separado de lo que sigue. |
| La fuente no cambia en parte del texto | El formato se puso en el estilo, pero el run tiene su propio formato. | Los runs tienen prioridad: formatea o en el estilo o en el run, sin mezclar. |
| El número de página sale como texto | Se escribió «Página 1» a mano. | Insertar el campo `PAGE` como en la sección de cabecera y pie. |
| Un fallo de imagen tumba todo el documento | Excepción sin capturar. | Capturar el error y escribir un aviso visible (`[Imagen no encontrada: ruta]`) en lugar de la imagen. |

El último caso merece una función propia, porque en un conversor automático es preferible un documento con un aviso visible a un proceso que se interrumpe en la imagen 17 de 30:

```python
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.shared import Cm, RGBColor

def poner_imagen(doc, texto_alt, ruta):
    parrafo = doc.add_paragraph()
    parrafo.alignment = WD_ALIGN_PARAGRAPH.CENTER
    try:
        parrafo.add_run().add_picture(ruta, width=Cm(14))
    except FileNotFoundError:
        aviso = parrafo.add_run(f'[Imagen no encontrada: {ruta}]')
        aviso.italic = True
        aviso.font.color.rgb = RGBColor(0xB4, 0x53, 0x09)
    if texto_alt:
        pie = doc.add_paragraph(texto_alt, style='Caption')
        pie.alignment = WD_ALIGN_PARAGRAPH.CENTER
```

Se captura `FileNotFoundError` y no `Exception` a secas, para no esconder otros errores. Si además se quiere tolerar formatos no soportados, se añade `except UnrecognizedImageError` (se importa de `docx.image.exceptions`).

## Alternativas

- **Pandoc** convierte Markdown a `.docx` directamente (`pandoc manual.md -o manual.docx`) y cubre casi toda la sintaxis. Con `--reference-doc=plantilla.docx` toma los estilos de un documento Word que tú diseñes. Si no necesitas un control fino sobre la composición, suele ser la opción más corta.
- **Plantillas con `docxtpl`**: parte de un `.docx` diseñado en Word con marcadores tipo Jinja (`{{ cliente }}`) y los rellena desde Python. Buena opción para facturas y cartas donde el diseño lo mantiene alguien que no programa.
- **`Document('plantilla.docx')` en `python-docx`**: abre un documento existente y toma sus estilos, cabeceras y márgenes como base en lugar de la plantilla por defecto. Sirve para dar una identidad visual sin reproducirla en código.
- **La librería `docx` de Node.js** (`docx.js.org`): equivalente para quien trabaja en JavaScript/TypeScript, con una API declarativa.

## Buenas prácticas avanzadas

- **El `.docx` es un producto derivado: la fuente se edita, el fichero se regenera.** Si se acepta que alguien retoque el `.docx` a mano, el siguiente `generar` lo pisará. Deja claro, en el propio fichero o en el README, cuál es la fuente de verdad.
- **Define estilos una vez y úsalos por nombre**, en lugar de formatear cada run. Cuando el diseño cambie (otra fuente, otro color de título), se toca un sitio, no cien.
- **Usa una plantilla `.docx` como base** (`Document('plantilla.docx')`) para márgenes, cabeceras, estilos de tabla y de cita. Es mucho más fácil diseñarlos en Word que en código, y el script solo rellena contenido.
- **Controla el flujo de página con propiedades, no con saltos manuales:** `keep_with_next` en títulos, `page_break_before` solo en los títulos de sección. Los `add_page_break()` sueltos se desordenan en cuanto el contenido crece.
- **Resuelve las rutas de recursos respecto al documento fuente.** Un conversor que solo funciona si se lanza desde cierta carpeta es una fuente de errores intermitentes.
- **Abre el resultado en Word, no solo en un visor.** Varios visores (y LibreOffice) toleran XML que Word rechaza con «el archivo está dañado»; los campos y el XML manual son donde suele ocurrir.

## Documentación oficial

- [Documentación de python-docx](https://python-docx.readthedocs.io/en/latest/) — la guía de inicio rápido (*Quickstart*) cubre párrafos, tablas e imágenes en pocas páginas; la sección *Working with Styles* explica la lógica de estilos.
- [Repositorio de python-docx](https://github.com/python-openxml/python-docx) — útil para consultar el código fuente cuando la documentación no dice qué hace exactamente un atributo, y para ver los *issues* cuando algo no encaja.
- [Pandoc: salida a Word](https://pandoc.org/MANUAL.html#option--reference-doc) — la opción `--reference-doc`, para quien prefiera convertir sin escribir código.

---

*En resumen: `python-docx` construye un `.docx` a base de párrafos, tablas e imágenes con estilos; si tratas el Word como un resultado que se regenera desde una fuente y no como un fichero que se edita, un conversor sencillo de unas decenas de líneas basta para la mayoría de manuales.*
