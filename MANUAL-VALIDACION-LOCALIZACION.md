# NKS Full Localización — Manual de Usuario y Checklist de Validación

> **Rama:** `18.0-refactor` · **Fecha:** 2026-07-15 · **Audiencia:** equipo NIKKOSOFT (no
> requiere conocer los módulos). **Alcance:** cada módulo NUEVO o CAMBIADO en el refactor,
> **sin nómina**.
>
> **Cómo usar este documento:** cada módulo trae (1) *Qué hace*, (2) *Configuración* paso
> a paso, (3) *Manual de uso* y (4) su **Checklist de validación**: marca ✅/❌ en cada fila
> y anota el resultado. Si una fila da ❌, reporta con el código de la fila (p. ej. `NC-04`),
> pasos, pantallazo y compañía.
>
> **Antes de empezar (aplica a TODO):**
> 1. Entra al staging (`nks-full-localizacion-18-0-refactor-*.dev.odoo.com`). La BD está
>    neutralizada (no envía correos) y tiene datos demo.
> 2. Trabaja en la compañía **Nikkosoft Prueba** (la VE principal), salvo indicación.
> 3. Tu usuario necesita: Contabilidad → *Contable*, Inventario → *Administrador*, y para
>    los apartados de configuración, *Ajustes de administración*.
> 4. **Severidad máxima** (reportar de inmediato): un asiento que no cuadre en Bs o USD,
>    un número de control duplicado o saltado, o un Internal Server Error.
>
> **Ruta sugerida** (de base a avanzado): §1 → §2 → §3 → §5 → §6 → §7 → §8 → resto.

---

## §1 · Dualidad Monetaria — `account_dual_currency` (CAMBIO MAYOR: nativización + lentes)

**Qué hace:** toda la contabilidad vive en Bs y se re-expresa en USD. Cada documento
congela su tasa BCV (`Tasa de Cambio`). En el refactor se eliminó ~2.700 líneas de lógica
duplicada: la verdad en Bs es 100% nativa de Odoo y TODO el lado USD deriva de una sola
doctrina con **3 lentes**: *Documento* (tasa histórica), *Día* (tasa de la fecha del
reporte) y *Mercado* (informativa).

**Configuración:**
1. Ajustes → Contabilidad → bloque *Dual Currency*: verificar **Moneda de referencia = USD**.
2. Contabilidad → Configuración → Monedas → USD: activa y con tasas (el cron BCV las trae
   solo; revisar que exista la tasa de hoy).

**Manual de uso:**
1. Crear factura de cliente en USD → el campo *Tasa* se llena con la BCV del día (editable
   antes de validar).
2. Validar → abrir el asiento: cada apunte muestra Débito/Crédito en Bs **y** su columna USD.
3. Contabilidad → Informes → Libro Mayor → arriba, selector de moneda **USD** y selector de
   **lente**.

**Checklist DUAL:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| DUAL-01 | Factura USD 100 con tasa 100 → totales | Total Bs = 10.000, total USD = 100 | |
| DUAL-02 | Cambiar la tasa manual antes de validar | Los montos Bs se recalculan al instante | |
| DUAL-03 | Asiento de la factura | Débito=Crédito en Bs **y** en USD (cuadre dual) | |
| DUAL-04 | Registrar pago con tasa distinta | El pago usa SU tasa; la factura queda saldada | |
| DUAL-05 | Libro Mayor en USD lente *Documento* | Cada línea a la tasa histórica de su asiento | |
| DUAL-06 | Libro Mayor en USD lente *Día* | Todo re-expresado a la tasa de hoy | |
| DUAL-07 | Cron de tasas (esperar al día siguiente o correrlo) | Tasa BCV nueva en USD sin intervención | |
| DUAL-08 | Compañía foránea: crear factura | No exige tasa dual ni aplica lo fiscal VE | |

---

## §2 · Retenciones clásicas y 3 flancos — `l10n_ve_full` (CAMBIO: ADR-011 + ADR-017)

**Qué hace:** retenciones IVA (75/100%), ISLR (Dcto. 1.808) y municipal, por los 3 flancos:
como **agente** (retenemos a proveedores), como **contribuyente** (nos retienen los
clientes = *retención soportada*) y municipal. **Cambio ADR-017 (nuevo):** la retención
soportada tiene UN solo motor — la pantalla `Retenciones recibidas` es ahora una **bandeja
de captura** que alimenta el comprobante `account.wh.iva` de venta (nada asienta dos veces).

