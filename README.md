# Casa virtual — Svelte + Three.js

Visor web de una casa en primera persona, con controles tipo FPS y colisiones contra la geometría del modelo.

## Ejecutar

```bash
npm install
npm run dev
```

Abre la dirección que muestra Vite. Para generar la versión publicable, ejecuta `npm run build`.

## Preparar y exportar desde Blender

Usa **glTF 2.0 Binary (`.glb`)**. Es el formato adecuado para WebGL/Three.js porque puede contener mallas, materiales PBR y texturas en un único archivo, sin rutas externas rotas.

Antes de exportar:

1. Trabaja en metros: **Scene Properties → Units → Metric**, con `Unit Scale: 1.0`. Un muro de 2.60 m debe medir 2.60 unidades en Blender.
2. Mantén las paredes, losas, escaleras, puertas y barandas como mallas con volumen real. Evita paredes de un solo plano: deben tener espesor para que la colisión funcione desde ambos lados.
3. En Object Mode, selecciona las mallas arquitectónicas y usa **Ctrl+A → Rotation & Scale**. Comprueba que la escala quede en `1, 1, 1`.
4. Aplica los modificadores que formen la geometría final (por ejemplo, Array, Mirror o Boolean) si no quieres depender de su evaluación en tiempo de ejecución.
5. Crea un Empty: **Add → Empty → Plain Axes**, nómbralo exactamente `Spawn` y colócalo donde comienza el usuario, a `1.65 m` sobre el suelo. No hace falta orientar la cámara todavía.
6. Para materiales futuros, haz UV unwrap de cada malla visible y usa materiales **Principled BSDF**. Lo más compatible es Base Color, Metallic, Roughness, Normal y Emission. Guarda las texturas como PNG/JPG/WebP y empácalas en el `.glb` al exportar.

En **File → Export → glTF 2.0**, usa:

| Opción | Valor |
| --- | --- |
| Format | `glTF Binary (.glb)` |
| Include | `Selected Objects` si seleccionaste solo la casa; de lo contrario, toda la escena sin cámaras/lámparas auxiliares |
| Transform | `+Y Up` (valor estándar del exportador) |
| Geometry → Apply Modifiers | Activado |
| Geometry → UVs / Normals / Tangents | Activados |
| Materials | `Export` |
| Images | `Automatic` o `WebP` si priorizas peso y el público usa navegadores actuales |
| Animation / Cameras / Punctual Lights | Desactivado, salvo que realmente los necesites |

Exporta el archivo como `casa.glb` y cópialo a `public/models/casa.glb`. Después reinicia `npm run dev` si el servidor ya estaba abierto.

## Rendimiento

- Para una vivienda, intenta que el GLB inicial no supere 15–25 MB. Comprime texturas a 1K o 2K; 4K solo para superficies muy cercanas.
- Evita miles de objetos repetidos: instancia o combina elementos estáticos cuando sea razonable.
- El visor deja el color y materiales del GLB intactos y habilita sombras. Al llegar las texturas, no será necesario cambiar código.
- Valida el archivo antes de publicar en [glTF Viewer de Khronos](https://gltf-viewer.donmccurdy.com/). Si se ve correctamente allí, es una buena señal para el visor.

## Controles

- `W A S D`: caminar
- Mouse: mirar
- `Shift`: correr
- `Espacio`: saltar
- `Esc`: liberar el cursor

Mientras no exista `public/models/casa.glb`, la aplicación muestra una maqueta gris de prueba con colisiones.
