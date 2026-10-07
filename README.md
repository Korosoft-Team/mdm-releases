# Canal de distribución y actualizaciones — MDM MultiCulture.Play

> **Qué es:** repositorio **público de solo artefactos**: los instaladores y los metadatos de
> actualización de las aplicaciones de la suite. **No contiene código fuente.**
> **Por qué es público:** el actualizador descarga desde aquí **sin credenciales**. Así ningún token
> viaja dentro de las apps instaladas, que es el riesgo que se quiso evitar.
> **Contrato:** Cláusula Tercera 3.3 (instalación en los equipos provistos por LA CONTRATANTE) y
> Cláusula Octava (autoría y cesión de derechos).
> **Créditos y condiciones de uso:** [`AVISO-CREDITOS.md`](./AVISO-CREDITOS.md).

---

## 1. Plataforma de entrega: Windows

**Los equipos de LA CONTRATANTE son Windows.** Esa es la plataforma que se entrega y la única que el
canal necesita publicar.

Linux (AppImage) se mantiene documentado **solo para el entorno de desarrollo** de quien programa desde
Linux. No forma parte de la entrega y no hay obligación de publicarlo.

| Aplicación | Identificador | Código fuente | Endpoint |
|---|---|---|---|
| **Estudiante** | `com.koroSoft.mdm.estudiante` | privado: `mdm-estudiante` | `https://korosoft-team.github.io/mdm-releases/estudiante/latest.json` |
| **Docente** (visor) | `com.koroSoft.mdm.docente` | privado: `mdm-docente` | `https://korosoft-team.github.io/mdm-releases/docente/latest.json` |
| **Gestor de contenido** | `com.koroSoft.mdm.gestor` | privado: `mdm-gestor` | `https://korosoft-team.github.io/mdm-releases/gestor/latest.json` |

Los tres repositorios de código son **privados** y permanecen en custodia del equipo de desarrollo
hasta la suscripción del acta de cesión (Cláusula Octava 8.4). **Este repositorio no contiene código.**

---

## 2. Cómo se instalan las aplicaciones

1. Entrar a la pestaña **Releases** de este repositorio.
2. Elegir la versión más reciente de la aplicación que corresponda.
3. Descargar el archivo terminado en **`-setup.exe`** (instalador NSIS).
4. Ejecutarlo e instalar.

**El instalador NSIS es obligatorio, y no es una preferencia:** el actualizador de Tauri en Windows
**reutiliza ese mismo instalador**. Un paquete `.deb` o `.rpm` (Linux) o un `.msi` no aportan nada aquí,
y en Linux solo la AppImage se actualiza sola.

> **Aviso de Windows SmartScreen.** Los instaladores **no están firmados con certificado Authenticode**,
> así que Windows mostrará el aviso «Windows protegió su PC». Se continúa con *Más información* →
> *Ejecutar de todas formas*. **Esto debe quedar escrito en la guía de instalación y explicado en la
> capacitación**, para que el docente no lo tome por un defecto. La firma **minisign** de este canal
> (sección 4) es otra cosa: verifica las actualizaciones, **no** elimina ese aviso.

---

## 3. Cómo funciona la actualización

1. Al abrirse, la aplicación pide su archivo `latest.json`.
2. Si la versión publicada es mayor que la instalada, descarga el instalador nuevo.
3. **Verifica la firma** contra la clave pública incrustada en la aplicación. Si no coincide,
   **no se instala nada**.
4. En Windows, Tauri **cierra la aplicación** antes de instalar la actualización; se aplica al volver a
   abrirla.

Propiedades de este diseño, dichas explícitamente:

- **No hay sondeo continuo**: es una petición al abrir la aplicación.
- **Si el endpoint no responde, la aplicación sigue funcionando** con la versión que tiene. El
  actualizador nunca bloquea una aplicación.
- **La firma es obligatoria y no se puede desactivar.** Un artefacto manipulado no se instala.
- **Exige HTTPS** (GitHub Pages lo es). Se puede desactivar esa exigencia con
  `dangerousInsecureTransportProtocol`, y **no se va a hacer**.
- **Requiere Rust ≥ 1.90** en el entorno de compilación.

---

## 4. Clave de firma

**Clave pública** — pública por diseño; va en `plugins.updater.pubkey` de cada `tauri.conf.json`, y
debe ser **el contenido**, nunca una ruta a un archivo:

```
dW50cnVzdGVkIGNvbW1lbnQ6IG1pbmlzaWduIHB1YmxpYyBrZXk6IDlCRTQ3RDVGRUJDRThCRDYKUldUV2k4N3JYMzNrbTEvWlhEUmxvam1LTXhlOTROUDdSTWJ6TzJhZkt0SjJnQmNoK2cxdzdsRXcK
```