**Configuración:**
1. Ajustes → Contabilidad → bloque *Retenciones VE*: cuentas de retención IVA/ISLR/municipal
   y diarios (compra y venta).
2. En el **proveedor**: pestaña Contabilidad → % de retención IVA (75 o 100) y conceptos
   ISLR habituales. En el **cliente** (si nos retiene): marcar agente de retención.
3. Para municipal: tabla de tarifas de patente (Contabilidad → Configuración → Tarifas
   municipales) y activar *Aplicar retención municipal* en la compañía.

**Manual de uso (flanco agente):**
1. Factura de proveedor con IVA → validar.
2. Botón/pestaña Retención IVA → genera el comprobante con su correlativo → confirmar.
3. Para ISLR: la factura de servicio detecta el concepto → documento de retención ISLR →
   confirmar → descargar XML/TXT SENIAT del período.

**Manual de uso (flanco soportado, ADR-017):**
1. Cobramos una factura y el cliente nos retuvo IVA → Contabilidad → Clientes →
   Retenciones recibidas → capturar el comprobante del cliente (número, fecha, monto).
2. Confirmar → el sistema genera el comprobante de venta (`account.wh.iva` tipo venta) que
   asienta contra la cuenta del diario de venta y entra al Libro de Ventas y al TXT.

**Checklist RET:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| RET-01 | Retención IVA 75% sobre factura con IVA 16 de base 1.000 | Comprobante por 120 (75% de 160) | |
| RET-02 | Proveedor al 100% | Retiene 160 completo | |
| RET-03 | Asiento del comprobante | Cuadra en Bs y USD; usa la cuenta configurada | |
| RET-04 | ISLR por concepto (honorarios PJ) | Tarifa correcta según tipo de persona; sustraendo aplicado | |
| RET-05 | TXT IVA SENIAT del período | Incluye los comprobantes confirmados, formato correcto | |
| RET-06 | Municipal: factura de actividad gravada | Base SIN IVA (LOPPM 216.1) × tarifa | |
| RET-07 | **ADR-017**: capturar retención recibida | Genera UN solo comprobante de venta; NO duplica asiento | |
| RET-08 | La retención recibida aparece en Libro de Ventas | Sí, con el número del comprobante del cliente | |
| RET-09 | Intentar retener dos veces la misma factura | Bloqueado con mensaje claro | |

---

## §3 · Retención EN el pago — `l10n_ve_withholding` + portal (NUEVO)

**Qué hace:** estilo Profit Plus — al pagar, el wizard calcula y descuenta la retención
automáticamente (IVA, ISLR, municipal), sin paso manual posterior. El **portal** deja al
proveedor descargar sus comprobantes.

**Configuración:**
1. Ajustes → Contabilidad → *Retención en el pago*: activar por tipo (IVA/ISLR/municipal)
   y asignar cuenta + impuesto técnico de cada uno.
2. Dar acceso portal al proveedor (Contactos → Conceder acceso al portal).

**Manual de uso:**
1. Factura de proveedor validada → *Registrar pago* → el wizard muestra la retención
   calculada y el neto a pagar.
2. Confirmar → un solo flujo deja: pago neto + comprobante de retención.
3. Portal: el proveedor entra → *Mis retenciones* → descarga PDF.

**Checklist RP:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| RP-01 | Pago de factura 1.160 (IVA 160) con retención 75% activa | Neto pagado 1.040; retención 120 | |
| RP-02 | Comprobante generado desde el pago | Con correlativo y asiento cuadrado (Bs y USD) | |
| RP-03 | Desactivar el flag por compañía | El wizard paga completo, sin retención | |
| RP-04 | Portal del proveedor | Ve SOLO sus comprobantes; descarga PDF con marca del cliente | |
| RP-05 | Compañía foránea | El wizard no ofrece retención | |

---

## §4 · Multi-compañía base y productos — `nks_base_multi_company` + `nks_product_multi_company` (NUEVO — equipo)

**Qué hace:** permite que registros base (productos, y los modelos que se adhieran)
pertenezcan a **varias compañías a la vez** (campo compañías M2M con abstracción), en vez
de "una o todas".

