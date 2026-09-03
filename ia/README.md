# IA — Guías

Cómo trabajar con modelos generativos desde el punto de vista de quien programa, en dos frentes.

En **texto**: qué información deben recibir los modelos de lenguaje (LLMs), cómo gestionarla cuando crece, cómo integrarlos en una aplicación que aguante producción y cómo usarlos para escribir código sin perder el control de lo que entra en el repositorio.

En **imagen generativa**: cómo funciona por dentro un modelo de difusión y cómo dejar de pedirle imágenes al azar para imponerle la estructura exacta que hace falta.

---

## Contenido

### [Ingeniería con LLMs](ingenieria-con-llms/README.md)
El recorrido completo en dos frentes: programar con IA (agentes de codificación, prompting, desarrollo dirigido por especificación, revisión de código generado y cómo empaquetar y distribuir las extensiones del agente con plugins y marketplaces) y construir software con LLMs (API, salidas estructuradas, tool use, agentes, MCP, evaluaciones, coste, seguridad).

### [Context Engineering](context-engineering/README.md)
Cómo diseñar y gestionar todo lo que recibe un LLM en cada llamada: ventana de contexto, RAG, memoria de agentes, compactación, aislamiento entre agentes y caché de *prompt*.

### [ControlNet](controlnets/README.md)
Imagen generativa: cómo funciona un modelo de difusión (latente, denoising, VAE, semilla, guidance scale) y cómo imponerle una estructura concreta con ControlNet — el catálogo de preprocesadores, la calibración del peso y la ventana de control, y cómo apilar varios sin degradar el resultado.
