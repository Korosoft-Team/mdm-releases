# Integración del actualizador en cada aplicación

> Complemento de [`../README.md`](../README.md). Esto es lo que hay que hacer **en cada repositorio de
> aplicación** (privados) para que este canal funcione.
> **Base:** documentación oficial del plugin, `https://v2.tauri.app/plugin/updater/`, consultada el
> 2026-10-07. Los nombres de configuración son los de Tauri v2.

Sustituir en todo el documento:

| Marcador | Estudiante | Docente | Gestor |
|---|---|---|---|
| `<APP>` | `estudiante` | `docente` | `gestor` |
| `<DIR>` | `app-estudiante` | `app-docente` | `app-gestor` |

---

## 1. Dependencia

```bash
cd <DIR>
pnpm tauri add updater
```

Eso agrega `tauri-plugin-updater` en `src-tauri/Cargo.toml`, el paquete de JavaScript y registra el
plugin. **Requiere Rust ≥ 1.90.**

Si además se quiere reiniciar la aplicación desde JavaScript tras instalar:

```bash
pnpm add @tauri-apps/plugin-process
```

---

## 2. `tauri.conf.json` — cuatro cambios

```jsonc
{
  "version": "0.2.0",                    // subir en CADA publicación: si no es mayor, no hay actualización
  "bundle": {
    "targets": ["nsis"],                 // hoy dice "all": eso genera .deb y .rpm, que no se actualizan
    "createUpdaterArtifacts": true,       // SIN ESTO NO SE GENERAN LOS ARTEFACTOS NI LAS FIRMAS
    "windows": {
      "nsis": {
        "installMode": "currentUser"      // decisión del 7 de octubre: por usuario, para no pedir administrador
      }
    }
  },
  "plugins": {
    "updater": {
      "pubkey": "dW50cnVzdGVkIGNvbW1lbnQ6IG1pbmlzaWduIHB1YmxpYyBrZXk6IDlCRTQ3RDVGRUJDRThCRDYKUldUV2k4N3JYMzNrbTEvWlhEUmxvam1LTXhlOTROUDdSTWJ6TzJhZkt0SjJnQmNoK2cxdzdsRXcK",
      "endpoints": [
        "https://korosoft-team.github.io/mdm-releases/<APP>/latest.json"
      ],
      "windows": {
        "installMode": "passive"
      }
    }
  }
}
```

Notas que evitan errores caros:

- **`pubkey` es el contenido de la clave, nunca una ruta a un archivo.**
- **`endpoints` debe ser un arreglo.** En producción el plugin **exige HTTPS**; para aceptar `http`
  habría que activar `dangerousInsecureTransportProtocol`, y no se va a hacer.
- Si hay varios endpoints, Tauri **pasa al siguiente solo si recibe un código distinto de 2XX**.
- **`createUpdaterArtifacts` es la causa número uno de «no se generan las firmas»** cuando alguien lo
  omite.

### Sobre `installMode` (Windows)

Son **dos ajustes distintos** y conviene no confundirlos:

| Ajuste | Dónde | Qué controla |
|---|---|---|
| `bundle.windows.nsis.installMode` | `bundle` | **Dónde** se instala: `currentUser` (por defecto), `perMachine` o `both` |
| `plugins.updater.windows.installMode` | `plugins.updater` | **Cómo** se ve la instalación de la actualización: `passive` (por defecto), `basicUi` o `quiet` |

### Decisión tomada el 7 de octubre de 2026

| Ajuste | Valor decidido | Motivo |
|---|---|---|
| `bundle.windows.nsis.installMode` | **`currentUser`** | Las actualizaciones **no piden permisos de administrador** en cada laptop |
| `plugins.updater.windows.installMode` | **`passive`** | Barra de progreso pequeña, sin intervención del docente |

`quiet` queda **descartado**: no puede pedir privilegios por sí solo, así que solo serviría en
instalaciones por usuario y a cambio pierde todo aviso visible para el docente.

---

## 3. Permisos (esto bloquea la API si se olvida)

El plugin expone sus comandos al frontend **solo si se concede el permiso**. En la capacidad
correspondiente:

```json
{
  "permissions": [
    "updater:default"
  ]
}
```

`updater:default` incluye `allow-check`, `allow-download`, `allow-install` y
`allow-download-and-install`.

---

## 4. Comprobación al arrancar

En el frontend, una vez al inicio. **Nunca en bucle**: el diseño es una petición al abrir la aplicación.

```ts
import { check } from '@tauri-apps/plugin-updater';
import { relaunch } from '@tauri-apps/plugin-process';

async function buscarActualizacion() {
  const update = await check();
  if (!update) return;
  // Avisar al usuario ANTES de descargar: son decenas de megabytes.
  await update.downloadAndInstall();
  await relaunch();
}
```

Dos cosas del comportamiento de Windows que hay que tener presentes:

- **Tauri cierra la aplicación antes de instalar la actualización**, porque el instalador necesita
  reemplazar archivos en uso. Si hay algo que guardar, se guarda antes: existe el gancho
  `on_before_exit` en Rust.
- **Descargar decenas de megabytes sin avisar es mala educación** en una laptop de aula. Conviene
  preguntar al usuario antes de descargar, o hacerlo al cerrar.

---

## 5. Dirección del servidor central: el otro desacople

El endpoint del actualizador **y** la dirección del servidor central tienen el mismo problema: hoy los
tres frontends hornean la dirección en tiempo de compilación (`import.meta.env.VITE_API_URL`), así que
cambiar de servidor obliga a recompilar.

El plugin del actualizador permite **sobrescribir el endpoint en tiempo de ejecución**:

```rust
use tauri_plugin_updater::UpdaterExt;

let update = app
    .updater_builder()
    .endpoints(vec![endpoint_del_canal])?
    .build()?
    .check()
    .await?;
```

**Se recomienda aplicar el mismo criterio a la dirección del servidor central**: leerla de un archivo de
configuración junto al ejecutable (o del directorio del usuario) al arrancar, y exponerla al frontend
desde Rust. Así, el día que LA CONTRATANTE encienda su VPS, **se cambia un valor y no se reinstala ni se
recompila nada**, y lo mismo vale para este canal cuando el repositorio se transfiera.

---

## 6. Comprobación de que quedó bien

En orden, y sin dar nada por bueno:

1. `pnpm tauri build --bundles nsis` con la clave privada en el entorno **genera un `.sig`** junto al
   `-setup.exe`. Si no aparece, falta `createUpdaterArtifacts`.
2. La release publicada contiene el `-setup.exe` y su `.sig`.
3. El endpoint del sitio devuelve un JSON con `windows-x86_64`, su `url` y su `signature`.
4. Instalar esa versión en una laptop, publicar una versión **mayor** y reabrir la aplicación:
   **se actualiza sola.** Hasta que eso pase, el canal no está probado.
