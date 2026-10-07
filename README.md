# Sitio de actualizaciones — MDM MultiCulture.Play

Esta rama es la que publica el sitio. **No contiene código ni binarios**: solo los metadatos que las
aplicaciones consultan para saber si hay una versión nueva.

| Aplicación | Endpoint |
|---|---|
| Estudiante | `https://korosoft-team.github.io/mdm-releases/estudiante/latest.json` |
| Docente (visor) | `https://korosoft-team.github.io/mdm-releases/docente/latest.json` |
| Gestor de contenido | `https://korosoft-team.github.io/mdm-releases/gestor/latest.json` |

Los binarios viven en las **releases** del repositorio, y los escribe el flujo de publicación. El
formato de cada `latest.json` está documentado en el `README.md` de la rama `main`.

**Estado:** los archivos `latest.json` aparecerán cuando exista la primera publicación firmada. Hasta
entonces estos endpoints responden que no hay nada, y **eso no rompe ninguna aplicación**: el
actualizador simplemente no encuentra actualización y la app sigue funcionando con su versión.