**Configuración:** instalar ambos módulos; en el producto aparece el campo *Compañías*.

**Manual de uso:** Producto → pestaña Información general → *Compañías*: elegir 2 de las 4.

**Checklist MC:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| MC-01 | Producto asignado a Compañía A y B | Visible en A y B; invisible en C | |
| MC-02 | Producto sin compañías | Visible en todas (comportamiento global) | |
| MC-03 | Usuario mono-compañía intenta usar producto ajeno | Bloqueado por regla de registro | |
| MC-04 | Facturar el producto compartido en A y en B | Ambas facturas validan sin error de compañía | |

---

## §5 · Números de Control — `nikko_control_number` (CAMBIO MAYOR: medio de emisión + series + guía)

**Qué hace:** el correlativo legal SENIAT. Tres novedades del refactor:
1. **Medio de emisión fiscal POR DIARIO** (forma libre / facturación digital / impresora
   fiscal / talonario de contingencia).
2. **Serie por diario = sucursal** para Factura/ND/NC (`00-`, `01-`…), con fallback a la
   configuración de compañía; correlativo compartido o separado por tipo.
3. **Guía de despacho** con número de control propio, serie por **almacén**.

**Configuración:**
1. Contabilidad → Configuración → **Números de Control**: revisar la config por defecto
   (sin diario/almacén). Para una sucursal: nueva línea, tipo *Factura de Venta*, elegir
   **Diario** de esa sucursal y una secuencia con prefijo propio (p. ej. `01-`).
2. Para la guía: línea tipo *Guía de Despacho*; **Almacén** vacío = serie única de
   compañía, o un almacén concreto para serie propia.
3. En cada **diario de venta** → Ajustes avanzados → *Emisión Fiscal (Venezuela)* →
   seleccionar el **Tipo de emisión fiscal**.

**Manual de uso:**
- Factura en diario *forma libre* → al validar recibe `Nro. Control` de su serie.
- Diario *digital* → valida SIN número (lo asignará la imprenta al emitir, ver §6).
- Diario *contingencia* → exige capturar el número del talonario preimpreso.
- Entrega de inventario (picking de salida) → al validar recibe número de guía.

**Checklist NC:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| NC-01 | Factura en diario forma libre | Nro. control correlativo de su serie | |
| NC-02 | Factura+ND+NC con secuencia compartida | Un solo correlativo entre los 3 tipos | |
| NC-03 | Dos diarios con series `00-` y `01-` | Correlativos independientes, prefijo correcto | |
| NC-04 | Crear dos series con prefijos que choquen (p. ej. `0` y `00`) | Bloqueado con mensaje claro | |
| NC-05 | Diario digital: validar factura | Sin nro. de control local (queda para la imprenta) | |
| NC-06 | Diario contingencia sin número manual | Bloquea la validación pidiendo el número | |
| NC-07 | Validar entrega (salida) con config de guía | Guía recibe su número (serie del almacén o compañía) | |
| NC-08 | Validar una recepción | NO recibe número de guía | |
| NC-09 | Almacén con serie propia vs almacén sin serie | El primero numera con la suya; el segundo con la de compañía | |
| NC-10 | Duplicar una factura | La copia nace SIN número de control | |
| NC-11 | Compañía foránea | No asigna números de control | |

---

## §6 · Facturación Digital — `nks_einvoice_ve` (NUEVO)

**Qué hace:** emite factura, NC/ND, guía de despacho y comprobantes de retención a la
**imprenta digital** (Prov. 102, driver The Factory HKA), integrado a *Enviar e Imprimir*.
Cola asíncrona con reintentos, log de auditoría, dashboard. El número que asigna la
imprenta **aterriza en el Nro. Control** del documento (lo leen los libros).

**Configuración:**
1. Ajustes → *Facturación Digital*: activar **Emite Factura Digital (SENIAT)** (interruptor
   maestro de la compañía).
2. Contabilidad → Emisión Fiscal → **Configuración de Emisión**: proveedor (driver),
   ambiente (demo/producción), credenciales, serie.
3. El/los diarios del canal digital en *Tipo de emisión = Facturación digital* (§5).
4. Para guías digitales: en el cliente, activar *Guía Digital Habilitada*.