**Clave privada** — la custodia el equipo de desarrollo y **nunca** entra a este repositorio ni al
historial de ningún repositorio. Vive en los secretos de CI de cada repositorio de aplicación:
`TAURI_SIGNING_PRIVATE_KEY` y `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`. **Un archivo `.env` no sirve**: la
clave tiene que estar en el entorno.

> **Rotar la clave obliga a recompilar y reinstalar las aplicaciones**, porque la pública está incrustada
> en el binario. Se genera una vez y se guarda como corresponde: es una puerta de un solo sentido.

---

## 5. Qué hay que configurar en cada aplicación

Antes de que este canal sirva para algo, cada aplicación necesita seis cosas. El detalle exacto, con los
nombres de archivo y las rutas, está en [`docs/integracion-app.md`](./docs/integracion-app.md).

1. El plugin `tauri-plugin-updater` como dependencia.
2. **`bundle.createUpdaterArtifacts: true`** — sin esto **no se generan los artefactos de actualización
   ni sus firmas**.
3. `bundle.targets` con **`["nsis"]`** (hoy está en `"all"` en las tres apps).
4. `plugins.updater.pubkey` y `plugins.updater.endpoints`.
5. El permiso **`updater:default`** en las capacidades, o la API queda bloqueada.
6. La comprobación al arrancar, con su aviso al usuario.

---

## 6. Cómo se publica una versión

**Secretos necesarios en cada repositorio de aplicación:**

| Secreto | Para qué |
|---|---|
| `TAURI_SIGNING_PRIVATE_KEY` | firmar el artefacto |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | contraseña de la clave privada |
| `RELEASES_TOKEN` | token de grano fino con `contents: write` **solo** en este repositorio: es lo que permite subir el artefacto desde un repositorio privado. El `GITHUB_TOKEN` **no** cruza repositorios |

**Pasos (plantilla en [`templates/publish-app.yml`](./templates/publish-app.yml)):**

1. **Subir la versión en `tauri.conf.json`** y etiquetar. La versión publicada debe ser **mayor** que la
   instalada, o el actualizador no hará nada.
2. El workflow compila en `windows-latest` con `tauri build --bundles nsis`.
3. Con la clave en el entorno, Tauri genera el instalador y **su `.sig`**.
4. Se crea la release en **este** repositorio con el `-setup.exe` y su `.sig`.
5. Se actualiza `updates/<app>/latest.json` en la rama del sitio.

> **Estado de esta plantilla: no ejecutada todavía.** No se considera válida hasta que exista una release
> real con su `latest.json` y una instalación que se haya actualizado sola.

---

## 7. Formato de `latest.json`

Un archivo por aplicación, en la rama publicada. **Solo la clave de Windows**, que es lo que se entrega:

```json
{
  "version": "0.2.0",
  "notes": "Qué cambia en esta versión, en español neutro",
  "pub_date": "2026-10-08T12:00:00Z",
  "platforms": {
    "windows-x86_64": {
      "signature": "<contenido literal del archivo .sig>",
      "url": "https://github.com/Korosoft-Team/mdm-releases/releases/download/<app>-v0.2.0/<app>_0.2.0_x64-setup.exe"
    }
  }
}
```

Reglas que la propia documentación de Tauri impone y conviene no olvidar:

- `signature` es **el contenido** del `.sig`, no su ruta ni una URL.
- **Tauri valida el archivo completo antes de mirar la versión**: si se agrega una plataforma, tiene que
  estar completa y correcta. Por eso, entregando solo Windows, **se publica solo `windows-x86_64`**.
- Si en algún momento se agrega Linux para el entorno de desarrollo, su entrada debe ser válida: la
  AppImage y su `.AppImage.sig`.
- Obligatorios: `version`, `platforms.[target].url` y `platforms.[target].signature`.

---

## 8. Nota para el traspaso

Cuando la UNMSM asuma la operación, este repositorio debe pasar a su cuenta. **Al transferirlo cambia la
URL del sitio** (`<nuevo-propietario>.github.io/...`), y el endpoint está incrustado en cada aplicación.
Dos caminos aceptables:

1. **Dominio propio para el sitio** (lo recomendable cuando la UNMSM tenga su dominio: la Cláusula Quinta
   5.6 ya prevé que asuma hosting y dominio).
2. **Sobrescribir el endpoint en tiempo de ejecución** desde el código Rust de cada aplicación
   (`app.updater_builder().endpoints(...)`), igual que la dirección del servidor central.

El tercer camino —recompilar y reinstalar— hay que evitarlo. **Decisión pendiente y anotada a
propósito**: todavía no se ha fijado cuál se usa, y se resuelve junto con la dirección del servidor
central, que tiene el mismo problema y la misma solución.
