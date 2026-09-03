# Modelos de difusión

## ¿Qué es?

Un modelo de difusión es un generador de imágenes que no dibuja: **borra ruido**. Parte de una imagen de ruido aleatorio puro y la limpia poco a poco, en decenas de pasos, hasta que lo que queda es una fotografía de una silla de diseño en un salón nórdico. El texto que escribes no describe lo que hay que pintar: describe hacia qué se debe limpiar.

## ¿Por qué existe?

El planteamiento intuitivo para generar una imagen es que la red produzca los píxeles de una vez: entra un texto, sale un PNG. Es lo que intentaron las GAN durante años, y funcionaba a medias. El problema es que acertar un millón de píxeles coherentes entre sí en una sola pasada es un salto enorme, y las redes que lo intentan son inestables de entrenar: colapsan hacia unas pocas imágenes que engañan bien al discriminador y dejan de explorar.

La difusión cambia un problema imposible por muchos problemas fáciles. En lugar de pedirle a la red «dibuja una silla», se le pide cincuenta veces seguidas «esta imagen es una silla con un poco de ruido encima; ¿qué parte es ruido?». Cada pregunta es sencilla y la red la responde con un error pequeño. La suma de cincuenta correcciones pequeñas produce algo que ninguna corrección grande conseguía.

El entrenamiento va al revés y es casi trivial de montar: coges imágenes reales, les añades ruido gaussiano en cantidades conocidas, y entrenas a la red a **predecir el ruido que añadiste**. Como sabes exactamente cuánto pusiste, tienes la respuesta correcta gratis para cada ejemplo. No hace falta un discriminador, no hay dos redes peleándose, y el entrenamiento es estable.

> Si vienes de backend, piensa en la diferencia como la que hay entre una migración de base de datos monolítica que lo cambia todo en una transacción gigante y una serie de migraciones pequeñas y reversibles. La segunda no es más elegante por gusto: es la única que puedes depurar cuando falla.

La consecuencia práctica de esa arquitectura recorre toda esta colección: **la imagen se construye progresivamente**, y eso significa que hay momentos distintos dentro de la generación. En los primeros pasos se decide la composición general; en los últimos, la textura. Por eso más adelante tendrá sentido inyectar una señal de control solo durante una parte del proceso.

## ¿Cuándo y para qué se usa?

Siempre que quieras imágenes que no existen y no puedas o no quieras encargarlas: variaciones de un mockup de producto para probar paletas, ilustraciones para una entrada de blog, fondos de escena, texturas que se repiten, bocetos convertidos en renders presentables, o generación masiva de material visual para un catálogo.

Los escenarios que van a acompañarte durante toda la colección son cuatro, y son los que mejor exponen las limitaciones:

- Un **mockup de producto** —una silla— del que hay que sacar diez versiones con distinto material y color, pero exactamente con la misma forma.
- Una **ilustración de personaje** de la que hacen falta variantes de vestuario manteniendo la pose.
- Una **foto de interiorismo** de un salón cuya composición se quiere conservar mientras cambia el estilo entero.
- Un **boceto a lápiz** que hay que convertir en render.

Los cuatro tienen algo en común: no piden «una imagen bonita», piden «una imagen bonita **que respete esto**». Un modelo de difusión a secas no sabe hacer eso, y esa es exactamente la carencia que abre la puerta a [ControlNet](ControlNet.md).

---

## Cómo funciona: quitar ruido paso a paso

El proceso de generación se llama **denoising** (eliminación de ruido) y es un bucle. Simplificando, esto es lo que ocurre cuando pides una imagen:

1. Se sortea una imagen de **ruido aleatorio** a partir de una semilla.
2. Se repite N veces (típicamente entre 20 y 50):
   - La red mira el ruido actual, el texto y en qué paso vamos.
   - Predice **cuánto ruido** hay en esa imagen.
   - Un algoritmo aparte resta una fracción de ese ruido predicho.
3. Lo que queda se convierte en píxeles y se guarda.

Dos detalles que sorprenden la primera vez:

- **La red nunca ve la imagen final.** En cada paso solo ve un estado intermedio ruidoso y opina sobre el ruido. La imagen emerge del bucle, no de una llamada.
- **La red no decide cuánto restar.** Eso lo decide un componente distinto —el *scheduler*, que veremos en un momento—, y por eso cambiar de scheduler cambia el resultado sin tocar el modelo.

