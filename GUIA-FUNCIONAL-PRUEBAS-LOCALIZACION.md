# NKS Full Localización — Guía Funcional para Pruebas del Equipo

> **Rama:** `18.0-refactor` · **Odoo:** 18 Enterprise · **Fecha:** 2026-07-10
> **Alcance:** TODA la localización venezolana **excepto nómina** (Súper Nómina tiene su
> propia guía). 42 módulos.
>
> **Entorno de pruebas:** el staging de Odoo.sh de la rama
> (`nks-full-localizacion-18-0-refactor-*.dev.odoo.com`). La BD está **neutralizada**
> (no envía correos) y viene **poblada con datos demo** de todos los módulos: 4 compañías
> (Nikkosoft Prueba = la principal VE, NIKKOSOFT PROFIT, y 2 foráneas para probar el
> interruptor de localización), facturas, cobros, expedientes de importación, retenciones.
>
> **Cómo reportar:** cualquier hallazgo va al canal del proyecto con: módulo, pasos para
> reproducir, pantallazo y compañía usada. Un descuadre Bs/USD o un número de control
> duplicado/saltado es **severidad máxima**.

---

## 0. El concepto en una página

La localización se sostiene sobre cuatro pilares:

1. **Dualidad monetaria total (Bs + USD).** La contabilidad legal es en Bs; TODO se
   re-expresa en USD a la tasa BCV del documento. Cada asiento debe cuadrar **en ambas
   monedas** ("cuadre dual"). La tasa BCV se actualiza sola (cron) y cada documento
   congela su tasa (`tax_today`).
2. **Motor fiscal SENIAT.** Retenciones (IVA/ISLR/Municipal) por los 3 flancos, IGTF,
   libros de compra/venta, número de control, facturación digital (Prov. 102), ISAE
   municipal, licores.
3. **Operación real.** Importaciones (expediente completo), cobros distribuidos,
   anticipos, tesorería bancaria, POS con impresora fiscal, carga histórica.
4. **Plataforma NIKKOSOFT.** Control de accesos por perfil, centro de seguridad,
   tableros por lente de moneda, datos maestros para arrancar clientes.

**Interruptor maestro:** una compañía marcada como *Empresa Foránea* (o con país ≠
Venezuela) NO aplica nada de lo fiscal VE (ni retenciones, ni nº de control, ni IGTF).
Probar siempre en **Nikkosoft Prueba** salvo que se indique lo contrario.

---

## 1. Dualidad monetaria (account_dual_currency + l10n_ve_base + reportes)

**Qué es:** cada factura, pago y asiento lleva su tasa (`Tasa de Cambio` en el documento) y
todos los montos existen en Bs y USD. Los reportes Enterprise (Libro Mayor, Balance,
Antigüedad, P&G, Diario, Impuestos) tienen **selector de moneda** (Bs/USD) y **lente**.

**Los 3 lentes de re-expresión USD** (selector en los reportes y tableros):
- **Documento**: cada monto a la tasa histórica de su documento (visión contable).
- **Día**: todo a la tasa de la fecha del reporte (visión gerencial "hoy").
- **Mercado**: tasa de mercado informativa (Binance P2P, módulo premium `nks_market_rates`).
  *Solo presentación; la contabilidad nunca la usa.*

**Probar:**
- Crear factura de cliente en USD → validar → verificar que la tasa quedó congelada y que
  los totales Bs/USD son coherentes (monto USD × tasa = Bs).
- Cambiar manualmente la tasa antes de validar (campo Tasa) → los montos se recalculan.
- Contabilidad → Informes → Libro Mayor → alternar moneda USD y los 3 lentes.
- Verificar que el cron de tasas trae la tasa BCV del día (Contabilidad → Configuración →
  Monedas → USD → tasas).
- En la compañía **foránea**: nada de esto debe estorbar (sin tasa dual obligatoria).

## 2. Retenciones venezolanas (l10n_ve_full + l10n_ve_withholding + portal)

**Qué es:** el sistema de retenciones completo por los **3 flancos**: (a) como agente de
retención de IVA a proveedores, (b) ISLR (Dcto. 1.808, conceptos y tarifas PN/PJ,
residente/no residente), (c) retención municipal (base sin IVA, LOPPM 216.1). Además el
flanco pasivo: registrar las retenciones que **nos** hacen los clientes.

