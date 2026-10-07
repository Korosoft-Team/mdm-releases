# Canal de distribución y actualizaciones — MDM MultiCulture.Play

> **Qué es:** repositorio **público de solo artefactos**: los instaladores y los metadatos de
> actualización de las tres aplicaciones de la suite. **No contiene código fuente.**
> **Por qué es público:** el actualizador de las aplicaciones descarga desde aquí **sin credenciales**.
> Así ningún token viaja dentro de las apps instaladas, que es el riesgo que se quiso evitar.
> **Contrato:** Cláusula Tercera 3.3 (instalación en los equipos provistos por LA CONTRATANTE) y
> Cláusula Octava (autoría y cesión de derechos).
> **Créditos y condiciones de uso:** [`AVISO-CREDITOS.md`](./AVISO-CREDITOS.md).

---

## 1. Las tres aplicaciones

| Aplicación | Identificador | Código fuente | Endpoint de actualización |
|---|---|---|---|
| **Estudiante** | `com.koroSoft.mdm.estudiante` | privado: `mdm-estudiante` | `https://korosoft-team.github.io/mdm-releases/estudiante/latest.json` |
| **Docente** (visor) | `com.koroSoft.mdm.docente` | privado: `mdm-docente` | `https://korosoft-team.github.io/mdm-releases/docente/latest.json` |
| **Gestor de contenido** | `com.koroSoft.mdm.gestor` | privado: `mdm-gestor` | `https://korosoft-team.github.io/mdm-releases/gestor/latest.json` |

Los tres repositorios de código son **privados** y permanecen en custodia del equipo de desarrollo
hasta la suscripción del acta de cesión (Cláusula Octava 8.4). **Este repositorio no contiene el
código**: solo lo que se instala en las máquinas.

---

## 2. Cómo se instalan las aplicaciones

1. Entrar a la pestaña **Releases** de este repositorio.
2. Elegir la versión más reciente de la aplicación que corresponda.
3. Descargar el instalador de la plataforma:
   - **Windows:** el archivo terminado en `-setup.exe` (instalador NSIS).
   - **Linux:** el archivo `.AppImage`.
4. Ejecutarlo e instalar.

**Formato correcto, y esto importa:** solo se actualizan solas las instalaciones hechas con el
**instalador NSIS** (Windows) y con la **AppImage** (Linux). Un paquete `.deb` o `.rpm` se instala,
pero **no se autoactualiza**: habría que reinstalarlo a mano en cada equipo.

---

## 3. Cómo funciona la actualización

1. Al abrirse, la aplicación pide su archivo `latest.json`.
2. Si la versión publicada es mayor que la instalada, descarga el artefacto de su plataforma.
3. **Verifica la firma** del artefacto contra la clave pública incrustada en la aplicación.
   Si la firma no coincide, **no se instala nada**.
4. La actualización se aplica al reiniciar.

Propiedades de este diseño, dichas explícitamente:

- **No hay sondeo continuo.** Es una petición al abrir la aplicación, no un bucle.
- **Si el endpoint no responde, la aplicación sigue funcionando** con la versión que tiene. El
  actualizador nunca bloquea una aplicación.
- **El canal es tolerante a fallos y la firma es obligatoria.** Un artefacto manipulado no se instala.
- **Descargar no requiere cuenta**: no hay credenciales en el canal ni en las aplicaciones.

---

## 4. Clave de firma

**Clave pública** — es pública por diseño y va dentro de cada aplicación, en
`tauri.conf.json`, en `plugins.updater.pubkey`:

```
dW50cnVzdGVkIGNvbW1lbnQ6IG1pbmlzaWduIHB1YmxpYyBrZXk6IDlCRTQ3RDVGRUJDRThCRDYKUldUV2k4N3JYMzNrbTEvWlhEUmxvam1LTXhlOTROUDdSTWJ6TzJhZkt0SjJnQmNoK2cxdzdsRXcK
```

**Clave privada** — la custodia el equipo de desarrollo y **nunca** entra a este repositorio ni al
historial de ningún repositorio. Vive en los secretos de CI de cada repositorio de aplicación
(`TAURI_SIGNING_PRIVATE_KEY` y `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`).

> **Rotar la clave obliga a recompilar y reinstalar las tres aplicaciones**, porque la pública está
> incrustada en el binario. Se genera una vez y se guarda como corresponde. Es una puerta de un solo
> sentido.

---

## 5. Cómo se publica una versión

El flujo está planteado para que publicar sea una etiqueta, no una operación manual.

**Secretos necesarios en cada repositorio de aplicación:**

| Secreto | Para qué |
|---|---|
| `TAURI_SIGNING_PRIVATE_KEY` | firmar el artefacto |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | contraseña de la clave privada |
| `RELEASES_TOKEN` | token de grano fino con `contents: write` **solo** en este repositorio: es lo que permite subir el artefacto desde el repositorio privado |

**Pasos (plantilla en [`templates/publish-app.yml`](./templates/publish-app.yml)):**

1. Se etiqueta la versión en el repositorio de la aplicación.
2. El workflow compila con `tauri build` en `windows-latest` y `ubuntu-latest`.
3. Firma los artefactos y crea la release en **este** repositorio con los binarios.
4. Actualiza `updates/<app>/latest.json` en la rama publicada, con la versión y las firmas nuevas.

> **Estado de esta plantilla: no ejecutada todavía.** Está escrita y documentada, pero el primer
> build firmado y publicado sigue pendiente. Se considerará válida cuando exista una release real con
> su `latest.json` y una instalación que se haya actualizado sola.

---

## 6. Formato de `latest.json`

Un archivo por aplicación, en la rama publicada del sitio. Es el formato del actualizador de Tauri v2:

```json
{
  "version": "0.2.0",
  "notes": "Qué cambia en esta versión, en español neutro",
  "pub_date": "2026-10-08T12:00:00Z",
  "platforms": {
    "windows-x86_64": {
      "signature": "<contenido del .sig del instalador NSIS>",
      "url": "https://github.com/Korosoft-Team/mdm-releases/releases/download/<app>-v0.2.0/<archivo>.nsis.zip"
    },
    "linux-x86_64": {
      "signature": "<contenido del .sig de la AppImage>",
      "url": "https://github.com/Korosoft-Team/mdm-releases/releases/download/<app>-v0.2.0/<archivo>.AppImage.tar.gz"
    }
  }
}
```

Los artefactos que consume el actualizador son el paquete `.nsis.zip` (Windows) y el
`.AppImage.tar.gz` (Linux), cada uno con su archivo `.sig` al lado. El `-setup.exe` y la `.AppImage`
sueltos son para la **primera instalación**, no para actualizar.

---

## 7. Nota para el traspaso

Cuando la UNMSM asuma la operación de la suite, este repositorio debe pasar a su cuenta. **Al
transferirlo cambia la URL del sitio** (`<nuevo-propietario>.github.io/...`), y el endpoint está
incrustado en cada aplicación. Tres caminos:

1. **Dominio propio para el sitio** (lo recomendable cuando la UNMSM tenga su dominio: la Cláusula
   Quinta 5.6 ya prevé que asuma hosting y dominio).
2. **Sobrescribir el endpoint en tiempo de ejecución** desde el código Rust de cada aplicación, igual
   que la dirección del servidor central.
3. Recompilar y reinstalar. Es el peor de los tres y hay que evitarlo.

**Decisión pendiente, anotada a propósito:** todavía no se ha fijado cuál de los tres caminos se usa.
Se resuelve junto con la dirección del servidor central, que tiene el mismo problema y la misma
solución.
