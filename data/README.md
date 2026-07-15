# Resultados de validación

Aquí vive un archivo JSON por consultor (`danny.json`, `chriselys.json`, `luis.json`),
escrito **automáticamente** por el [checklist](../checklist-validacion-localizacion.html):
cada consultor toca su nombre al entrar y sus marcas se guardan solas aquí, con
historial de commits.

Nadie edita esta carpeta a mano. El [tablero de estadísticas](../index.html) lee estos
archivos en vivo.

## Formato

```json
{
 "version": 1,
 "tester": "Luis",
 "fecha": "2026-07-15",
 "actualizado": "2026-07-15T14:30:00.000Z",
 "resumen": { "ok": 12, "fail": 2, "pend": 95, "total": 109 },
 "rows": { "DUAL-01": "ok", "DUAL-02": "fail" },
 "notes": { "DUAL-02": "Los montos Bs no se recalcularon al cambiar la tasa" }
}
```
