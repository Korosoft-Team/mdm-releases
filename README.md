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

### Estado de la integración en las tres aplicaciones

| Aplicación | Implementación | Estado |
|---|---|---|
| **Estudiante** | `mdm-estudiante` PR #316 (referencia) | Escrita y verificada en local |
| **Docente** (visor) | `mdm-docente` PR #39 | Escrita y verificada en local |
| **Gestor de contenido** | `mdm-gestor` PR (rama `feat/updater-gestor`) | Escrita y verificada en local |

**Ninguna está fusionada todavía**, y **el canal no ha servido ninguna actualización**: falta crear los
secretos de firma y el `RELEASES_TOKEN` en cada repositorio, y publicar la primera versión. Hasta que
alguien instale esa primera versión y la vea actualizarse sola, **el canal no está probado**.

Para replicar el patrón, o para revisar qué se hizo: [`docs/replicar-en-otra-app.md`](./docs/replicar-en-otra-app.md).

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
> *Ejecutar de todas formas*. La firma **minisign** de este canal (sección 4) es otra cosa: verifica las
> actualizaciones, **no** elimina ese aviso.
>
> **Decisión tomada (7 de octubre de 2026):** se entrega **sin** certificado de firma de código y el
> aviso **se declara en la guía de instalación y se explica en la capacitación**, para que el docente no
> lo tome por un defecto.

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

### De dónde saca la actualización una aplicación ya instalada

**Del repositorio público, nunca del código privado.** Esa es toda la respuesta, y conviene verla entera
porque es la confusión más frecuente:

```text
[ repositorio de código  PRIVADO ]
        │   (1) el CI compila y firma con la clave privada
        v
[ mdm-releases  PUBLICO ]          (2) recibe el -setup.exe, su .sig y el latest.json
        │
        │   (3) HTTPS sin credenciales: cualquier app lo descarga
        v
[ aplicacion instalada en la laptop ]
        (4) pide <app>/latest.json al abrirse
        (5) compara la version publicada con la suya
        (6) descarga el instalador desde la release publica
        (7) VERIFICA LA FIRMA con la clave publica incrustada en su propio binario
        (8) si coincide, instala y se reinicia; si no, no instala absolutamente nada
```

El repositorio privado **solo participa cuando publicamos**, y para eso el CI usa un token
(`RELEASES_TOKEN`) que vive en los secretos del repositorio, **nunca dentro de la aplicación**.

Esta es exactamente la razón por la que el canal es público: si las apps tuvieran que leer un
repositorio privado, llevarían un token dentro y **cualquiera podría extraerlo del binario**. Publicar
los binarios evita eso, y el código fuente sigue privado.

Tres consecuencias prácticas:

- **La aplicación no necesita cuenta ni autenticación** para actualizarse, y **no consulta la API de
  GitHub**: son descargas de archivos (`github.io` y los adjuntos de la release), así que no hay límite
  de peticiones.
- **Una aplicación instalada sin el plugin del actualizador no se actualiza nunca**, por muy bien que
  funcione el canal: no lleva el mecanismo dentro. Ver la sección 5 y
  [`docs/integracion-app.md`](./docs/integracion-app.md).
- **La versión publicada tiene que ser mayor** que la instalada. Se sube en `tauri.conf.json` antes de
  etiquetar, o el actualizador no hace nada.

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
3. `bundle.targets` **se queda en `"all"`**, y el build de entrega pasa `--bundles nsis`. Fijarlo en
   `["nsis"]` **rompe `pnpm tauri build` en Linux**, que es el entorno de desarrollo de quien programa
   desde ahí. La entrega no depende de ese valor: depende del argumento del comando.
4. `plugins.updater.pubkey` y `plugins.updater.endpoints`.
5. **Solo si se usa la API de JavaScript**: el permiso **`updater:default`** en las capacidades, o la
   API del plugin queda bloqueada para el frontend. **La implementación de referencia no lo necesita**,
   porque hace la comprobación desde Rust. Ver
   [`replicar-en-otra-app.md`](./replicar-en-otra-app.md).
6. La comprobación al arrancar, con su aviso al usuario.

---

## 6. Cómo se publica una versión

