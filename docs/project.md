# Project Architecture

This document summarizes how the Raytracer workspace is structured and how data flows from scene files to final pixels.

## Workspace layout

- `src/raytracer`: core crate (library + binary entrypoint)
- `src/primitives/*`: primitive plugins (`cdylib`)
- `src/materials/*`: material plugins (`cdylib`)
- `src/lights/*`: light plugins (`cdylib`)

The binary crate orchestrates scene loading, plugin instantiation, and rendering. Plugin crates implement extension points and are loaded dynamically from `plugins/`.

## End-to-end runtime flow

1. **CLI + scene input** (`src/raytracer/main.rs`)
   - Reads scene path from arguments.
   - Loads file content and runs preprocessor (`scene/preprocessor`).
2. **Preprocessing**
   - Expands `imports` and `components`.
   - Resolves `props` and `@references`.
   - Composes nested transforms into concrete per-primitive transforms.
3. **Deserialization**
   - Converts preprocessed RON into `SceneDesc` (`primitives`, `lights`, `camera`, `ambient`).
4. **Dynamic plugin creation**
   - `PluginLoader` loads libraries from `plugins/` and calls exported `create` symbols.
   - Produces runtime trait objects:
     - `Primitive = Box<dyn CanHit + Send + Sync>`
     - `Material = Box<dyn CanShade + Send + Sync>`
     - `Light = Box<dyn CanLight + Send + Sync>`
5. **Scene assembly**
   - Runtime scene stores objects as `(Primitive, Material, Transform)` tuples.
6. **Rendering**
   - `Renderer::from_scene` builds a BVH over object AABBs.
   - `Renderer::render` casts rays tile-by-tile in parallel, performs shadow checks, shades hits, and writes RGB output.
7. **Output**
   - Writes `image.ppm`.
   - Optionally starts SFML preview when built/used with preview support.

## Plugin contract and naming

Each plugin must export:

```rust
#[unsafe(no_mangle)]
pub fn create(cfg: &ron::Value) -> ...
```

`PluginLoader` uses strict file naming:

- primitive: `libraytracer_<kind>.<ext>`
- material: `libraytracer_material_<kind>.<ext>`
- light: `libraytracer_light_<kind>.<ext>`

where `<ext>` is platform-dependent (`so`, `dylib`, `dll`).

## Rendering notes

- BVH traversal is used for both closest-hit and shadow occlusion queries.
- Rendering is tile-based (`16x16`) and distributed across available CPU threads.
- Ambient intensity is applied as a post-shade scalar at the renderer level.