**Manual de uso:**
1. Factura validada en diario digital → **Enviar e Imprimir** → marcar *Factura Digital
   (SENIAT)* → Enviar.
2. La factura muestra el estado de emisión (En cola → Emitido) y, al emitirse, su número
   de control de la imprenta + URL de consulta.
3. Contabilidad → Emisión Fiscal: dashboard, cola de trabajos (reintentos), log inmutable.

**Checklist EFD** (en staging el driver está en demo; valida FLUJO y estados):

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| EFD-01 | Enviar e Imprimir en diario digital | Aparece la opción *Factura Digital (SENIAT)* | |
| EFD-02 | Diario forma libre | NO aparece la opción digital | |
| EFD-03 | Flag maestro de compañía apagado | Ningún diario ofrece digital | |
| EFD-04 | Emisión exitosa (o simulada) | `Nro. Control` = número de la imprenta; estado *Emitido* | |
| EFD-05 | Factura emitida en el Libro de Ventas | Sale con el número de la imprenta | |
| EFD-06 | Error de emisión | Estado *Error* con mensaje; el cron reintenta; la factura contable NO se afecta | |
| EFD-07 | Cliente "solo Bolívares" | El documento digital va sin bloque USD | |
| EFD-08 | Log de emisión | Cada intento con payload/respuesta; no editable | |

---

## §7 · Cobros Distribuidos — `nikko_cobros_clientes` (CAMBIO MAYOR: rescate)

**Qué hace:** UN cobro global del cliente contra N facturas, mezclando pagos nuevos,
créditos existentes y NC. El **diferencial cambiario** (tasa factura vs tasa cobro) se
materializa como **ND/NC fiscal con IVA por línea**; si es < 1 Bs, asiento directo. El
excedente queda como saldo a favor. Cancelación con reversa completa (solo gerentes).

**Configuración:** Ajustes → Contabilidad → *Cobros Distribuidos NKS*: diario de ND/NC del
diferencial e impuesto exento por defecto.

**Manual de uso:**
1. Contabilidad → Clientes → **Cobros Distribuidos** → Nuevo → cliente, moneda, monto, tasa.
2. Pestaña *Cobros*: líneas de pago (nuevo por diario, o crédito existente del cliente).
3. Pestaña *Asignación de Documentos*: facturas/NC a saldar (botones *Asignar residual
   completo* / *Distribuir en partes iguales*).
4. Confirmar → Validar. Revisar smart buttons: facturas, pagos, ND generada.
5. Cancelar uno validado (como gerente): todo se revierte.

**Checklist COB:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| COB-01 | Cobro USD contra 2 facturas a tasa vieja | Facturas en *Pagado*; ND por el diferencial con IVA por línea | |
| COB-02 | ND del diferencial | Tiene número de control y cuadra en Bs y USD | |
| COB-03 | Diferencial < 1 Bs | NO genera ND: asiento directo | |
| COB-04 | Cobro mayor que lo asignado | Excedente queda como saldo a favor del cliente | |
| COB-05 | Usar ese saldo a favor en un cobro nuevo | Se aplica como crédito existente | |
| COB-06 | Cobro con NC del cliente en las líneas | La NC resta del neto y se concilia | |
| COB-07 | Cobro en divisa con IGTF | Línea IGTF en el pago; todo cuadra | |
| COB-08 | Cancelar cobro validado (gerente) | Facturas vuelven a abiertas; ND revertida; pagos eliminados | |
| COB-09 | Usuario NO gerente intenta cancelar validado | Bloqueado | |
| COB-10 | Validar dos veces (doble clic) | Idempotente: no duplica pagos ni ND | |

---

## §8 · Anticipos — `nks_anticipos` (NUEVO)

**Qué hace:** anticipos de **cliente** y a **proveedor** SIN factura, sobre el pago nativo.
Controla el saldo disponible y la aplicación posterior a facturas; estados
borrador/publicado/parcial/aplicado; IGTF si es en divisa.

**Configuración:** ninguna especial (usa diarios de pago existentes). Grupos: cualquier
usuario de Facturación lo ve; los gestores contables pueden cancelar.

**Manual de uso:**
1. Contabilidad → Clientes → **Anticipos de cliente** → Nuevo → cliente, diario, moneda,
   monto → *Publicar* (crea y postea el pago).
