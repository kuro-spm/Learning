# ControlNet — Guía de control estructural en modelos de difusión

Cómo dejar de pedirle imágenes a un generador y empezar a imponerle **qué forma tienen que tener**: recolorear el mockup de un producto conservando su silueta exacta, sacar variaciones de una ilustración manteniendo la pose del personaje, convertir un boceto en render, o cambiar el estilo de una foto de interiorismo sin mover un solo mueble de sitio.

Está pensada para perfiles backend que programan a diario y puede que no hayan tocado nunca un modelo de difusión. No presupone nada: la primera ficha construye el vocabulario entero —latente, U-Net, paso de denoising, guidance scale, semilla— y a partir de ahí cada concepto se define antes de usarse.

Los ejemplos de código van en **`diffusers`** (Python), porque es donde la mecánica se ve sin capas de interfaz por encima. Pero lo que se explica aquí es la técnica, no la librería: cada ficha traduce sus parámetros a los grafos de ComfyUI, a la pestaña de AUTOMATIC1111 y a las APIs de proveedor, donde ControlNet se reduce a un array de parámetros en un JSON y no se ve que debajo hay una red entrenada.

Los cuatro escenarios de arriba —la silla, el personaje, el boceto y el salón— se usan de principio a fin en todas las fichas, para que las piezas encajen entre documentos.

---

## Orden de lectura recomendado

### 1. La rampa de entrada

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Modelos de difusión](Modelos-de-Difusion.md) | Cómo se genera una imagen quitando ruido paso a paso, qué es el latente y qué significa cada parámetro que verás repetido en el resto de la colección. **Si ya trabajas con difusión, sáltatela y empieza por la 2.** |

### 2. El mecanismo

Qué es ControlNet, cómo se acopla a un modelo congelado y por qué no es lo mismo que image-to-image.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [ControlNet](ControlNet.md) | El concepto completo: el segundo canal de entrada, la copia entrenable con zero convolutions, el primer ejemplo que funciona de punta a punta y el contraste con image-to-image que resuelve la confusión más común. |

### 3. Las decisiones que determinan el resultado

Con el mecanismo entendido, tres fichas de trabajo. Se leen en orden porque cada una asume la anterior, pero también funcionan como referencia suelta.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [Preprocesadores](Preprocesadores.md) | La decisión más importante de todo el flujo: el catálogo completo —Canny, soft edge, lineart, scribble, MLSD, depth, normales, OpenPose, segmentación, tile— con qué captura cada uno, qué tira a la basura y cuándo elegir cuál. |
| 4 | [Peso y ventana de control](Peso-y-Ventana-de-Control.md) | Los dos diales: cuánto manda la estructura y durante qué parte del proceso. Incluye los modos de prioridad y el protocolo de calibración con semilla fija y un dial cada vez. |
| 5 | [Multi-ControlNet](Multi-ControlNet.md) | Apilar varios: cómo se suman los pesos, qué combinaciones funcionan, cuáles se pelean y por qué apilar de más degrada el resultado y duplica el coste. |

---

## Las cuatro cosas que más tiempo hacen perder

Si solo te llevas cinco líneas de toda la colección, que sean estas.

| Error | Qué pasa | Dónde se explica |
|---|---|---|
| Usar un ControlNet de SD 1.5 con un checkpoint SDXL (o al revés) | Error de dimensiones, o peor: imágenes sucias sin explicación | [ControlNet](ControlNet.md) |
| Pasar la foto original en vez del mapa de condicionamiento | `diffusers` no preprocesa por ti; las interfaces gráficas sí | [ControlNet](ControlNet.md) |
| Preprocesar algo que ya era un mapa | Cada trazo se convierte en dos y sale todo doblado | [Preprocesadores](Preprocesadores.md) |
| Dejar el peso en su valor por defecto (`1.0`) | La imagen sale calcada y el prompt se ignora | [Peso y ventana de control](Peso-y-Ventana-de-Control.md) |

Y el hábito que evita la mayoría de las sesiones largas de depuración: **guarda el mapa de condicionamiento en disco y ábrelo antes de generar**. Es la única prueba de qué está viendo realmente el modelo.

---

> Piezas relacionadas en otras colecciones: [Context Engineering](../context-engineering/README.md) para cómo se diseña lo que recibe un modelo de lenguaje en cada llamada, e [Ingeniería con LLMs](../ingenieria-con-llms/README.md) para integrar modelos en una aplicación de producción — coste, evaluación, seguridad y consumo de APIs de proveedor.
