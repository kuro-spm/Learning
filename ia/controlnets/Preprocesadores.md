# Preprocesadores

## ¿Qué es?

Un preprocesador —también llamado *annotator*, «anotador»— es el programa que convierte una imagen normal en el **mapa de condicionamiento** que consume un modelo [ControlNet](ControlNet.md): bordes, profundidad, esqueleto de pose, regiones semánticas. Es la pieza que decide **qué información de tu imagen sobrevive** y cuál se destruye.

## ¿Por qué existe?

Un modelo ControlNet está entrenado para leer un tipo muy concreto de imagen: el de Canny espera líneas blancas sobre fondo negro; el de profundidad espera una escala de grises donde el gris significa distancia; el de pose espera un esqueleto de palotes de colores fijos. Ninguno sabe interpretar una fotografía.

Alguien tiene que hacer la traducción, y ese alguien es el preprocesador. Unos son algoritmos clásicos de visión por computador de los años ochenta —Canny es de 1986— y otros son redes neuronales entrenadas para la tarea. En ambos casos son **independientes de la difusión**: no saben que existe un modelo generativo detrás, y se pueden ejecutar y mirar por separado.

Pero el motivo profundo por el que esta pieza merece una ficha entera es otro. El preprocesador **no es un detalle de implementación: es la decisión de diseño principal de todo el flujo**. Cuando eliges Canny en lugar de profundidad no estás eligiendo «cómo se detecta la estructura», estás eligiendo qué te reservas y qué le regalas al modelo. Todo lo que el mapa no contenga queda libre para el prompt; todo lo que contenga queda impuesto. El peso y la ventana ajustan *cuánto* se respeta lo que hay en el mapa, pero no pueden devolver lo que el preprocesador ya tiró.

> Piénsalo como un `SELECT` sobre la imagen. El preprocesador es la lista de columnas: el resto de la consulta no puede recuperar lo que no proyectaste.

## ¿Cuándo y para qué se usa?

Cada vez que uses ControlNet, salvo que ya tengas el mapa hecho (un boceto a línea, un render de profundidad salido de un motor 3D, un esqueleto dibujado a mano).

La elección se hace planteando la pregunta al revés de como sale sola. La pregunta natural es «¿cuál captura mejor mi imagen?», y es la equivocada: el que mejor la captura es el que más ata, y acabas con un calco. La pregunta correcta es:

> **¿Qué es lo mínimo que necesito imponer para que lo demás pueda cambiar?**

Sobre los cuatro escenarios de la colección:

| Lo que quieres | Lo que debe quedar libre | Preprocesador |
|---|---|---|
| Recolorear el mockup de la silla | Material, color, iluminación | **Canny** o **lineart** (acromáticos) |
| Variar el vestuario del personaje | Ropa, fondo, estilo, hasta el cuerpo | **OpenPose** (solo el esqueleto) |
| Convertir el boceto en render | Todo salvo el trazo | **Scribble** o **lineart** |
| Cambiar el estilo del salón | Materiales, colores, época | **Depth** (solo el volumen) |

---

## Cómo se ejecuta un preprocesador

Casi todos viven en el paquete `controlnet_aux`, de Hugging Face:

```bash
pip install controlnet_aux opencv-python
```

Y casi todos siguen el mismo patrón: se instancian una vez —descargando pesos si son redes— y se llaman con la imagen.

```python
from controlnet_aux import HEDdetector
from diffusers.utils import load_image

detector = HEDdetector.from_pretrained("lllyasviel/Annotators")   # se hace UNA vez
original = load_image("salon.jpg")

mapa = detector(original, detect_resolution=1024, image_resolution=1024)
mapa.save("mapa-hed.png")
```

Los dos parámetros de resolución son los que casi nadie mira y sí importan:

- `detect_resolution` — a qué tamaño analiza la red la imagen. Más alto detecta más detalle fino y tarda más.
- `image_resolution` — a qué tamaño devuelve el mapa. Debe coincidir con la resolución a la que vas a generar.

Si los dejas por defecto (512 en la mayoría) y luego generas a 1024, el mapa se amplía y las líneas salen gruesas y blandas. Pásalos siempre explícitamente.

La excepción al patrón es `CannyDetector`, que no descarga nada porque no es una red:

```python
from controlnet_aux import CannyDetector

detector = CannyDetector()                                    # sin from_pretrained
mapa = detector(original, low_threshold=100, high_threshold=200,
                detect_resolution=1024, image_resolution=1024)
```

Estos son los preprocesadores que expone el paquete, para tenerlos localizados: `CannyDetector`, `HEDdetector`, `PidiNetDetector`, `TEEDdetector`, `AnylineDetector`, `LineartDetector`, `LineartAnimeDetector`, `LineartStandardDetector`, `MLSDdetector`, `MidasDetector`, `ZoeDetector`, `LeresDetector`, `NormalBaeDetector`, `OpenposeDetector`, `DWposeDetector`, `MediapipeFaceDetector`, `SamDetector` y `ContentShuffleDetector`. Los que faltan —segmentación semántica y tile— se resuelven por otras vías y se ven más abajo.

## Qué conserva y qué tira cada familia

Antes del detalle, el mapa general. Esta es la tabla a la que volver:

| Familia | Qué captura | Qué destruye | Grado de atadura |
|---|---|---|---|
| Bordes (Canny, lineart, scribble, MLSD) | Contornos y líneas | Color, material, volumen, iluminación | Alto a medio |
| Soft edge (HED, PiDiNet) | Contornos suaves con intensidad | Color y material; conserva algo de sombra | Medio |
| Profundidad | Distancia a la cámara, volumen | Color, textura, detalle plano | Medio |
| Normales | Orientación de cada superficie | Color y distancia absoluta | Medio-alto |
| Pose | Posición de articulaciones | Absolutamente todo lo demás | Bajo |
| Segmentación | Qué clase de objeto hay en cada región | Forma fina, color, textura | Medio |
| Tile | Un recorte de la imagen tal cual | Casi nada | Muy alto |

«Grado de atadura» es lo que más conviene interiorizar: no es una medida de calidad, es una medida de **cuánta libertad le queda al prompt**.

---

## La familia de bordes: lo que sobrevive es la línea

Todos estos producen mapas **acromáticos**: líneas claras sobre fondo oscuro. Es la familia de «misma forma, otra apariencia», y no sirve para nada que dependa del color de la entrada.

### Canny

Detección de bordes clásica, por gradiente. Produce líneas de un píxel, duras y binarias: un píxel es borde o no lo es, sin medias tintas.

```python
from controlnet_aux import CannyDetector

detector = CannyDetector()
mapa = detector(load_image("mockup-silla.png"),
                low_threshold=100, high_threshold=200,
                detect_resolution=1024, image_resolution=1024)
```

Los dos umbrales son el único mando que tiene, y funcionan por histéresis: un gradiente por encima del **alto** es borde seguro; uno entre ambos solo cuenta si conecta con un borde seguro; uno por debajo del **bajo** se descarta.

| Umbrales | Resultado | Cuándo |
|---|---|---|
| `50 / 100` | Muchas líneas, incluida la textura de la superficie | Objetos de bajo contraste, o cuando quieres atar mucho |
| `100 / 200` | El punto de partida razonable | Casi todo |
| `200 / 300` | Solo los contornos más marcados | Fotos con mucho ruido o textura que no quieres imponer |

**Cuándo usarlo:** productos, arquitectura, cualquier cosa con contornos limpios y bien definidos. El escenario del mockup de la silla es su caso ideal — la silueta y las líneas de construcción se conservan clavadas y el material queda libre.

**Cuándo no:** con fotos de personas o de escenas complejas, porque captura también la textura de la piel, los pliegues de la ropa y las sombras del fondo, y eso ata muchísimo más de lo que parece. Y con imágenes de bajo contraste, donde puede salirte un mapa casi vacío sin que te des cuenta.

**El fallo típico:** un mapa demasiado denso. Si al mirarlo ves una maraña de líneas cubriendo toda la imagen, el ControlNet las va a respetar todas y la salida parecerá una ilustración calcada. Sube los umbrales.

### Soft edge: HED y PiDiNet

Redes de detección de bordes que producen líneas **suaves, con grosor y con intensidad variable**. Un borde muy marcado sale blanco; uno tenue, gris.

```python
from controlnet_aux import HEDdetector, PidiNetDetector

hed = HEDdetector.from_pretrained("lllyasviel/Annotators")
pidi = PidiNetDetector.from_pretrained("lllyasviel/Annotators")

mapa = hed(load_image("salon.jpg"), detect_resolution=1024, image_resolution=1024)
```