2. Con facturas ya emitidas: botón *Aplicar* → elegir facturas → el saldo baja y el estado
   avanza (parcial → aplicado).
3. Proveedores → **Anticipos a proveedor**: espejo exacto.

**Checklist ANT:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| ANT-01 | Anticipo cliente USD 100 publicado | Pago creado; saldo disponible 100 USD (no Bs) | |
| ANT-02 | Aplicar a factura de 60 | Estado *Parcial*, saldo 40 | |
| ANT-03 | Aplicar el resto a otra factura | Estado *Aplicado*, saldo 0 | |
| ANT-04 | Aplicar dos veces a la misma factura | Idempotente, no duplica | |
| ANT-05 | Anticipo a proveedor | Flujo espejo completo | |
| ANT-06 | Anticipo en divisa con IGTF | El pago lleva su IGTF | |
| ANT-07 | Cancelar con saldo ya aplicado | Bloqueado o reversa limpia (según estado), nunca descuadre | |

---

## §9 · Importaciones — `nks_importaciones` + 4 satélites (NUEVO)

**Qué hace:** el expediente de importación completo (7 estados), sobre costos en destino
nativos: catálogo arancelario, exoneraciones por decreto, permisología con vigencias,
**tributos** (arancel sobre CIF a la tasa del DUA, tasa de régimen 1%, IVA de importación
deducible o al costo), distribución multi-criterio, costo provisional y **reliquidación**,
ISLR por servicios al exterior, puente al Libro de Compras.

**Configuración:**
1. Ajustes → *Importaciones*: impuesto de IVA de importación, % tasa de régimen (1.0),
   default de IVA no deducible.
2. Inventario → Configuración: catálogo arancelario (códigos HS con gravamen), exoneraciones
   (decreto + vigencia + %), tipos de permiso (SACS/SENCAMER) por régimen.
3. En el producto: su código arancelario.

**Manual de uso:**
1. Compras: OC al proveedor del exterior → confirmar → recibir.
2. Importaciones → **Expedientes** → Nuevo → vincular la OC → incoterm y moneda.
3. Avanzar estados: *Confirmar embarque* (avisa permisos faltantes sin bloquear) →
   *Llegada a puerto* → capturar **datos del DUA** (número, planilla, fecha, tasa).
4. Cargar costos (flete, seguro, gastos) → **Liquidar tributos**: el sistema calcula
   arancel + tasa + IVA sobre el CIF en Bs a la tasa del DUA.
5. Distribuir costos al inventario (por valor/peso) → cerrar expediente (o dejarlo
   provisional y **reliquidar** después).

**Checklist IMP:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| IMP-01 | CIF 10.000 USD, tasa DUA 50, arancel 20% | Base 500.000 Bs; arancel 100.000 Bs | |
| IMP-02 | Tasa de régimen | 1% del CIF en Bs (5.000) | |
| IMP-03 | IVA de importación deducible | Va a crédito fiscal, NO al costo | |
| IMP-04 | Marcar IVA no deducible (Art. 33) | El IVA se suma al costo del producto | |
| IMP-05 | Producto régimen 3 sin registro sanitario | Aviso de permiso faltante; NO bloquea | |
| IMP-06 | Permiso vigente registrado | El aviso desaparece; permiso vencido vuelve a avisar | |
| IMP-07 | Exoneración 90% vigente en la partida | El arancel baja al 10%; vencida NO aplica | |
| IMP-08 | Distribución por peso entre 2 productos | El pesado absorbe más costo | |
| IMP-09 | Vender producto de expediente provisional | Aviso de costo provisional en el pedido | |
| IMP-10 | Reliquidar tras la venta parcial | Ajuste al costo de lo vendido (asiento) | |
| IMP-11 | Factura de flete del exterior | Retención ISLR automática (Dcto. 1.808) | |
| IMP-12 | Libro de Compras del mes | La importación sale con formato de importación | |

---

## §10 · Pedido → Compra — `nks_sale_to_purchase` (NUEVO, premium)

**Qué hace:** desde el pedido de venta genera la OC al proveedor **elegido por línea**.

**Configuración:** Ajustes → Ventas → activar *Pedido a Compra*; producto con proveedores
en su ficha (pestaña Compra) y la ruta configurada.

