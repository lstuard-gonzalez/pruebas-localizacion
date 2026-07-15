# Pruebas de Localización — NKS Full Localización 18.0

Validación funcional de la localización venezolana (sin nómina) por el equipo de consultores NIKKOSOFT.

| Recurso | Descripción |
|---|---|
| [Checklist interactivo](checklist-validacion-localizacion.html) | 19 secciones · 109 verificaciones. Cada consultor marca ✓/✗, anota fallas y exporta sus resultados. |
| [Estadísticas por consultor](index.html) | Tablero que consolida el avance de todos leyendo la carpeta `data/`. |
| [data/](data/) | Un JSON por consultor con su avance ([instrucciones](data/README.md)). |
| [Guía funcional](GUIA-FUNCIONAL-PRUEBAS-LOCALIZACION.md) | Detalle funcional de los módulos bajo prueba. |
| [Manual de validación](MANUAL-VALIDACION-LOCALIZACION.md) | Procedimiento general de la validación. |

## Flujo de trabajo

1. El consultor abre el **checklist**, escribe su nombre y marca cada prueba (✓ pasó / ✗ falló + nota).
2. El avance se guarda automáticamente en su navegador.
3. Al avanzar, pulsa **Descargar JSON (repo)** y sube el archivo a `data/` (web de GitHub o git).
4. El tablero de **estadísticas** muestra el avance global y por sección de cada consultor, con sus fallas reportadas.

> Con GitHub Pages activado, el tablero queda publicado en
> `https://lstuard-gonzalez.github.io/pruebas-localizacion/`
> y el checklist en `https://lstuard-gonzalez.github.io/pruebas-localizacion/checklist-validacion-localizacion.html`.