**Dos estilos conviven:**
- **Retención clásica** (documento de retención sobre la factura): comprobantes IVA
  (75%/100%), ISLR por concepto, municipal por tarifa de patente.
- **Retención EN EL PAGO** (estilo Profit Plus, `l10n_ve_withholding`): al registrar el
  pago, el wizard calcula y descuenta la retención automáticamente (se activa por compañía
  en Ajustes).

**Portal del proveedor** (`l10n_ve_withholding_portal`): el proveedor entra al portal y
descarga sus comprobantes de retención (IVA/ISLR/Municipal) con la marca del cliente.

**Probar:**
- Factura de proveedor con IVA → generar retención IVA 75% → validar el comprobante y su
  correlativo; verificar el asiento (cuenta de retención) y el cuadre dual.
- Factura de servicio → retención ISLR por concepto (verificar tarifa según tipo de
  persona); descargar el XML/TXT SENIAT desde el documento de retención ISLR.
- Activar retención-en-el-pago en Ajustes → pagar una factura con el wizard → la retención
  se descuenta y genera su comprobante.
- Entrar como usuario portal (proveedor demo) y descargar un comprobante.

## 3. IGTF (l10n_ve_full + account_dual_currency + POS)

**Qué es:** el 3% a pagos en divisas, en ambos sentidos (cobros a clientes y pagos a
proveedores), configurable por compañía (porcentaje y cuentas), aplicable por pago con su
diario propio. En POS tiene flujo propio (ver §10).

**Probar:**
- Registrar pago de cliente en USD con "Aplicar IGTF" → el asiento lleva la línea IGTF
  separada y el total cuadra en Bs y USD.
- Pagar a proveedor en divisa con IGTF → verificar el débito.
- Confirmar que un pago en Bs NO genera IGTF.

## 4. Libros fiscales y declaraciones (l10n_ve_full + l10n_ve_resquimcf + l10n_ve_isae)

**Qué es:**
- **Libro de Compras y Libro de Ventas** SENIAT (con nº de control, retenciones, tipos de
  documento), exportables.
- **TXT de retenciones IVA** (formato SENIAT para declarar).
- **RESQUIMCF**: resumen de compras/ventas agrupado por tipo de documento (Factura/ND/NC).
- **ISAE / Patente municipal** (`l10n_ve_isae`): declaración periódica del impuesto sobre
  actividades económicas — calcula "el mayor" entre ingresos brutos × alícuota y el mínimo
  tributable, con criterio **percibido** (lo cobrado en el período).
- **Libro de inventario** y reportes auxiliares.

**Probar:**
- Generar Libro de Ventas del mes → cada factura debe traer su **número de control**; una
  factura digital emitida debe mostrar el número que asignó la imprenta.
- Generar Libro de Compras → las facturas de importación aparecen con el formato de
  importación (ver §8).
- Crear una declaración ISAE del período → verificar que la base son los cobros del
  período (no lo facturado) y que aplica el mínimo si corresponde.
- RESQUIMCF: generar el resumen y validar los subtotales por tipo de documento.

## 5. Número de control (nikko_control_number) — ★ NUEVO esta semana

**Qué es:** el correlativo legal SENIAT de facturas, ND, NC **y guías de despacho**, ahora
con **series configurables por nivel**:

- **Por compañía** (default): una serie única (p. ej. `00-`). Factura+ND+NC pueden
  **compartir** un solo correlativo (forma libre) o llevar cada uno el suyo (preimpreso).
- **Por diario = sucursal** (★ nuevo): cada sucursal factura por su diario con su propia
  serie (`00-`, `01-`…). Config en Contabilidad → Configuración → Números de Control
  (columna Diario).
- **Guía de despacho por almacén** (★ nuevo): el picking de salida recibe su número de
  control al validarse, con serie por almacén (o de compañía si el almacén no define una).
  Espacio de numeración SEPARADO del de facturas.

**Guardas fiscales:** dos series distintas no pueden tener prefijos que colisionen (el
sistema lo bloquea al configurar); la unicidad dura la garantiza la BD.