Esa gradación cambia el comportamiento por completo respecto a Canny: como el mapa distingue «borde fuerte» de «insinuación de borde», el modelo tiene margen para interpretar las zonas grises. El resultado es más natural y menos calcado, a cambio de menos precisión.

Entre los dos: **HED** es más antiguo y produce líneas más gruesas y expresivas; **PiDiNet** es mucho más rápido y algo más limpio. `TEEDdetector` es una alternativa moderna que da líneas finas y estables. Para elegir, genera los tres mapas y mira cuál se parece más a lo que tienes en la cabeza — cuesta treinta segundos.

**Cuándo usarlo:** retratos, escenas naturales, cualquier cosa orgánica. Y siempre que Canny te esté saliendo demasiado rígido.

### Lineart

Redes entrenadas específicamente para producir **dibujo de línea** al estilo de una ilustración, no bordes al estilo de un algoritmo. La diferencia frente a soft edge es que lineart entiende qué líneas dibujaría una persona: descarta la textura y se queda con el contorno significativo.

```python
from controlnet_aux import LineartDetector, LineartAnimeDetector

lineart = LineartDetector.from_pretrained("lllyasviel/Annotators")

mapa = lineart(load_image("mockup-silla.png"),
               coarse=False,                        # True = trazo más suelto
               detect_resolution=1024, image_resolution=1024)
```

`LineartAnimeDetector` es la variante entrenada con ilustración de estilo anime, y sobre ese material funciona notablemente mejor. `LineartStandardDetector` es una versión ligera y determinista, sin red.

**Cuándo usarlo:** ilustración, cómic, diseño de producto, y en general cuando lo que quieres conservar es «el dibujo» y no «los bordes». Suele ser mejor opción que Canny para cualquier cosa que no sea geometría dura.

### Scribble

Bordes reducidos a **trazo grueso y basto**, como un garabato. Es deliberadamente impreciso: tira casi todo el detalle y deja solo la idea de la forma.

```python
from controlnet_aux import HEDdetector

hed = HEDdetector.from_pretrained("lllyasviel/Annotators")
mapa = hed(load_image("boceto-silla.png"), scribble=True,     # ← el modo garabato
           detect_resolution=1024, image_resolution=1024)
```

Es el más liberador de la familia: el modelo entiende la composición y la respeta a grandes rasgos, pero rellena el detalle a su gusto.

**Cuándo usarlo:** el escenario del boceto a render es literalmente para lo que se diseñó. Y —esto es lo importante— si lo que tienes ya *es* un boceto a mano, **no lo preprocesa nadie**: se pasa directo al ControlNet de scribble, invirtiendo los colores si hace falta para que las líneas queden claras sobre fondo oscuro.

### MLSD

Detector de **segmentos rectos**, y solo rectos. Todo lo curvo desaparece.

```python
from controlnet_aux import MLSDdetector

mlsd = MLSDdetector.from_pretrained("lllyasviel/Annotators")
mapa = mlsd(load_image("salon.jpg"), thr_v=0.1, thr_d=0.1,
            detect_resolution=1024, image_resolution=1024)
```

`thr_v` filtra por confianza de la detección y `thr_d` por longitud mínima del segmento; subirlos deja solo las líneas largas y seguras.

**Cuándo usarlo:** interiorismo y arquitectura, donde lo que hay que conservar es la perspectiva, las paredes, las ventanas y el suelo — y donde te da igual la forma de los muebles. En la foto del salón, MLSD conserva la caja de la habitación y deja el mobiliario completamente libre. Es una elección muy poco frecuente y sorprendentemente buena para «mismo espacio, otro contenido».

**Cuándo no:** con cualquier cosa orgánica. Sobre una silla curva, MLSD devuelve un mapa casi vacío.

---

## La familia geométrica: profundidad y normales

Aquí ya no hablamos de líneas sino de **volumen**. Son mapas que describen el espacio tridimensional de la escena.

### Depth (profundidad)

Una imagen en escala de grises donde el brillo significa **distancia a la cámara**: por convención en este ecosistema, más claro es más cerca.

```python
import numpy as np
from PIL import Image
from transformers import pipeline

estimador = pipeline("depth-estimation", model="depth-anything/Depth-Anything-V2-Large-hf")
salida = estimador(load_image("salon.jpg"))["depth"]

arr = np.array(salida)
arr = (arr - arr.min()) / (arr.max() - arr.min()) * 255      # normalizar a 0-255
mapa = Image.fromarray(np.stack([arr.astype(np.uint8)] * 3, axis=-1)).resize((1024, 1024))
```

