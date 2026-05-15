# Raytracer

A modular and extensible raytracer written in Rust. The project uses dynamic plugins for primitives, materials, and lights so new rendering elements can be added without rebuilding the core crate.

## Features

- **Modular Architecture**: Cargo workspace with dynamic libraries (`.so` / `.dylib` / `.dll`) loaded at runtime.
- **Primitives**: Sphere, Plane, Cylinder, Cone, Cube, OBJ, Tetrahedron, and Torus.
- **Lights**: Point Light, Directional Light.
- **Materials**: Flat Color.
- **Configuration Engine**: RON scene loading with pre-processing (`imports`, reusable `components`, and property references).
- **SFML Integration**: Optional preview window on supported builds/platforms.
- **Hot-reload helper**: `make watch` reruns rendering after scene changes.

## Prerequisites

- [Rust](https://www.rust-lang.org/tools/install) (Cargo)
- GNU Make
- [SFML](https://www.sfml-dev.org/) (required only for SFML preview builds)

## Building the Project

Build the workspace and copy plugin dynamic libraries into `plugins/`:

```bash
make all
```

The runtime loader resolves plugins from `plugins/`, so this copy step is required.

## Running the Raytracer

Run with the default scene:

```bash
make run
```

On macOS, this renders to `image.ppm` without SFML preview.

On non-macOS, `make run` enables `sfml-preview` and passes `--sfml`.

You can also run directly with Cargo:

```bash
cargo run --release -p raytracer -- ./config.ron
```

For SFML preview (non-macOS):

```bash
cargo run --release -p raytracer --features sfml-preview -- ./config.ron --sfml
```

### Development checks

```bash
cargo fmt --all -- --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test --verbose
```

Run tests for one crate:

```bash
cargo test -p raytracer
```

Run a single test by name filter:

```bash
cargo test -p raytracer <test_name> -- --exact
```

### Watch mode

If you are editing a scene and want to see the changes update automatically, you can use the watch mode:

```bash
make watch
```
This helper script uses `inotifywait` and is intended for Linux-like environments.

## Scene Configuration

Scenes are written in RON. The pipeline supports:
- direct primitive/light/material definitions;
- reusable component declarations via `components`;
- external component imports via `imports`;
- property substitution via `@references`.

Examples included in the repository:
- `config.ron`
- assets in `assets/` (e.g., `car.ron`, `city.ron`, `house.ron`, `color.ron`)

## Architecture Overview

1. `src/raytracer/main.rs` reads a scene file, preprocesses it, and deserializes into scene descriptors.
2. `PluginLoader` dynamically resolves primitive/material/light `create` symbols from the `plugins/` directory.
3. The scene is converted into runtime objects (`Primitive`, `Material`, `Light`) and fed to `Renderer`.
4. `Renderer` builds a BVH, performs multithreaded tile-based ray casting, computes shading/shadows, and writes `image.ppm` (or displays via SFML preview when enabled).

## Documentation

This project uses [Doxygen](https://www.doxygen.nl/) for generated documentation.

Generate docs with:
```bash
doxygen Doxyfile
```

## Cleaning

```bash
make clean
```

To fully remove `target/`:
```bash
make fclean
```