**Probar:**
- Validar facturas en dos diarios de venta con series distintas → correlativos
  independientes con su prefijo.
- Intentar crear dos series con el mismo prefijo → debe rechazarlo con mensaje claro.
- Validar una entrega (picking de salida) → recibe nº de control de guía; una recepción NO.
- Diario de contingencia: al validar exige capturar el número manual del talonario.

## 6. Medio de emisión fiscal por diario — ★ NUEVO esta semana

**Qué es:** cada diario de venta declara su **medio de emisión** (pestaña Ajustes avanzados
del diario, sección "Emisión Fiscal (Venezuela)"):

1. **Forma libre** — numera con la secuencia local (default).
2. **Facturación digital** — la factura se emite a la imprenta digital (Prov. 102) y el
   número de control lo asigna la imprenta.
3. **Impresora fiscal** — el número lo asigna la máquina fiscal homologada.
4. **Talonario de contingencia** — número manual del talonario preimpreso (obligatorio por
   ley como medio alterno).

Esto permite que **una misma compañía conviva con varios medios**: tienda con impresora
fiscal + ventas web digitales + talonario de respaldo.

**Probar:**
- Configurar un diario como "Facturación digital" → validar factura → NO consume secuencia
  local (queda sin nº de control hasta emitirse).
- Diario "Forma libre" → validar → nº de control local inmediato.
- El flag global "Emite Factura Digital (SENIAT)" de la compañía es el interruptor maestro:
  apagado, ningún diario emite digital.

## 7. Facturación digital (nks_einvoice_ve)

**Qué es:** el emisor fiscal digital multi-proveedor (driver The Factory HKA): emite
factura, NC/ND, **guía de despacho** y **comprobantes de retención** al proveedor de
imprenta digital, integrado al asistente nativo "Enviar e imprimir". Con cola asíncrona,
reintentos, log de auditoría inmutable y dashboard de emisión.

**El lazo del número:** al emitir con éxito, el número que asigna la imprenta aterriza en
el campo nº de control del documento → Libro de Ventas y retenciones lo leen.

**Probar** (en pruebas sin credenciales reales, el driver está en modo demo/error
controlado — lo importante es el flujo):
- Factura en diario digital → Enviar e Imprimir → opción "Factura Digital (SENIAT)".
- Ver el estado de emisión en la factura (Borrador/En cola/Emitido/Error) y el historial.
- Menú Contabilidad → Emisión Fiscal: dashboard, cola de trabajos, log.
- Cliente con "Emitir solo en Bolívares": la factura en divisa se emite sin bloque USD.

## 8. Importaciones (nks_importaciones + 4 satélites)

**Qué es:** el expediente de importación completo, 7 estados (borrador → embarque →
llegada → DUA → liquidación → cierre), sobre landed costs nativos:
- Catálogo arancelario (códigos HS, gravámenes), exoneraciones por decreto (90%/100%),
  permisología (SACS/SENCAMER) con vigencias y alertas.
- **Cálculo de tributos**: arancel ad-valorem sobre CIF a la **tasa del DUA**, tasa por
  régimen aduanero (1%), IVA de importación (deducible o no — Art. 33 LIVA).
- Distribución de costos multi-criterio (valor/peso/volumen), costo provisional y
  **reliquidación** con venta parcial (ajuste al costo de lo vendido).
- **ISLR por servicios al exterior** (fletes, asistencia técnica — Dcto. 1.808).
- Puente al **Libro de Compras** formato importaciones y aviso al vender mercancía con
  costo provisional.

**Probar:**
- Abrir un expediente demo → recorrer los estados → en la liquidación verificar arancel,
  tasa y IVA contra el CIF en Bs a la tasa DUA.
- Marcar "IVA no deducible" → el IVA se va al costo en vez de crédito fiscal.
- Registrar una factura de flete al exterior → retención ISLR automática por concepto.
- Vender producto de un expediente provisional → aviso; cerrar el expediente → asiento de
  reliquidación.

## 9. Cobros, anticipos y pedido→compra