La normalización no es opcional: los estimadores devuelven valores en rangos arbitrarios y el ControlNet espera 0-255. Un mapa sin normalizar sale casi todo negro y el control no funciona.

Hay varios estimadores y la diferencia entre ellos se nota:

| Estimador | Cómo se usa | Carácter |
|---|---|---|
| **Depth Anything V2** | `transformers`, como arriba | El mejor hoy en día. Detalle fino y bordes limpios. |
| **MiDaS** | `MidasDetector.from_pretrained("lllyasviel/Annotators")` | El clásico. Mapas suaves y algo planos, pero es con el que se entrenaron los ControlNet de SD 1.5, así que encaja bien con ellos. |
| **ZoeDepth** | `ZoeDetector.from_pretrained("lllyasviel/Annotators")` | Profundidad métrica, con más contraste entre planos. |
| **LeReS** | `LeresDetector.from_pretrained("lllyasviel/Annotators")` | Buena separación de primer plano y fondo. |

**Cuándo usarlo:** cambiar el estilo de una escena conservando su espacio. Es la elección para la foto de interiorismo: el salón mantiene la perspectiva, las proporciones y la posición de los muebles, y el estilo, los materiales y los colores son libres. También va muy bien para conservar la pose y el volumen de una figura sin atarla al contorno exacto.

**Su punto débil:** la profundidad no distingue objetos que están a la misma distancia. Dos cuadros colgados en la misma pared son, para el mapa, la misma mancha gris. Si necesitas que sigan siendo dos cuadros, la profundidad sola no basta — y ahí es donde empieza a tener sentido [apilar controles](Multi-ControlNet.md).

### Normal map

Cada píxel codifica en RGB la **orientación de la superficie** en ese punto: los tres canales son las componentes X, Y y Z del vector normal. De ahí ese aspecto característico entre morado y verde.

```python
from controlnet_aux import NormalBaeDetector

detector = NormalBaeDetector.from_pretrained("lllyasviel/Annotators")
mapa = detector(load_image("mockup-silla.png"), detect_resolution=1024, image_resolution=1024)
```

Frente a la profundidad, el mapa de normales captura mucho mejor el **relieve local** —un pliegue, una moldura, la curvatura de un respaldo— y peor la distancia global. Se traduce en resultados con más definición de superficie y en una iluminación más creíble, porque el modelo sabe hacia dónde mira cada trozo de objeto.

**Cuándo usarlo:** producto y escultura, cuando el detalle de la superficie importa más que la posición en la escena. Es un mapa poco usado y bastante infravalorado.

**Aviso de compatibilidad:** los ControlNet de normales son abundantes en SD 1.5 y escasos en SDXL. Comprueba que existe uno para tu familia antes de montar el flujo.

---

## La familia semántica: pose y segmentación

Estos dos no describen píxeles: describen **qué hay** y **dónde**.

### OpenPose

Detecta personas y devuelve su **esqueleto**: puntos de articulación unidos por segmentos de colores fijos, sobre fondo negro. Opcionalmente añade manos y cara.

```python
from controlnet_aux import OpenposeDetector

detector = OpenposeDetector.from_pretrained("lllyasviel/Annotators")

mapa = detector(load_image("personaje.png"),
                include_body=True,
                include_hand=True,      # 21 puntos por mano
                include_face=True,      # 68 puntos faciales
                detect_resolution=1024, image_resolution=1024)
```

`DWposeDetector` es la alternativa moderna: bastante más precisa, sobre todo en manos, a cambio de descargar más pesos.

Este es, con diferencia, **el preprocesador que menos ata**. El mapa no contiene la complexión de la persona, ni su ropa, ni su cara, ni el fondo, ni el estilo: contiene diecisiete puntos y unas rayas. Por eso es la herramienta para el escenario del personaje — se mantiene el gesto exacto y todo lo demás se puede reinventar.

Dos advertencias prácticas:

- **Si no detecta a nadie, devuelve un mapa negro** y la generación sale como si no hubiera control. Míralo siempre. Falla con figuras muy recortadas, muy pequeñas o vistas desde ángulos extremos.
- **Las manos siguen siendo el punto flaco.** Activar `include_hand` ayuda, pero los ControlNet de pose reproducen la posición de la mano mucho peor que la del cuerpo. No esperes milagros ahí.