**Manual de uso:** pedido de venta → en la línea, elegir el proveedor → confirmar pedido →
se crea la OC enlazada.

**Checklist S2P:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| S2P-01 | Confirmar pedido con proveedor elegido | OC creada a ESE proveedor, enlazada al pedido | |
| S2P-02 | 2 líneas con proveedores distintos | 2 OCs, una por proveedor | |
| S2P-03 | Línea sin proveedor elegido | Usa el proveedor por defecto del producto | |
| S2P-04 | Flag de compañía apagado | No genera OC | |

---

## §11 · Tesorería bancaria — `l10n_ve_bank_hub` + `l10n_ve_bank_format` (NUEVO)

**Qué hace:** catálogo de bancos VE, credenciales cifradas, **lotes de pago a proveedores**
con separación de funciones (quien monta NO aprueba) y **TXT de dispersión** por banco.

**Configuración:** Contabilidad → Tesorería → Bancos: configurar el banco pagador (formato
TXT, cuenta). Asignar los grupos *Montador* y *Aprobador* a usuarios DISTINTOS.

**Manual de uso:**
1. Tesorería → **Lotes de pago** → Nuevo → seleccionar facturas de proveedor a pagar.
2. *Montar* (usuario montador) → *Aprobar* (usuario aprobador) → descargar el **TXT**.

**Checklist BH:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| BH-01 | Montar lote con 3 facturas | Total correcto; estado *Montado* | |
| BH-02 | El MISMO usuario intenta aprobar su lote | Bloqueado (maker-checker) | |
| BH-03 | Aprobar con otro usuario | Estado *Aprobado*; TXT descargable | |
| BH-04 | TXT generado | Estructura del banco correcta (cuenta, RIF, montos sin descuadre) | |
| BH-05 | Auditoría del lote | Chatter registra quién montó y quién aprobó | |

---

## §12 · Tasa de mercado — `nks_market_rates` (NUEVO, premium)

**Qué hace:** tercera moneda informativa (USD a tasa de mercado / Binance P2P) SOLO para
dashboards y reportes. **La contabilidad nunca la usa.**

**Configuración:** Ajustes → activar *Tasas de mercado*; fijar la métrica y desviación
máxima permitida.

**Checklist MKT:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| MKT-01 | Lente *Mercado* en un reporte | Montos a tasa de mercado, distintos del lente Documento | |
| MKT-02 | Ver un asiento contable | En Bs/tasa BCV; la tasa de mercado NO aparece en asientos | |
| MKT-03 | Desactivar el flag | El lente mercado desaparece de los selectores | |

---

## §13 · ISAE / Patente municipal — `l10n_ve_isae` (NUEVO)

**Qué hace:** declaración periódica del Impuesto sobre Actividades Económicas: base =
**lo cobrado** en el período (criterio percibido) × alícuota, contra el mínimo tributable
(paga "el mayor"). Genera el asiento del gasto/pasivo.

**Configuración:** Ajustes → *ISAE*: cuenta de gasto, cuenta por pagar, diario, alícuota y
mínimo de la ordenanza municipal.

**Manual de uso:** Contabilidad → Declaraciones → **ISAE** → Nueva → período → *Calcular* →
revisar base percibida → *Confirmar* (asiento).

**Checklist ISAE:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| ISAE-01 | Cobros del período 10.000 (sin IVA), alícuota 1% | Impuesto 100 | |
| ISAE-02 | Impuesto calculado < mínimo tributable | Paga el mínimo | |
| ISAE-03 | Factura emitida pero NO cobrada en el período | NO entra en la base (percibido) | |
| ISAE-04 | Asiento de la declaración | Gasto vs por pagar, cuadre dual | |

---

## §14 · RESQUIMCF — `l10n_ve_resquimcf` (NUEVO)

**Qué hace:** resumen de compras/ventas por TIPO de documento (Factura/ND/NC) con bases e
impuestos, para la declaración informativa.

**Manual de uso:** Contabilidad → Informes → RESQUIMCF → período → generar.

**Checklist RQ:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| RQ-01 | Mes con facturas, ND y NC | Tres grupos con subtotales correctos | |
| RQ-02 | NC restan | Los totales netos descuentan las NC | |
| RQ-03 | Cruce contra Libro de Ventas | Totales consistentes entre ambos | |

---

## §15 · Carga histórica — `nks_backdating` (CAMBIO)