## Latent diffusion: por qué esto cabe en una GPU normal

Hacer el bucle anterior sobre píxeles es carísimo. Una imagen de 512×512 en color son 786.432 números, y hay que procesarlos entre 20 y 50 veces por imagen. Los primeros modelos de difusión hacían exactamente eso y necesitaban hardware de centro de datos.

**Latent diffusion** —la idea detrás de Stable Diffusion— añade una compresión previa. En lugar de difundir píxeles, se difunde una representación comprimida de la imagen, el **latente**. La compresión y la descompresión las hace un **VAE** (*Variational Autoencoder*), una red entrenada aparte que sabe hacer dos cosas:

- **Codificar** (`encode`): imagen de 512×512×3 → latente de 64×64×4.
- **Descodificar** (`decode`): latente de 64×64×4 → imagen de 512×512×3.

La cuenta explica por qué esto lo cambió todo:

| | Números que procesar |
|---|---|
| Píxeles, 512×512 RGB | 786.432 |
| Latente, 64×64×4 | 16.384 |

Son **48 veces menos**. El bucle de denoising, que es la parte cara, opera sobre la versión pequeña; el VAE solo se ejecuta una vez al principio (si partes de una imagen) y una vez al final. Por eso una GPU de consumo genera una imagen en segundos.

El precio de la compresión es real y conviene conocerlo: el latente no guarda todo. El VAE es quien decide qué información se conserva y qué se reconstruye «a ojo», y es el culpable habitual de que los textos pequeños salgan ilegibles, de que las caras a pocos píxeles se deformen y de que una imagen pierda algo de nitidez con solo codificarla y descodificarla sin generar nada.

Otra consecuencia práctica: **el latente mide una octava parte del lado de la imagen**, así que las dimensiones que pidas deben ser múltiplos de 8. Si pides 500×500, la librería lo redondea o protesta.

## Instalación y primer ejemplo de punta a punta

Toda la colección usa `diffusers`, la librería de Hugging Face, porque es donde la mecánica se ve sin capas de interfaz por encima.

```bash
pip install diffusers transformers accelerate torch --upgrade
```

