# Replicar el actualizador en otra aplicación

> **El ejemplo probado está en `mdm-estudiante`, PR #316**, en
> `.github/workflows/publish.yml`. Este documento dice qué copiar y qué cambiar; **no hay dos copias
> que puedan divergir**. La plantilla suelta que existía antes en `templates/` se retiró por eso mismo.

Las tres aplicaciones comparten el mismo canal (`mdm-releases`) y el mismo par de claves. Lo único que
cambia por aplicación es **el identificador, el nombre del tag y la ruta del endpoint**.

## Lo que hay que copiar

### 1. En `app-<x>/src-tauri/Cargo.toml`

```toml
tauri-plugin-updater = "2"
```

### 2. Un módulo `src/actualizaciones.rs`

Copiar `app-estudiante/src-tauri/src/actualizaciones.rs` tal cual, **cambiando dos cosas**:

- El endpoint del canal: `.../mdm-releases/<x>/latest.json`.
- Las dos pruebas, que llevan el endpoint y el nombre del canal escritos.

### 3. En `src/lib.rs`

```rust
mod actualizaciones;
// ...
        .plugin(tauri_plugin_updater::Builder::new().build())
        .setup(|app| {
            // ...
            actualizaciones::buscar_actualizacion(app.handle().clone());
            Ok(())
        })
```

### 4. En `app-<x>/src-tauri/tauri.conf.json`

```jsonc
"bundle": {
  "targets": "all",                    // NO fijar en ["nsis"]: rompe el build en Linux
  "createUpdaterArtifacts": true,      // sin esto no hay artefactos ni firmas
  "windows": { "nsis": { "installMode": "currentUser" } }
},
"plugins": {
  "updater": {
    "pubkey": "<la misma clave pública de las tres>",
    "endpoints": ["https://korosoft-team.github.io/mdm-releases/<x>/latest.json"],
    "windows": { "installMode": "passive" }
  }
}
```

### 5. El workflow `.github/workflows/publish.yml`

Copiar el de `mdm-estudiante` y cambiar:

| Qué | Estudiante | Docente | Gestor |
|---|---|---|---|
| Tag | `estudiante-v*` | `docente-v*` | `gestor-v*` |
| `working-directory` | `app-estudiante` | `app-docente` | `app-gestor` |
| Carpeta del canal | `sitio/estudiante` | `sitio/docente` | `sitio/gestor` |
| Nombre de la release | `estudiante-v$v` | `docente-v$v` | `gestor-v$v` |

**No cambiar**: las acciones ancladas por SHA, el paso que comprueba que la etiqueta y
`tauri.conf.json` digan la misma versión, ni el paso que **falla si no se generó el `.sig`**.

### 6. Los secretos del repositorio

`TAURI_SIGNING_PRIVATE_KEY`, `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` y `RELEASES_TOKEN` (token de grano
fino con `contents: write` **solo** sobre `mdm-releases`).

## Lo que **no** hay que tocar

- **El paquete de JavaScript del updater y el permiso `updater:default`**: el ejemplo funciona solo
  desde Rust. Hacen falta únicamente si algún día se quiere preguntar al usuario desde la interfaz
  antes de descargar.
- **Los archivos `gen/schemas/`**: se regeneran solos al compilar. Ya están versionados en los
  repositorios, así que el cambio aparece en el diff y se confirma junto con lo demás.

## Comprobación, en orden

1. `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` y `cargo test` en limpio.
2. El YAML del workflow, validado con un analizador (`npm install yaml` y `yaml.parse` sirve).
3. Etiqueta de prueba: el runner debe producir el `-setup.exe` **con su `.sig`**.
4. Instalar esa versión, publicar una **mayor** y reabrir: **se actualiza sola.** Hasta que eso pase,
   el canal no está probado.