### Segmentación semántica

Divide la imagen en regiones y pinta cada una con el color que su clase tiene asignado en una paleta fija —la de ADE20K, con 150 clases—. El resultado parece un mapa político: un bloque de color por «pared», otro por «suelo», otro por «silla».

En `diffusers` la vía habitual pasa por `transformers`:

```python
import numpy as np
import torch
from PIL import Image
from transformers import AutoImageProcessor, UperNetForSemanticSegmentation

procesador = AutoImageProcessor.from_pretrained("openmmlab/upernet-convnext-small")
segmentador = UperNetForSemanticSegmentation.from_pretrained("openmmlab/upernet-convnext-small")

original = load_image("salon.jpg")
entradas = procesador(original, return_tensors="pt")
with torch.no_grad():
    salida = segmentador(**entradas)

seg = procesador.post_process_semantic_segmentation(salida, target_sizes=[original.size[::-1]])[0]

color = np.zeros((seg.shape[0], seg.shape[1], 3), dtype=np.uint8)
for etiqueta, tono in enumerate(PALETA_ADE20K):        # la paleta oficial de 150 colores
    color[seg.numpy() == etiqueta] = tono
mapa = Image.fromarray(color).resize((1024, 1024))
```

Ese `PALETA_ADE20K` no es decorativo ni intercambiable: **el ControlNet de segmentación aprendió esos colores exactos**. Si inventas tu propia paleta, el modelo interpretará mal las clases. La lista canónica está en la documentación de ControlNet y en los ejemplos de `diffusers`.

Aquí está el matiz que engaña: el mapa es de colores, pero **esos colores no son los colores de la escena**. Un mapa de segmentación con una pared en beige claro no significa que la pared vaya a salir beige; significa que ahí hay una pared. Es una etiqueta, no una muestra.

**Cuándo usarlo:** cuando necesitas mandar sobre el reparto del espacio pero no sobre la forma. Es la única familia que permite **editar el mapa a mano** de forma cómoda: pintar un rectángulo del color de «ventana» en una pared, y el modelo pondrá ahí una ventana. Para diseñar una escena desde cero, esa capacidad no la tiene ningún otro preprocesador.

**Aviso de compatibilidad:** como los normales, la segmentación está bien cubierta en SD 1.5 y floja en SDXL.

---

## Tile: el que no describe la estructura, la relee

`tile` es el raro de la familia y merece su propia sección porque no encaja en el esquema mental de los demás: **su mapa de condicionamiento es la propia imagen, sin transformar**.

No hay preprocesador. Le pasas un recorte de una imagen —o la imagen entera— y el ControlNet lo usa como referencia de «esto es lo que hay aquí» mientras el modelo regenera esa zona con detalle nuevo.

```python
controlnet = ControlNetModel.from_pretrained("xinsir/controlnet-tile-sdxl-1.0",
                                             torch_dtype=torch.float16)

imagen = pipe(
    prompt="silla de terciopelo verde, textura de tejido, macro, fotografía de producto",
    image=recorte_ampliado,        # el recorte tal cual, sin preprocesar
    controlnet_conditioning_scale=0.8,
).images[0]
```

Su uso principal es el **escalado con invención de detalle**: partes de una imagen pequeña, la amplías con un algoritmo cualquiera (que da un resultado borroso), la troceas en cuadrados y regeneras cada trozo con tile activo. Cada trozo mantiene su contenido pero gana textura real en vez de píxeles interpolados. Es lo que hay detrás de casi todos los flujos de *upscaling* de alta calidad.

Su segundo uso es **añadir detalle sin cambiar nada**: aplicar tile sobre la imagen completa con un peso medio y un prompt más descriptivo enriquece la textura conservando la composición.

Dos avisos:

- Es el control **más atado** de todos, por definición. No esperes que el prompt cambie el color o el material: tile impone la apariencia, no solo la geometría.
- La versión de referencia (`control_v11f1e_sd15_tile`) es de SD 1.5; para SDXL el equivalente habitual es `xinsir/controlnet-tile-sdxl-1.0`. Como siempre, la familia manda.

---

## Tabla de decisión