### 9.1 Cobros Distribuidos (nikko_cobros_clientes)
**Qué es:** UN cobro global del cliente aplicado contra N facturas, mezclando pagos nuevos,
créditos existentes y notas de crédito. Su joya: el **diferencial cambiario** entre la tasa
de la factura y la del cobro se materializa como **ND/NC fiscal con IVA discriminado por
línea** (Art. 51 RLIVA); si el diferencial es < 1 Bs, asiento directo (regla SENIAT). El
excedente queda como saldo a favor del cliente. Cancelación con reversa completa (solo
gerentes).
**Dónde:** Contabilidad → Clientes → Cobros Distribuidos.
**Probar:** cobro multi-factura con tasa distinta a la de las facturas → validar → revisar
la ND generada (con su nº de control), el cuadre Bs/USD y que las facturas quedaron
pagadas. Cancelar un cobro validado y verificar que todo se revierte.

### 9.2 Anticipos (nks_anticipos)
**Qué es:** anticipos de cliente Y a proveedor sin factura (sobre `account.payment`
nativo), con saldo disponible, aplicación posterior a facturas (conciliación nativa),
estados (borrador/publicado/parcial/aplicado) e IGTF si es en divisa.
**Dónde:** Contabilidad → Clientes → Anticipos de cliente / Proveedores → Anticipos a
proveedor.
**Probar:** crear anticipo de cliente en USD → publicar → aplicarlo a 2 facturas →
verificar saldo restante y estados. Lo mismo con proveedor.

### 9.3 Pedido → Compra (nks_sale_to_purchase, premium)
**Qué es:** desde el pedido de venta, generar automáticamente la orden de compra al
proveedor elegido por línea (sobre MTO nativo + seam de supplierinfo).
**Probar:** activar en Ajustes → pedido con producto configurado → confirmar → la OC nace
con el proveedor elegido y enlazada al pedido.

## 10. Punto de Venta (6 módulos POS)

**Qué es:** POS venezolano completo: precios en **dual moneda** en pantalla y ticket,
**IGTF** en el flujo de pago (`pos_igtf_workflow`), tasa BCV del día capturada en la orden,
**impresora fiscal** (3mit print server + reportes Z almacenados con tipo), RIF/dirección
obligatorios en el cliente (`wdu_contact_required`), múltiples diarios de efectivo, cierre
forzado de sesiones (solo gerente) y diario de respaldo para el cierre dual.

**Probar:**
- Abrir sesión POS → los productos muestran precio Bs y USD.
- Venta con pago en divisa → IGTF aplicado en el flujo de pago.
- Cierre de sesión → popup dual currency cuadrado; probar el cierre forzado como gerente.
- Cliente nuevo en POS: exige RIF.

## 11. Tesorería bancaria (l10n_ve_bank_hub + l10n_ve_bank_format)

**Qué es:** el núcleo de integración bancaria VE: catálogo de bancos, credenciales
cifradas, **lotes de pago a proveedores** con maker-checker (quien monta no aprueba), y
generación de **archivos TXT de dispersión** por banco (motor puro, testeable). Auditoría
completa.

**Probar:** crear un lote de pagos a proveedores → montarlo → aprobarlo con OTRO usuario →
descargar el TXT y validar el formato del banco.

## 12. Carga histórica / Backdating (nks_backdating)

**Qué es:** registrar cotizaciones, entregas, facturas y órdenes de producción **con fecha
en el pasado** (migraciones, carga inicial), respetando numeración y contabilidad, con
control por compañía.

**Probar:** activar en Ajustes → crear una factura con fecha del mes pasado → el asiento y
el correlativo respetan la fecha histórica.

## 13. Formatos de factura (em_format_invoice + easy_invoice_*)

**Qué es:** los PDF legales: **forma libre** (em_format_invoice, exige condición de pago y
valida requisitos SENIAT al crear), y dos variantes Bs/USD (estándar y Cyslato/PDVSA, con
el flag "Factura PDVSA" en compañía).

**Probar:** imprimir una factura en cada formato → verificar RIF, nº de control, tasa,
montos duales y el cintillo legal.

## 14. Restricciones fiscales y banner (nks_vzla_fiscal_restr)

