# Resultados de validación

`resultados.json` es el **respaldo automático** del avance del equipo: un GitHub Action
([respaldo-datos.yml](../.github/workflows/respaldo-datos.yml)) lo trae del almacén en vivo
cada 6 horas (también se puede lanzar a mano desde la pestaña *Actions → Respaldo de resultados
→ Run workflow*).

Nadie edita esta carpeta a mano: los consultores marcan sus pruebas en el
[checklist](../checklist-validacion-localizacion.html) y el guardado es automático.
El [tablero de estadísticas](../index.html) lee el almacén en vivo y, si no responde,
cae a este respaldo.

## Formato

```json
{
 "version": 1,
 "consultores": {
  "luis": {
   "tester": "Luis",
   "fecha": "2026-07-15",
   "actualizado": "2026-07-15T14:30:00.000Z",
   "resumen": { "ok": 12, "fail": 2, "pend": 95, "total": 109 },
   "rows": { "DUAL-01": "ok", "DUAL-02": "fail" },
   "notes": { "DUAL-02": "Los montos Bs no se recalcularon al cambiar la tasa" }
  }
 }
}
```