Lo que hay que consultar cuando no sabes cuál coger. Los identificadores son los de referencia en el momento de escribir esto; el catálogo se mueve, sobre todo en SDXL.

| Quiero conservar... | Preprocesador | ControlNet SD 1.5 | ControlNet SDXL |
|---|---|---|---|
| El contorno exacto | Canny | `lllyasviel/control_v11p_sd15_canny` | `diffusers/controlnet-canny-sdxl-1.0` |
| El contorno, con margen | HED / PiDiNet | `lllyasviel/control_v11p_sd15_softedge` | `xinsir/controlnet-scribble-sdxl-1.0` |
| El dibujo de línea | Lineart | `lllyasviel/control_v11p_sd15_lineart` | `TheMistoAI/MistoLine` |
| Solo la idea de la forma | Scribble | `lllyasviel/control_v11p_sd15_scribble` | `xinsir/controlnet-scribble-sdxl-1.0` |
| La perspectiva arquitectónica | MLSD | `lllyasviel/control_v11p_sd15_mlsd` | Escaso — usa un modelo *union* |
| El volumen y el espacio | Depth | `lllyasviel/control_v11f1p_sd15_depth` | `diffusers/controlnet-depth-sdxl-1.0` |
| El relieve de las superficies | Normal | `lllyasviel/control_v11p_sd15_normalbae` | Escaso — usa un modelo *union* |
| La pose de una figura | OpenPose / DWpose | `lllyasviel/control_v11p_sd15_openpose` | `xinsir/controlnet-openpose-sdxl-1.0` |
| El reparto del espacio por clases | Segmentación | `lllyasviel/control_v11p_sd15_seg` | Escaso |
| El contenido, para ampliar | (ninguno) | `lllyasviel/control_v11f1e_sd15_tile` | `xinsir/controlnet-tile-sdxl-1.0` |

Cuando en SDXL no encuentres el que buscas, mira los **ControlNet *union***, como `xinsir/controlnet-union-sdxl-1.0`: son un único modelo entrenado para aceptar varios tipos de mapa, se selecciona el modo al llamarlo y cubren huecos del catálogo con un solo juego de pesos en memoria.

Y una regla de oro para empezar: **si dudas entre dos, elige el que menos ate**. Es mucho más fácil subir el peso de un control laxo que rescatar una imagen que ha salido calcada.

## Cómo aparecen fuera de `diffusers`

- En **ComfyUI**, los preprocesadores son nodos propios que vienen en un paquete aparte (típicamente `comfyui_controlnet_aux`), colocados antes del nodo *Apply ControlNet*. La cadena se ve dibujada, con el mapa saliendo por un cable que puedes previsualizar.
- En **AUTOMATIC1111**, el desplegable *Preprocessor* está justo al lado del de *Model*, y la extensión los encadena por ti. Los nombres siguen una convención propia: `canny`, `softedge_hed`, `softedge_pidinet`, `lineart_realistic`, `depth_midas`, `depth_zoe`, `openpose_full`, `seg_ofade20k`, `tile_resample`. El botón de previsualización que muestra el mapa es la mejor costumbre que se puede coger allí. Y `none` como preprocesador significa «lo que te paso ya es un mapa».
- En las **APIs de proveedor**, el preprocesador suele ser un `preprocessorId` numérico de un catálogo cerrado y **lo ejecuta el proveedor**: le mandas la foto original, no el mapa. Esa es la diferencia que rompe las migraciones entre una API y `diffusers` — el código que funcionaba mandando la foto de repente tiene que mandar el mapa.

## Errores frecuentes

- **Preprocesar algo que ya es un mapa.** El error rey. Pasar un boceto a línea por Canny genera dos líneas por trazo y la imagen sale con todo doblado; pasar un mapa de profundidad por un estimador de profundidad devuelve algo sin sentido. Si la entrada ya es un mapa, va directa al modelo.
- **Mapa negro o casi vacío.** OpenPose que no detecta a nadie, Canny con los umbrales demasiado altos, MLSD sobre una escena sin rectas. La generación sale sin control y parece un problema del peso. Míralo.
- **Colores invertidos.** Los ControlNet de la familia de bordes esperan **líneas claras sobre fondo oscuro**. Un boceto escaneado es lo contrario: trazo negro sobre papel blanco. Hay que invertirlo (`ImageOps.invert`) antes de pasarlo.
- **Usar la paleta equivocada en segmentación.** Colores bonitos elegidos a mano en vez de la paleta ADE20K: el modelo interpreta clases al azar.
- **Olvidar normalizar un mapa de profundidad.** Sale casi todo negro y el control apenas actúa.
- **Dejar `detect_resolution` por defecto generando a 1024.** El mapa se calcula a 512 y se amplía; pierdes detalle fino sin enterarte.
- **Instanciar el detector dentro del bucle.** `HEDdetector.from_pretrained(...)` carga pesos. Hacerlo por cada imagen convierte un proceso de segundos en uno de minutos. Instáncialo una vez y reutilízalo.