**Qué es:** el guardián SENIAT: restricciones de edición sobre documentos fiscales,
**banner de estado fiscal** en ventas (el "Cintillo Vivo": alerta escalada por período de
IVA), IGTF informativo en documentos y reporte de forma libre.

**Probar:** intentar modificar una factura validada → bloqueado; ver el banner en el
formulario de pedidos/facturas.

## 15. Plataforma NIKKOSOFT (seguridad y arranque)

- **NKS Access PRO** (`nks_access_control`): perfiles de acceso que ocultan/restringen
  campos, menús, botones y vistas por grupo — sin Studio. *Probar:* asignar un perfil
  restrictivo a un usuario de prueba y verificar que los elementos desaparecen.
- **NKS Security PRO** (`nks_security_center`): auditoría de inicios de sesión, firewall
  de IPs (allow/block), bloqueo del modo desarrollador. *Probar:* revisar el log de
  sesiones; activar bloqueo de devtools para un grupo.
- **Datos Maestros** (`l10n_ve_nikkosoft_master`): plan de cuentas maestro NIKKOSOFT,
  bancos, impuestos y diarios para arrancar cualquier cliente VE. *Probar:* revisar el
  plan de cuentas instalado en una compañía nueva.

## 16. Analítica

- **Cubo de Ventas** (`nks_ventas_dashboard`): pivote de análisis de ventas (cliente,
  producto, mes, USD/VES) que incluye facturas migradas.
- **Tableros por lente de moneda** (ADR-016): los dashboards de spreadsheet muestran
  medidas en Bs, USD documento, USD día y mercado (variantes en el sidebar).
- **Reportes MRP dual** (`l10n_ve_dual_currency_reports_mrp`): análisis de costos de
  producción re-expresado en USD.

**Probar:** abrir el cubo de ventas y cruzar cliente × mes en USD; abrir un tablero y
alternar variantes de moneda.

## 17. Licores (nikko_licores_ve)

**Qué es:** la localización del ramo de licores: impuesto LIAE por grado alcohólico /
precio de venta al público, registro de licores del producto.

**Probar:** producto de licor con sus datos LIAE → factura → el impuesto del ramo se
calcula y declara.

---

## Apéndice A — Qué se estrenó ESTA semana (probar con cariño)

| Novedad | Dónde |
|---|---|
| Medio de emisión fiscal por diario (4 medios) | Diario de venta → Ajustes avanzados |
| Serie de nº de control por diario (sucursal) | Config → Números de Control |
| Guía de despacho con nº de control por almacén | Validar picking de salida |
| Puente digital: nº de la imprenta → libro | Emitir factura/guía digital |
| Cero API prohibida en los 2 módulos core | (interno, estabilidad) |
| Gate anti-caída de registry en BDs con datos | (interno, estabilidad) |

## Apéndice B — Matriz módulo → sección

| Módulo | Sección |
|---|---|
| account_dual_currency, l10n_ve_base, l10n_ve_dual_currency_reports(_mrp) | 1, 16 |
| l10n_ve_full | 2, 3, 4, 13 |
| l10n_ve_withholding, l10n_ve_withholding_portal | 2 |
| l10n_ve_resquimcf, l10n_ve_isae | 4 |
| nikko_control_number | 5 |
| nks_einvoice_ve | 6, 7 |
| nks_importaciones (+bcv_rate, +fiscal_book, +sale, +withholding) | 8 |
| nikko_cobros_clientes, nks_anticipos, nks_sale_to_purchase | 9 |
| 3mit_print_server, nks_impresora_fiscal, pos_* , wdu_*, l4l_* | 10 |
| l10n_ve_bank_hub, l10n_ve_bank_format | 11 |
| nks_backdating | 12 |
| em_format_invoice, easy_invoice_bsusd, easy_invoice_standard_bsusd | 13 |
| nks_vzla_fiscal_restr | 14 |
| nks_access_control, nks_security_center, l10n_ve_nikkosoft_master | 15 |
| nks_ventas_dashboard, nks_market_rates | 16, 1 |
| nikko_licores_ve | 17 |
| stock_batch_report_custom | (reporte de transferencias por lote con empaques) |

---

*NIKKOSOFT CONSULTORES C.A. — Localización Venezolana NKS · rama 18.0-refactor*