> **Esta es la única compilación que queda en GitHub, y es deliberada.** Mientras el producto está en
> desarrollo, **GitHub no compila**: los workflows `tests.yml` de los tres repositorios de aplicación
> y `demo-windows.yml` de la app del estudiante están **deshabilitados a propósito** (ADR-015 en
> `mdm-gestor`), y la verificación se hace en local y se reporta en el PR. La publicación es la
> excepción porque el instalador NSIS de Windows **no se puede construir desde Linux**: se compila en
> el runner de Windows, y **solo** cuando alguien crea a propósito una etiqueta de versión
> (`estudiante-v*`, `docente-v*`, `gestor-v*`). No corre en pushes ni en PRs. Este repositorio, además,
> no tiene workflows propios: solo artefactos.

**Lo que se hace UNA SOLA VEZ (configuración inicial)**

Tres secretos por repositorio de aplicación:

| Secreto | Para qué | Quién lo aporta |
|---|---|---|
| `TAURI_SIGNING_PRIVATE_KEY` | firmar el artefacto | el equipo: `~/.config/mdm/updater/updater.key` |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | contraseña de la clave privada | el equipo: `~/.config/mdm/updater/updater.key.password` |
| `RELEASES_TOKEN` | subir el artefacto a **este** repositorio desde un repositorio privado: el `GITHUB_TOKEN` **no** cruza repositorios | **el responsable de la cuenta** (ver abajo) |

Se publican con el script del equipo, que lee los valores de los archivos y no los imprime:

```bash
# 1. el responsable crea el token y lo guarda (permisos 600), nunca en el chat:
#    ~/.config/mdm/updater/releases.token
# 2. se publican los nueve secretos (tres por repositorio):
bash ~/.config/mdm/updater/publicar-secretos.sh
```

**El `RELEASES_TOKEN` es lo único que no puede automatizar el agente, y conviene entender por qué:**
GitHub **no expone ninguna API para crear tokens**; hay que generarlos desde la cuenta. Y las **claves
de despliegue SSH no son una alternativa en esta organización**: GitHub las rechaza en los cuatro
repositorios (`Deploy keys are disabled for this repository`, verificado el 2026-10-09), así que el
camino por SSH no existe y el token de la API es la única vía. Alcance mínimo recomendado:

- **Token de grano fino** (preferido): *Repository access* → **Only select repositories** →
  `mdm-releases`; *Permissions* → **Contents: Read and write**. No toca ningún repositorio privado.
- **Token clásico** (alternativa): marcar **solo `public_repo`**. Alcanza para este repositorio, que
  es público, y no da acceso a los repositorios privados del equipo.

> **Por qué el token no viaja dentro de las aplicaciones:** vive como secreto cifrado en cada
> repositorio y solo lo leen los workflows de ese repositorio. La app instalada descarga de este canal
> **sin credenciales**, que es justamente el motivo de que este repositorio sea público y no contenga
> código.

**Lo que YA ES AUTOMÁTICO (cada versión)**

Archivo de referencia: [`docs/replicar-en-otra-app.md`](./replicar-en-otra-app.md), que apunta al
**workflow ya probado** en `mdm-estudiante` (PR #316) y dice exactamente qué cambiar por aplicación.
**No hay copias sueltas del workflow**: la plantilla anterior en `templates/` se retiró justamente para
que no divergieran.

1. **Subir la versión en `tauri.conf.json`** y etiquetar. La versión publicada debe ser **mayor** que la
   instalada, o el actualizador no hará nada. El workflow **falla** si la etiqueta y el archivo no
   coinciden.
2. El workflow compila en `windows-latest` con `tauri build --bundles nsis`.
3. Con la clave en el entorno, Tauri genera el instalador y **su `.sig`**. Si no aparece, el workflow
   **falla** en vez de publicar algo inservible.
4. Se crea la release en **este** repositorio con el `-setup.exe` y su `.sig`.
5. Se actualiza `updates/<app>/latest.json` en la rama del sitio.

Nada de eso se hace a mano: la etiqueta es la única acción y el resto encadena solo. Lo que la
etiqueta **no** puede hacer por sí solo es existir: crearla es la decisión de publicar.

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
