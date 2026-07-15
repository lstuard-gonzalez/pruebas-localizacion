# Resultados de validación por consultor

Aquí vive un archivo JSON por consultor con su avance del checklist.
El tablero [index.html](../index.html) lee esta carpeta y consolida las estadísticas.

## Cómo publicar tu avance

1. Abre el [checklist](../checklist-validacion-localizacion.html), escribe tu nombre y marca tus pruebas.
2. Pulsa **Descargar JSON (repo)** → se descarga `tu-nombre.json`.
3. Sube el archivo a esta carpeta `data/`:
   - Por la web de GitHub: entra a la carpeta → **Add file → Upload files** → arrastra el JSON → **Commit changes**.
   - O por git: copia el archivo aquí, `git add data/tu-nombre.json && git commit && git push`.
4. Para actualizar tu avance, repite los pasos: el archivo se llama igual y se reemplaza.

> Los archivos que empiezan con `_` (como `_ejemplo.json`) son plantillas y el tablero los ignora.

## Formato

```json
{
 "version": 1,
 "tester": "Nombre Apellido",
 "fecha": "2026-07-15",
 "resumen": { "ok": 12, "fail": 2, "pend": 95, "total": 109 },
 "rows": { "DUAL-01": "ok", "DUAL-02": "fail" },
 "notes": { "DUAL-02": "Los montos Bs no se recalcularon al cambiar la tasa" }
}
```