- `diffusers` — los pipelines y los modelos de difusión.
- `transformers` — los codificadores de texto (CLIP), que `diffusers` usa por debajo.
- `accelerate` — carga los pesos en GPU de forma eficiente.
- `torch` — PyTorch. Con GPU NVIDIA conviene instalarlo desde el índice de CUDA que indique [pytorch.org](https://pytorch.org/get-started/locally/), no el genérico.

Un primer ejemplo completo, de texto a fichero:

```python
import torch
from diffusers import StableDiffusionXLPipeline

pipe = StableDiffusionXLPipeline.from_pretrained(
    "stabilityai/stable-diffusion-xl-base-1.0",
    torch_dtype=torch.float16,      # la mitad de memoria, calidad prácticamente idéntica
    variant="fp16",
).to("cuda")

imagen = pipe(
    prompt="silla de diseño escandinavo en madera clara, salón luminoso, fotografía de producto",
    negative_prompt="borroso, deformado, marca de agua, texto",
    num_inference_steps=30,
    guidance_scale=7.0,
    generator=torch.Generator("cuda").manual_seed(42),
).images[0]

imagen.save("silla.png")
```

La primera ejecución descarga unos 7 GB de pesos a la caché local (`~/.cache/huggingface`); las siguientes arrancan en segundos. Lo que devuelve `pipe(...)` es un objeto con una lista `images` de imágenes PIL, aunque hayas pedido una sola.

Cada uno de los parámetros de esa llamada es un término que el resto de la colección da por sabido. Vamos uno a uno.

---

## El vocabulario, término a término

### Latente

La imagen comprimida sobre la que trabaja el bucle. Es un tensor de números, no algo que puedas mirar: si lo guardas como PNG se ve como una miniatura de colores extraños. Cuando en cualquier documento leas «se parte del latente ruidificado de la imagen», significa que a esa versión comprimida se le ha sumado ruido.

Puedes tocarlo directamente, y ver el VAE en acción aclara mucho las ideas:

```python
from diffusers.utils import load_image

imagen = load_image("silla.png")
tensor = pipe.image_processor.preprocess(imagen).to("cuda", torch.float16)

latente = pipe.vae.encode(tensor).latent_dist.sample() * pipe.vae.config.scaling_factor
print(latente.shape)     # torch.Size([1, 4, 128, 128])  para una imagen de 1024×1024
```

El `128` es `1024 / 8`: el factor de compresión del VAE. Y los `4` son los canales del latente, que no son rojo, verde y azul — son cuatro dimensiones aprendidas que no tienen significado legible.

### U-Net

La red que hace la predicción de ruido en cada paso. Se llama así por su forma: una rama que **comprime** la información (el *encoder*, o rama descendente), un cuello de botella (*middle block*) y una rama que la **reconstruye** (el *decoder*, o rama ascendente), con conexiones directas —*skip connections*— entre los niveles equivalentes de las dos ramas.

Esa estructura importa mucho para entender ControlNet, así que quédate con tres ideas:

- El **encoder** ve la imagen a resoluciones cada vez más pequeñas: ahí se codifica la composición general.
- El **decoder** vuelve a subir de resolución y recupera el detalle.
- Las **skip connections** llevan información del encoder al decoder saltándose el cuello de botella. Son el punto exacto por el que ControlNet inyectará su señal.

La U-Net es el grueso del modelo: unos 860 millones de parámetros en Stable Diffusion 1.5 y unos 2.600 millones en SDXL.

### Paso de denoising (*step*)

Una iteración del bucle: una pasada de la U-Net más una resta de ruido. `num_inference_steps` fija cuántas hay.

Más pasos no es linealmente mejor. Por debajo de unos 15 la imagen sale sin resolver, con formas blandas; entre 20 y 30 está el punto habitual; por encima de 50 la diferencia rara vez se ve y el coste crece proporcionalmente. El número de pasos es el parámetro que más directamente multiplica el tiempo de generación: 40 pasos tardan el doble que 20.

### Scheduler (o *sampler*)

El algoritmo que decide cuánto ruido se resta en cada paso a partir de lo que predijo la U-Net. Es un componente **intercambiable y sin pesos entrenados**: cambiarlo cambia el resultado y la velocidad de convergencia sin tocar el modelo.

```python
from diffusers import DPMSolverMultistepScheduler

pipe.scheduler = DPMSolverMultistepScheduler.from_config(
    pipe.scheduler.config, use_karras_sigmas=True
)
```

`DPMSolverMultistepScheduler` (lo que en las interfaces gráficas aparece como *DPM++ 2M Karras*) da imágenes nítidas en 20-25 pasos y es un buen valor por defecto. `EulerAncestralDiscreteScheduler` inyecta algo de ruido nuevo en cada paso, lo que da resultados más variados pero **nunca converge del todo**: con más pasos sigue cambiando la imagen. El «ancestral» del nombre es justo eso, y por eso conviene evitar esa familia mientras calibras algo.

### Guidance scale (CFG)

En cada paso el modelo hace en realidad **dos** predicciones: una con tu prompt y otra sin él (o con el *negative prompt*). Luego las combina exagerando la diferencia:

```
predicción_final = sin_prompt + guidance_scale × (con_prompt − sin_prompt)
```

Eso es *classifier-free guidance*. `guidance_scale` mide cuánto se exagera:

| Valor | Efecto |
|---|---|
| 1.0 | Se ignora el prompt por completo (la diferencia no se amplifica). |
| 3-5 | Interpretación laxa, resultados más naturales y variados. |
| 7-8 | El rango habitual en Stable Diffusion. |
| 12+ | Imágenes saturadas, con contornos duros y colores quemados. |

Dos consecuencias prácticas: el CFG **duplica el coste** de cada paso, porque son dos pasadas de la U-Net (`diffusers` las agrupa en un solo lote, pero la memoria y el cómputo son los de dos); y subir el CFG para «que haga más caso» es la reacción equivocada casi siempre — un prompt que se ignora no se arregla gritándolo más fuerte.

> Ojo: en modelos *guidance-distilled* como Flux.1 [dev], el parámetro `guidance_scale` existe pero significa otra cosa y su rango útil está en torno a 3.5. No traslades los valores de Stable Diffusion.

### Prompt y negative prompt

El `prompt` es la descripción de lo que quieres. El `negative_prompt` es la descripción de lo que **no** quieres, y no es un filtro posterior: es literalmente el texto que se usa como predicción «sin prompt» en la fórmula del CFG. Por eso funciona — el modelo se aleja activamente de eso.

Un negative prompt razonable de partida para fotografía de producto: `"borroso, deformado, baja resolución, marca de agua, texto, firma"`. En la variante distilada de Flux no aplica igual, porque no ejecuta la rama negativa.

### Seed (semilla)

El número que determina el ruido inicial. **Misma semilla + mismo prompt + mismos parámetros + mismo modelo = misma imagen**, bit a bit.

```python
generator = torch.Generator("cuda").manual_seed(42)
```

Esta es, con diferencia, **la herramienta de trabajo más importante de toda la colección**. Si quieres saber qué hace un parámetro, fijas la semilla, generas, cambias **un solo** parámetro, vuelves a generar y comparas. Sin semilla fija estás comparando dos imágenes que se diferencian en el parámetro *y* en el azar, y no puedes atribuir el cambio a nada.

Una advertencia sobre la reproducibilidad: la semilla garantiza el mismo resultado en la **misma** máquina, con las mismas versiones de las librerías y el mismo backend de atención. Entre una GPU NVIDIA y una CPU, o entre dos versiones de PyTorch, la misma semilla puede dar imágenes ligeramente distintas. Sirve para comparar, no como identificador permanente de una imagen.

### Checkpoint (o modelo base)

El paquete de pesos entrenados: U-Net, VAE y codificador de texto. `stabilityai/stable-diffusion-xl-base-1.0` es un checkpoint; también lo son los miles de modelos afinados que publica la comunidad para un estilo concreto (fotorrealismo, ilustración, anime).

Lo importante para lo que viene: un checkpoint pertenece a una **familia arquitectónica** (SD 1.5, SDXL, Flux...). Los checkpoints de una misma familia son intercambiables entre sí y comparten todos los accesorios; los de familias distintas **no comparten nada**. Este punto reaparecerá en todas las fichas siguientes porque es la causa número uno de errores.

---

## Las tres operaciones básicas

### Text-to-image

Lo que ya has visto: el ruido inicial se sortea de la nada y el prompt es la única información sobre qué debe salir. Máxima libertad, control nulo sobre la composición.

### Image-to-image y su `strength`

Aquí se parte de una imagen existente. El pipeline la codifica a latente, le añade ruido y ejecuta el bucle desde un punto intermedio en lugar de desde el principio. Cuánto ruido se añade —y por tanto desde qué punto se arranca— lo decide `strength`, un valor entre 0 y 1:

```python
import torch
from diffusers import StableDiffusionXLImg2ImgPipeline
from diffusers.utils import load_image

pipe = StableDiffusionXLImg2ImgPipeline.from_pretrained(
    "stabilityai/stable-diffusion-xl-base-1.0", torch_dtype=torch.float16, variant="fp16",
).to("cuda")

mockup = load_image("mockup-silla.png")

imagen = pipe(
    prompt="silla tapizada en terciopelo verde esmeralda, fotografía de producto",
    image=mockup,
    strength=0.6,                   # el dial: cuánto se conserva de la entrada
    num_inference_steps=30,         # se ejecutan 30 × 0.6 = 18 pasos reales
    generator=torch.Generator("cuda").manual_seed(42),
).images[0]
```

Fíjate en el comentario del número de pasos: con `strength=0.6` y 30 pasos configurados, el bucle solo corre 18. Es una fuente clásica de confusión — con `strength` muy bajo el modelo apenas tiene pasos para hacer nada.

Y aquí está el detalle que hay que retener para el resto de la colección: **`strength` es un único dial que gobierna toda la información de la entrada a la vez**. La forma de la silla, su color, su textura y el fondo viajan todos por el mismo canal:

- `strength=0.3` → la silla mantiene la forma **y** sigue siendo del mismo color. Terciopelo verde ni de broma.
- `strength=0.8` → aparece el terciopelo verde, pero la silla ya no es la misma silla.

No hay ningún valor intermedio que dé «esta forma exacta, otro material». No es que haya que buscarlo mejor: **el parámetro no puede expresar esa petición**, porque no distingue estructura de apariencia. Ese es el problema concreto que resuelve [ControlNet](ControlNet.md).

### Inpainting

Regenerar solo una zona, delimitada por una máscara en blanco y negro: blanco es lo que se regenera, negro lo que se conserva.

```python
from diffusers import StableDiffusionXLInpaintPipeline

pipe = StableDiffusionXLInpaintPipeline.from_pretrained(
    "diffusers/stable-diffusion-xl-1.0-inpainting-0.1",
    torch_dtype=torch.float16, variant="fp16",
).to("cuda")

imagen = pipe(
    prompt="cojín de lino crudo sobre la silla",
    image=load_image("salon.jpg"),
    mask_image=load_image("mascara-cojin.png"),   # blanco = zona a regenerar
    strength=0.99,
).images[0]
```

Los checkpoints específicos de inpainting funcionan bastante mejor que usar uno normal con máscara, porque su primera capa acepta canales extra: además del latente ruidoso reciben el latente de la parte conservada y la propia máscara. Es la razón de que respeten los bordes en vez de dejar una costura visible.

---

## El ecosistema mínimo para orientarse

Tres familias cubren casi todo lo que te vas a encontrar. Lo que las separa no es solo la calidad: es la **arquitectura**, y de ahí que los accesorios no crucen de una a otra.

| Familia | Resolución nativa | Arquitectura | Nota |
|---|---|---|---|
| **Stable Diffusion 1.5** | 512×512 | U-Net (~860M), latente de 4 canales | Antigua, pero con el ecosistema de extensiones más grande y las exigencias de VRAM más bajas. |
| **SDXL** | 1024×1024 | U-Net (~2.6B), dos codificadores de texto | El estándar de facto para trabajo serio con ControlNet. |
| **Flux.1** | 1024×1024 | Transformer (MMDiT), no U-Net; latente de 16 canales | Mejor obediencia al prompt y texto legible. Al no haber U-Net, el mecanismo de las extensiones es distinto. |

Un aviso que ahorra tardes enteras: pedirle a SD 1.5 una imagen de 1024×1024 no da una imagen más grande, da una imagen **rota** —personajes duplicados, composiciones repetidas— porque se está usando fuera de la resolución con la que se entrenó. Y pedirle a SDXL 512×512 da una imagen pobre por el motivo simétrico. Genera en la resolución nativa de la familia y escala después.

## Cómo se extiende un modelo base sin reentrenarlo

Reentrenar un checkpoint desde cero está fuera del alcance de casi todo el mundo: son cientos de GPUs durante semanas. Lo habitual es partir de un modelo base congelado y añadirle algo encima. Las dos formas dominantes resuelven problemas distintos y **no compiten**:

- **LoRA** (*Low-Rank Adaptation*) — un fichero pequeño, de decenas de MB, que modifica sutilmente los pesos de la U-Net. Enseña al modelo **qué** dibujar o **con qué aspecto**: un estilo de ilustración concreto, un producto específico, un personaje recurrente. Se pueden cargar varios a la vez y darles un peso.

  ```python
  pipe.load_lora_weights("mi-usuario/estilo-producto-minimalista")
  pipe.fuse_lora(lora_scale=0.8)
  ```

- **ControlNet** — una red adicional que corre en paralelo a la U-Net y le dice **dónde** va cada cosa: qué silueta, qué profundidad, qué pose. No cambia el estilo ni el contenido; impone la geometría.

La distinción vale la pena memorizarla porque decide cuál necesitas:

| Lo que quieres imponer | Herramienta |
|---|---|
| Un estilo o un sujeto concreto | LoRA |
| Una composición, silueta o pose concreta | ControlNet |
| Una paleta o un «que se parezca a esta referencia» | IP-Adapter / referencias de estilo |
| Todo a la vez, a partir de una foto | image-to-image (con las limitaciones ya vistas) |

Ambas se pueden usar juntas, y de hecho es lo normal: un LoRA de estilo más un ControlNet de estructura es la combinación estándar para producir un catálogo coherente.

Las dos comparten una restricción: **se entrenan contra una familia arquitectónica concreta**. Un LoRA de SD 1.5 no carga en SDXL, y un ControlNet de SD 1.5 no funciona con SDXL. La ficha siguiente insiste en esto porque el error es tan frecuente como silencioso.

## Errores frecuentes

- **`RuntimeError: expected scalar type Half but found Float`** — has cargado el pipeline en `torch.float16` y le estás pasando un tensor propio en `float32`. Conviértelo con `.to(torch.float16)` antes.
- **`CUDA out of memory`** — antes de bajar la resolución, prueba `pipe.enable_model_cpu_offload()`, que va moviendo a la GPU solo el componente que se está usando. Cuesta algo de velocidad y ahorra varios GB.
- **La imagen sale gris, negra o con manchas de color** — casi siempre es el VAE de SDXL en `float16`, un problema conocido de desbordamiento numérico. Se arregla cargando `madebyollin/sdxl-vae-fp16-fix` como VAE, o pasando el VAE a `float32`.
- **Dos generaciones con la misma semilla dan imágenes distintas** — el `Generator` se consume. Hay que crearlo (o volver a llamar a `manual_seed`) antes de **cada** llamada, no una vez al principio del script.
- **Composiciones duplicadas: dos sillas, dos horizontes** — estás generando fuera de la resolución nativa de la familia. Revisa la tabla de arriba.

## Buenas prácticas avanzadas

- **Fija la semilla *y* el scheduler antes de comparar nada.** La semilla es la mitad del control; la otra mitad es que el scheduler sea determinista. Los ancestrales (`EulerAncestral`, cualquier nombre acabado en `a` en las interfaces) reinyectan ruido en cada paso y no convergen: dos ejecuciones con 25 y 30 pasos dan imágenes distintas aunque todo lo demás sea idéntico. Con `DPMSolverMultistep` o `UniPC`, la única variable eres tú.
- **Genera en lote, no de una en una.** Pasar una lista de prompts o `num_images_per_prompt=4` aprovecha la GPU mucho mejor que cuatro llamadas seguidas, porque el coste dominante es mover los pesos, no calcular. Cuatro imágenes en un lote tardan bastante menos que cuatro imágenes sueltas.
- **Desconfía del CFG alto como solución a un prompt ignorado.** Cuando el modelo no hace caso a una parte del prompt, subir `guidance_scale` de 7 a 14 quema los colores y rara vez arregla nada: los conceptos que el modelo no separa bien no se separan por amplificación. Reordena el prompt poniendo delante lo importante, o impón esa parte por otra vía (un LoRA, un ControlNet).
- **Reutiliza los componentes entre pipelines en lugar de recargar pesos.** `StableDiffusionXLImg2ImgPipeline.from_pipe(pipe_txt2img)` crea el pipeline de image-to-image compartiendo la U-Net, el VAE y los codificadores ya cargados. Sin eso, tener text-to-image e image-to-image en el mismo proceso duplica el consumo de VRAM sin necesidad.
- **Guarda los parámetros junto a la imagen.** Semilla, prompt, negative, scheduler, pasos, CFG y checkpoint exacto. Una imagen buena de la que no sabes reproducir la configuración es una imagen que no puedes iterar, y en cuanto pasas de una docena de pruebas ya no te acuerdas. Un JSON al lado del PNG, o los metadatos del propio PNG, bastan.

## Documentación oficial

- [Documentación de `diffusers`](https://huggingface.co/docs/diffusers) — la referencia de la librería. Las secciones de *pipelines* y de *schedulers* son las dos que se consultan de verdad.
- [*High-Resolution Image Synthesis with Latent Diffusion Models*](https://arxiv.org/abs/2112.10752) (Rombach et al., 2022) — el paper de latent diffusion, el que introduce el VAE y el espacio latente que hacen viable todo esto.
- [*Denoising Diffusion Probabilistic Models*](https://arxiv.org/abs/2006.11239) (Ho et al., 2020) — el trabajo fundacional de los modelos de difusión modernos, para cuando la duda sea «¿pero qué se está entrenando exactamente?».

## Recursos didácticos

- [*The Illustrated Stable Diffusion*](https://jalammar.github.io/illustrated-stable-diffusion/) de Jay Alammar — el recorrido visual pieza a pieza. Si algo de esta ficha no ha terminado de encajar, aquí lo ves dibujado.
- [Diffusion Explainer](https://poloclub.github.io/diffusion-explainer/) — herramienta interactiva que muestra cómo evoluciona la imagen paso a paso y cómo cambia al mover el prompt y el guidance scale. Media hora aquí vale por varias lecturas.
- [*What are Diffusion Models?*](https://lilianweng.github.io/posts/2021-07-11-diffusion-models/) de Lilian Weng — el mismo contenido con las matemáticas delante, para cuando quieras el porqué formal.
- [Curso de difusión de Hugging Face](https://huggingface.co/learn/diffusion-course) — gratuito y con cuadernos ejecutables, si prefieres aprender tocando.

---

*En resumen: un modelo de difusión no dibuja una imagen, la desentierra del ruido en decenas de pasos pequeños — y como el proceso es progresivo y comprimido, se puede intervenir por dentro en vez de conformarse con el resultado.*