**Qué hace:** registrar ventas, entregas, facturas y órdenes de producción **con fechas
pasadas** (migraciones), manteniendo numeración y contabilidad coherentes.

**Configuración:** Ajustes → activar *Carga Histórica* en la compañía (solo mientras se
migra; apagar al terminar).

**Checklist BD:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| BD-01 | Factura con fecha del mes pasado | Asiento y libro en ese mes | |
| BD-02 | Entrega con fecha pasada | Movimiento de stock a esa fecha | |
| BD-03 | Flag apagado | Vuelve el comportamiento estándar de fechas | |

---

## §16 · Restricciones fiscales — `nks_vzla_fiscal_restr` (NUEVO)

**Qué hace:** el guardián SENIAT: bloquea la edición de documentos fiscales validados,
muestra el **banner de estado fiscal** en ventas (alerta escalada por período de IVA) e
IGTF informativo.

**Checklist FR:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| FR-01 | Editar factura validada (campos fiscales) | Bloqueado con mensaje | |
| FR-02 | Banner en pedidos/facturas | Visible con el estado del período fiscal | |
| FR-03 | Compañía foránea | Sin restricciones ni banner | |

---

## §17 · Plataforma — `nks_access_control` + `nks_security_center` + `l10n_ve_nikkosoft_master` (NUEVO)

**Qué hace:**
- **Access PRO:** perfiles que ocultan/restringen campos, menús, botones y vistas por
  grupo, sin Studio. (Configuración → NKS Access → Perfiles.)
- **Security PRO:** auditoría de inicios de sesión, firewall de IPs, bloqueo del modo
  desarrollador. (Configuración → NKS Security.)
- **Datos maestros:** plan de cuentas/bancos/impuestos/diarios NIKKOSOFT para arrancar
  clientes.

**Checklist SEC:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| SEC-01 | Perfil que oculta un menú a un grupo | El usuario de prueba deja de verlo | |
| SEC-02 | Perfil que vuelve un campo solo-lectura | El campo no se puede editar para ese grupo | |
| SEC-03 | Log de sesiones | Registra usuario, IP y fecha de cada login | |
| SEC-04 | Bloqueo de modo desarrollador para un grupo | `?debug=1` no activa devtools para ese usuario | |
| SEC-05 | Plan de cuentas maestro | Cuentas NIKKOSOFT presentes y con los impuestos mapeados | |

---

## §18 · Analítica — `nks_ventas_dashboard` + tableros por lente (NUEVO)

**Qué hace:** cubo/pivote de ventas (cliente × producto × mes, Bs/USD, incluye migradas) y
tableros de spreadsheet con **variantes por lente de moneda** (Bs, USD documento, USD día,
mercado) en el panel lateral.

**Checklist DASH:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| DASH-01 | Cubo de ventas: cliente × mes en USD | Totales consistentes con las facturas | |
| DASH-02 | Tablero: variante USD documento vs USD día | Montos difieren coherentemente con las tasas | |
| DASH-03 | Facturas migradas | Aparecen en el cubo | |

---

## §19 · Licores — `nikko_licores_ve` (NUEVO)

**Qué hace:** impuesto del ramo de licores (LIAE) por grado alcohólico / PVP, con los
datos del registro de licores en el producto.

**Checklist LIC:**

| # | Verificación | Resultado esperado | ✅/❌ |
|---|---|---|---|
| LIC-01 | Producto de licor con datos LIAE | Campos de grado/registro visibles y obligatorios donde aplique | |
| LIC-02 | Factura del licor | El impuesto del ramo se calcula según la configuración | |

---

## Cierre de la validación

| Área | Secciones | Responsable | Estado |
|---|---|---|---|
| Contable base | §1, §15 | | |
| Fiscal | §2, §3, §5, §6, §13, §14, §16, §19 | | |
| Operación | §7, §8, §9, §10, §11 | | |
| Plataforma | §4, §12, §17, §18 | | |

**Regla final:** una sección se da por validada cuando TODAS sus filas están en ✅ o cada ❌
tiene su reporte creado. Al terminar, enviar este documento marcado al canal del proyecto.

---
*NIKKOSOFT CONSULTORES C.A. — Localización Venezolana NKS · rama 18.0-refactor · 2026-07-15*