## Buenas prácticas avanzadas

- **Genera los mapas de tres preprocesadores candidatos antes de generar ni una imagen.** Cuesta segundos y decide el resultado más que cualquier otra cosa que hagas después. Mirar el mapa de Canny, el de HED y el de depth uno al lado del otro te dice al instante cuál impone lo que quieres imponer; probar a ciegas y ajustar el peso es el camino largo.
- **Simplifica el mapa a mano cuando el preprocesador se pase de listo.** Un mapa es una imagen: puedes borrar de él lo que no quieres imponer. Pintar de negro el fondo en un mapa de Canny deja la silla atada y el fondo libre, y es más rápido y más controlable que buscar unos umbrales que hagan lo mismo. Poca gente cae en que el mapa es editable.
- **Elige el estimador de profundidad que se parezca al del entrenamiento.** Los ControlNet de profundidad de SD 1.5 se entrenaron con MiDaS, y alimentarlos con Depth Anything V2 —que es mejor estimador— puede dar resultados peores por desajuste de distribución: los mapas tienen otro contraste y otra escala. Cuando un depth se comporte raro, prueba el estimador de su época antes de tocar nada más.
- **Desconfía de los mapas «perfectos».** Un mapa que reproduce fielmente cada detalle de la imagen original es un mapa que no deja espacio al modelo, y la señal de alarma es una salida técnicamente correcta pero plana, sin personalidad, que parece la original coloreada. Cuando eso pase, el arreglo suele estar en el preprocesador (uno más laxo) y no en el peso.
- **Cachea los mapas.** En cuanto haces un barrido de prompts o de pesos sobre la misma imagen, el preprocesador se está ejecutando idénticamente una y otra vez. Guardar el mapa en disco y reutilizarlo quita un factor entero del tiempo de iteración, y de paso te obliga a mirarlo.

## Documentación oficial

- [Repositorio `lllyasviel/ControlNet-v1-1-nightly`](https://github.com/lllyasviel/ControlNet-v1-1-nightly) — la referencia del autor: para cada uno de los catorce modelos v1.1 dice qué preprocesador le corresponde, qué espera exactamente de entrada y cuáles son sus límites conocidos.
- [`controlnet_aux` en GitHub](https://github.com/huggingface/controlnet_aux) — el código de los preprocesadores, que es donde se ven los parámetros reales de cada detector y sus valores por defecto.
- [ControlNet en la documentación de `diffusers`](https://huggingface.co/docs/diffusers/using-diffusers/controlnet) — ejemplos completos por tipo de condicionamiento, incluida la paleta ADE20K de segmentación.

## Recursos didácticos

- [ControlNet 1.1 en el wiki de `sd-webui-controlnet`](https://github.com/Mikubill/sd-webui-controlnet/wiki/ControlNet-1.1) — el catálogo visual: entrada, mapa y salida para cada preprocesador. Es la mejor forma de decidir cuál usar sin instalar nada.
- [Demo de ControlNet v1.1 en Hugging Face Spaces](https://huggingface.co/spaces/hysts/ControlNet-v1-1) — sube tu propia imagen y compara los mapas en el navegador. Media hora aquí ahorra semanas de intuición mal calibrada.
- [Canny edge detector en Wikipedia](https://en.wikipedia.org/wiki/Canny_edge_detector) — de 1986 y explicado con figuras. Entender la histéresis de los dos umbrales hace que dejes de moverlos al azar.
- [Explorador del dataset ADE20K](https://groups.csail.mit.edu/vision/datasets/ADE20K/) — las 150 clases de la segmentación semántica y qué cubre cada una.

---

*En resumen: el preprocesador no captura tu imagen, la recorta — y lo que decide el resultado no es cuánta información conserva, sino cuál tira a propósito para dejársela al prompt.*
