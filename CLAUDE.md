# JiTel Capital — Contexto para Claude

## Descripción del negocio

**JiTel Capital** es una app de gestión de cartera de préstamos informales en Colombia. Permite registrar préstamos a clientes, llevar el cobro mensual de intereses, registrar los giros (pagos) a socios inversores, generar imágenes para WhatsApp y hacer backups en Google Drive.

Los préstamos generan interés mensual cobrado al cliente (`int_socio + int_propio`). El socio inversor recibe `int_socio × monto` y el propietario retiene `int_propio × monto` como ganancia neta.

---

## Arquitectura

- **Tres páginas, sin framework ni build system**:
  - `index.html` (~5.200 líneas) — app de administración (solo admin).
  - `socio.html` — portal de solo lectura para cada socio (login con email/contraseña).
  - `dayam.html` — portal público de solo lectura para el cliente DAYAM (sin login).
- **JS embebido** en `<script>` al final de cada HTML; estilo ES5 estricto (sin `const`/`let`, sin arrow functions, sin template literals)
- **Persistencia**: **Firebase** (Auth + Firestore) como fuente principal; `localStorage` (`jitel_v1`) se mantiene como red de seguridad / fallback. Ver sección "Firebase" abajo.
- **Despliegue**: GitHub Pages (rama `main`) del repo `jitelcapital/jitel-capital`. El link del portal de socio que copia la app es `https://jitelcapital.github.io/jitel-capital/socio.html`.
- **Sin librerías externas**: SVG nativo, Canvas nativo, Google Identity Services vía CDN para OAuth2

---

## Las 5 pestañas

| Pestaña | Función |
|---------|---------|
| **Cobros** | Vista del mes agrupada por día de cobro (acordeones). Pago rápido/por grupo, toggle "Cobré", botón "🏦 Pagué socio" (abre el modal de giro), sumador temporal, mensaje multi-día. Sección "📦 Pendientes del mes anterior" (cuotas arrastradas) y aviso "Cobrado sin girar hace más de 3 días". Los préstamos que aún no se cobran (`!isCollectable`) se ven en su día como tarjeta atenuada "🆕 Nuevo" (ver `vCobros` en "Traspasos y cierre de préstamos"). |
| **Resumen** | Dashboard ejecutivo: capital activo, ganancia mensual, mora, KPIs, "💸 Adelantos por recuperar", gráfico histórico SVG (últimos 6 meses), preferencias, backup Google Drive, "Nuevo mes" (cierre mensual). |
| **Préstamos** | Lista completa (activos + cerrados) con búsqueda y filtros. Cada fila abre `showLoanDetail`: historial de pagos, registrar pago (con periodo), Actualizar/Extender (plazo, mes de inicio, socio, tasas) y cerrar préstamo. |
| **Socios** | Por socio: capital, interés mensual, préstamos, finalizados, sección **Giros** por mes, historial mensual (`ST.snapshots`), botón "🏦 Registrar giro", imagen paginada para WhatsApp. Aviso "Cobrado sin girar" arriba. Los socios inactivos van al final en una sección plegada (ver "Socios inactivos"). |
| **Clientes** | Por cliente/responsable: préstamos activos, cuota total mensual. Imagen paginada (10 filas/página) con estado de cuenta. |

---

## Funciones clave

### Cálculo de intereses
```js
function IM(l)  // Interés mensual total cobrado al cliente = monto × (int_socio + int_propio)  (moneda nativa)
function IS(l)  // Interés del socio inversor              = monto × int_socio
function IP(l)  // Interés propio (ganancia neta)          = monto × int_propio
// Versiones en COP (convierten USD con la tasa fija del préstamo): IMc, ISc, IPc, montoCOP
```

### Detección de préstamos directos
```js
function isJulianLoan(l)
// true si socio==='JULIAN' && int_socio===0
// Préstamo directo del propietario sin socio externo
// SIEMPRE usar esta función — nunca comparar socio o int_socio por separado
```

### Periodo de pago y estado del mes
```js
function pagoPeriodo(p)            // 'YYYY-MM' de la CUOTA que paga: p.periodo, o el mes de p.fecha (pagos viejos)
function paidMth(l)                // true si l.pagos tiene un pago con periodo === CURR_MK
function hasPagoForPeriodo(l,mk)   // busca en l.pagos Y en ST.archives[mk][l.id]
function isCollectableFor(l,mk)    // false si mes_comienzo >= mk (el primer cobro es el mes siguiente al inicio)
function isCollectable(l)          // isCollectableFor(l, CURR_MK)
function aplicarPagoALoan(c,pago)  // punto ÚNICO para registrar un pago en una copia del préstamo
```

### Persistencia
```js
function saveST()    // localStorage (siempre) + sync Firestore con debounce (solo admin) — llamar tras toda mutación de ST
function renderApp() // Re-renderiza la vista activa completa — COLAPSA acordeones abiertos
                     // Preferir actualizaciones DOM in-place cuando sea posible
```

### WhatsApp (`openWhatsApp(texto, telefono?)`)
Punto ÚNICO para abrir WhatsApp; no usar `window.open`/`_blank` en móvil (ni en `socio.html`/`dayam.html`).
- **iPhone/iPad** (`esIOS()`, incluye iPadOS por `MacIntel` + `maxTouchPoints>1` y la app abierta desde el ícono de inicio): solo `location.href='whatsapp://send?…'`, **sin** respaldo temporizado a wa.me (dejaba una hoja en blanco al volver).
- **Android** (`esAndroid()`): `whatsapp://send?…` + respaldo a `https://wa.me/` en la misma pestaña si a los 1,5 s (`WA_FALLBACK_MS`) la página sigue visible.
- **Computador**: `https://web.whatsapp.com/send?…` en pestaña nueva.

### Mensajes al socio sobre pagos/cuotas
Todo mensaje al socio que hable de un pago o cuota identifica **cada préstamo** con `socioLoanLines(loan, periodo, monto)`, igual que lo ve en `socio.html`:
```
• $ 20.000.000 · Día 5 · Mar 2026 → Mar 2027     ← capital COP (montoCOP) · día · fmtPlazoSocio (= fmtPeriodo de socio.html)
   Interés/mes: $ 600.000 (3%)                   ← ISc y tasaSocioPct (solo int_socio)
   Cuota de noviembre 2026: $ 600.000            ← "Monto: $X" si no hay periodo (giros de capital)
```
- Lo usan `waGiroMsg` (vía `giroLineas`, que ahora devuelve también `loan` y `periodo`), `waSocioMsg(name, dia, amt, fecha, periodo, loans)` (modal "Notificar a socios" y copia rápida de un préstamo) y el mensaje multi-día (`showMensajeMultidia`, debajo de cada "Día X").
- **Nunca** incluir el nombre del cliente, la tasa del cliente (`int_socio + int_propio`) ni `int_propio`. El resto del mensaje (monto, fecha, modalidad, referencia) no cambia.

### Mensajes de préstamo nuevo
Al guardar en +Nuevo (`showNewLoan`) se abre `showNewLoanMessagesModal(loan)`, con el mismo patrón que `showCloseMessagesModal` (WhatsApp / Copiar / Ver mensaje):
- **Al socio** (`waNuevoSocioMsg`): "Hola X 👋, tienes un préstamo nuevo", el bloque de `socioLoanLines` sin la línea de cuota (capital, día, plazo, interés/mes y tasa del socio) y "Primer pago: Mes AAAA" (`primerPagoMk`: mes siguiente a `mes_comienzo`). Sin nombre del cliente ni `int_propio`. Una continuación se anuncia igual, **sin** mencionar traspaso ni continuación. No se ofrece si `isJulianLoan`.
- **Al cliente** (`waNuevoClienteMsg`): capital y cuota mensual (`cobrarCliente`, con `fmtDual` si es USD), día de pago, primer pago y vencimiento (`mes_fin` + `año`). Sin socios ni tasas. **No se ofrece si es continuación** (`continuacion_de`): para el cliente nada cambia.
- Préstamo de JULIAN que además es continuación: no se muestra el modal.

---

## Modelo de datos `ST`

```js
var ST = {
  loans: [/* préstamos */],
  archives:  { "2026-09": { loanId: [pagos...] } },   // pagos archivados por mes al cerrar
  snapshots: { "2026-09": { SOCIO: {capital, count, intS, loanIds} } }  // foto mensual por socio
};
```

### Campos de cada préstamo (`ST.loans[i]`)

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | string | Generado con `GID()` |
| `año` | number | Año de vencimiento |
| `dia` | number | Día de cobro mensual (1–31) |
| `mes_fin` | string | Mes de vencimiento en español mayúsculas (e.g. `'JULIO'`) |
| `mes_comienzo` | string\|null | Mes de inicio `'YYYY-MM'`. El primer cobro es el **mes siguiente**. Editable desde Actualizar (no cuenta como extensión; queda en `nota`) |
| `monto` | number | Capital en la moneda nativa |
| `moneda` / `tasa_cambio` | string / number\|null | `'COP'` (default) o `'USD'` con tasa fija COP por USD |
| `socio` | string | Nombre del socio inversor (e.g. `'VICTOR'`, `'JULIAN'`) |
| `socio_email` | string\|null | Email del socio (de la colección `socios`); lo agrega `loanToDoc` al escribir en Firestore. Las reglas filtran por este campo |
| `int_socio` / `int_propio` | number | Tasas mensuales (e.g. `0.035`) |
| `responsable` | string | Nombre del cliente/deudor |
| `status` | string | `'active'` o `'paid'` (cerrado) |
| `pagos` | array | Pagos del ciclo en curso: `[{id, tipo, monto, fecha, periodo, nota, monto_usd?}]` |
| `recibido` | boolean | Conciliación: cliente pagó este mes (toggle "Cobré") |
| `pagado_socio` | boolean | Giro al socio del ciclo en curso. **Solo lo pone un giro** (ver "Giros") |
| `cuotas_pendientes` | array | Cuotas arrastradas de meses cerrados: `[{id, periodo, recibido, pagado_socio}]` |
| `veces_extendido` / `mes_fin_original` | number / string | Extensiones de plazo |
| `cierre_tipo` / `fecha_cierre` | string / string | Al cerrar: `'finalizado'` o `'traspaso'`, y fecha ISO |
| `continuacion_de` | string | id del préstamo original cuando este es su continuación (traspaso) |
| `nota` | string | Nota libre; acumula cambios de plazo, inicio, socio y tasas |
| `created_at` | string | ISO timestamp de creación |

### Pagos (`pagos[i]`)
- `tipo`: `'interes'` (el más común), `'abono'` (abono a capital), `'total'` (pago total, cierra el préstamo).
- `periodo`: `'YYYY-MM'` de la cuota que paga (puede ser un mes anterior: pago tardío o cuota arrastrada). Pagos viejos sin `periodo` usan el mes de `fecha`.
- Si el periodo ya está archivado, `aplicarPagoALoan` también lo agrega en `ST.archives[periodo]`.
- "Mes de apertura": al crear un préstamo cuyo día de cobro ya pasó y cuyo inicio es el mes actual, se inserta un pago simbólico con esa nota.

---

## Cierre mensual ("Nuevo mes", `showReset`)

- Se **elige el mes a cerrar** (sugerido: el mes anterior al actual).
- **Protecciones** (`bloqueoCierre(mk)`; si alguna aplica, el paso 2 muestra el motivo y **no** hay botón para confirmar; `doReset` vuelve a comprobarlo):
  1. **Mes ya cerrado**: si `ST.archives[mk]` existe → "Octubre 2026 ya fue cerrado". No existe opción de sobrescribir: cerrarlo otra vez borraría los pagos archivados de los activos y volvería a marcar como "falta girar" cuotas ya giradas.
  2. **Mes que no ha terminado**: `mk >= CURR_MK` → solo se puede cerrar desde el 1 del mes siguiente.
- **Pagos de meses posteriores** (`pagosPosterioresA`): préstamos activos con pagos de periodo > `mk`. **No bloquean**: el paso 2 los muestra como aviso ("📌 N pagos de meses posteriores…"), porque el cierre los deja en el préstamo. Si alguno en realidad era de `mk`, se corrige su periodo antes de cerrar (ver "Corregir el periodo de un pago").
- Antes de confirmar muestra las cuotas sin cobrar o sin girar del mes (`loansPendientesParaCierre`), las **cobradas sin girar hace más de 3 días** por socio, y un **recordatorio para exportar el backup** (botón "💾 Exportar backup ahora").
- **No se puede deshacer** desde la app: la única vuelta atrás es restaurar el backup exportado justo antes.
- Conviene hacerlo desde **un solo dispositivo**. Una sesión abierta con datos viejos ya no reescribe la nube (ver "Control de versión"): su siguiente guardado se rechaza, avisa y recarga.
- Al confirmar (`doReset`), en este orden:
  1. `generarArrastrePendientes(mk)`: cada cuota sin resolver del mes pasa a `loan.cuotas_pendientes` con su `recibido`/`pagado_socio` (`estadoCuotaCierre`: los flags del préstamo; si el préstamo ya tiene pagos de meses posteriores, `recibido = hasPagoForPeriodo(l, mk)` y `pagado_socio = !!giroDeCuota(l.id, mk)`, porque los flags pueden ser de esos meses). `loansPendientesParaCierre` usa el mismo criterio.
  2. `ST.snapshots[mk] = buildSociosSnapshot()`.
  3. `ST.archives[mk]` = pagos de todos los préstamos con periodo **<= `mk`**.
  4. Préstamos activos: se quitan de `pagos` los archivados y **se quedan los de periodo > `mk`**. `recibido` y `pagado_socio` pasan a ser los del ciclo que sigue abierto (`nextMk(mk)`): `recibido` = queda un pago de ese periodo; `pagado_socio` = un giro vigente cubre esa cuota (`giroDeCuota`), así que lo sigue poniendo solo un giro. Los cerrados conservan todos sus pagos.
- Un pago tardío (periodo < `mk` ya archivado) que sigue en `pagos` se archiva también bajo `mk` al cerrarlo, como antes. Las comprobaciones por periodo (`hasPagoForPeriodo`) no se confunden, porque miran `pagoPeriodo`.

### Corregir el periodo de un pago
- En `showLoanDetail` → "Historial de pagos", cada pago muestra "Cuota de MES AÑO" y, **solo para admin** (`isAdmin()`), el botón "✏️ Periodo": mes nuevo + "Guardar periodo" + confirmación (`showConfirm`).
- `corregirPeriodoPago(lid, pagoId, nuevo)` cambia `pago.periodo`, lo saca de `ST.archives[viejo][lid]`, lo agrega a `ST.archives[nuevo][lid]` si ese mes está cerrado (igual que `aplicarPagoALoan`), recalcula `recibido` de las cuotas arrastradas de ambos periodos con `hasPagoForPeriodo`, y el `recibido` del préstamo si alguno es `CURR_MK`.
- **No toca `pagado_socio`**. Si la cuota del periodo viejo ya tiene un giro vigente, se rechaza: primero hay que anular el giro.
- Una cuota arrastrada está resuelta cuando tiene pago para su periodo **y** (`pagado_socio` o `isJulianLoan`). Las sin resolver se ven en Cobros → "Pendientes del mes anterior".

---

## Giros a socios (colección `giros`)

Cada pago a un socio se registra como un **giro**. Arranca en **octubre de 2026** (`GIROS_DESDE`); no se importaron giros anteriores.

- **Sin giro no hay "Pagué socio"**: `pagado_socio` (del mes y de `cuotas_pendientes`) solo se pone en `true` al guardar un giro (`registrarGiro`) y vuelve a `false` solo al anular el giro que la cubría. `toggleConcil`, `conciliBtn` y `conciliBtnCuota` ignoran el campo `pagado_socio`; el botón es `giroBtn(loan, periodo)`.
- **Documento**: `{id, socio, socio_email, fecha, monto (COP), modalidad, modalidad_detalle, referencia, tipo, cuotas:[{loanId, periodo, monto, adelanto?}], nota, creado_en, anula_a?}`.
  - `modalidad`: `'transferencia' | 'efectivo' | 'abono_tarjeta' | 'otro'`.
  - `tipo`: `'intereses' | 'devolucion_capital' | 'aporte_capital'`. En capital, `cuotas` lleva los préstamos asociados con `periodo: null`.
  - `anula_a` solo existe en anulaciones.
- **Obligatorios**: referencia en transferencia y abono a tarjeta; a quién se entregó o nota en efectivo; `modalidad_detalle` en otro.
- **Nunca se editan ni se borran**. Para corregir: "Anular giro" pide motivo y crea un giro nuevo con `anula_a = <id>`. Un giro está anulado si existe otro con `anula_a` igual a su id (`giroVigente`). Las cuotas del anulado vuelven a pendientes de girar (si el mes ya estaba cerrado, se agrega una cuota arrastrada).
- **Estado girado de una cuota** (`cuotaGirada(l, periodo)`): la cuota arrastrada de ese periodo si existe; si el periodo no está archivado, `l.pagado_socio`; si está archivado y sin arrastre, se considera resuelta.
- **Cuotas por girar** (`cuotasPorGirar(socio)`): con pago del cliente registrado y sin girar (mes en curso + arrastradas). El aviso (`alertasSinGirar`) muestra las de más de 3 días.
- **Adelantos al socio** (girar una cuota que el cliente **aún no paga**):
  - En "Registrar giro" (`showGiroModal`), debajo de las cobradas, sección "💸 Adelantar (el cliente aún no paga)" con `cuotasAdelantables(socio)`: cuotas girables (`cuotaGirable`) del mes en curso (préstamo activo) y arrastradas, **sin pago del cliente** y sin girar. Desmarcadas por defecto; al tocar una aparece una confirmación dentro del modal ("Sí, adelantar" / "No") y solo entonces queda marcada.
  - En el giro esas cuotas van como `{loanId, periodo, monto, adelanto:true}`. `registrarGiro` no cambia: la cuota queda `pagado_socio=true` y `recibido=false`. `sanitizeGiro` conserva `adelanto`.
  - "Adelantada" es **derivado**, no se guarda en el préstamo: `giroAdelantoDeCuota(loanId, periodo)` (giro vigente que la cubre con `adelanto:true`) y sin pago del cliente (`hasPagoForPeriodo`). `cuotaEstadoCobros` devuelve `'adelantada'` → etiqueta "💸 Adelantada · falta cobrar" (`cqKind` → `'adel'`, morado `--st-adel`), con el botón de pago y la etiqueta "💸 Adelantado · fecha" del giro. Al registrar el pago del cliente queda "✓ Completo" normal. Sigue contando en mora (`paidMth` no cambia).
  - Resumen: "💸 Adelantos por recuperar" (`adelantosPorRecuperar()`): total y lista socio · cliente · día · cuota · fecha del giro. Desaparece cuando el cliente paga o se anula el giro.
  - Cierre: si sigue sin pago, `generarArrastrePendientes` la arrastra con `recibido:false, pagado_socio:true` ("Sin cobrar · ya girada" en la lista del cierre). Como ya está girada, **nunca vuelve a "por girar"**; en "Pendientes del mes anterior" se ve como adelantada hasta que el cliente pague.
  - `socio.html` la ve como pagada y nunca menciona el adelanto: su `cuotaResuelta` de arrastradas es solo `pagado_socio`. El mensaje de WhatsApp del giro (`waGiroMsg`) es igual que siempre.
- **Escritura**: fuera de `fbPushAll`. `fbCreateGiro` hace `set` con 2 reintentos (antes de reintentar comprueba si el doc ya existe, porque un segundo `set` sería un update y las reglas lo rechazan). Si falla, aviso rojo visible y cola `_pendingGiros`, que `flushPendingGiros` reintenta en cada `fbPushAll`.
- **Carga**: `loadGiros` al iniciar sesión de admin (Firestore + cache local `jitel_giros_v1`). En modo sin Firebase solo la cache local.
- **Vistas**: Socios → sección "Giros" por mes (generado = interés de las cuotas del periodo, girado = cuotas de ese periodo en giros vigentes, diferencia; anulados tachados con motivo). `socio.html` → "Mis giros" con el mismo cálculo. Después de guardar se ofrece el mensaje de WhatsApp (`waGiroMsg`, sin nombres de clientes; cada cuota identifica su préstamo, ver "Mensajes al socio sobre pagos/cuotas").
- **Comprobantes** (ver "Comprobantes de giros"): opcionales, aparte del giro; se conservan si el giro se anula.
- **Backup CSV**: fila especial `__GIROS__` (JSON), como `__ARCHIVES__`. Al importar solo se agregan los que falten; no se suben a la nube.
- El backup CSV (`exportBackupCSV` y `gdriveBuildCSV`) también incluye las fotos mensuales en la fila `__SNAPSHOTS__` (JSON de `ST.snapshots`). Al importar reemplaza `ST.snapshots`; un backup viejo sin esa fila conserva las fotos actuales.
- Compatibilidad: cuotas marcadas antes de los giros siguen marcadas (se ven "✓ Girado" sin giro asociado).

### Comprobantes de giros
- **`comprobantes/{giroId}`** → `{imagen, socio_email, creado_en, reemplazado_en}`. `imagen` es un dataURL JPEG comprimido en el navegador (`comprimirImagen`): lado mayor ≤ 1600 px (`COMP_MAX_LADO`) y dataURL ≤ 300 KB (`COMP_MAX_CHARS`); baja la calidad de 0.85 a 0.45 y, si no alcanza, reduce el tamaño un 15% por ronda. PNG con transparencia → fondo blanco.
- **`comprobantes_idx/{giroId}`** → los mismos campos **sin** `imagen`. Se escribe en el mismo `batch` que el comprobante. Sirve para saber qué giros tienen 📎 sin descargar imágenes: el admin lo carga en `bootData` (`loadComprobantesIdx` → `COMP_IDX`), y `socio.html` con `where socio_email == …`.
- **No van dentro del giro** (los giros son inmutables), **no se cargan al abrir la app** (solo el índice), no van a `localStorage` ni al backup CSV. La imagen se descarga solo al tocar 📎 (`verComprobante` → `showImageViewer(url, {instr, uso})`; cache en memoria `COMP_CACHE`).
- **Adjuntar**: en "Registrar giro" (`comprobantePicker`: botón que en iPhone abre Fotos, o pegar con Cmd/Ctrl+V mientras el modal está abierto); se guarda después de `registrarGiro`. O después desde el detalle del giro (`showGiroDetail` → `showAdjuntarComprobante`). **Reemplazar** pide confirmación (`showConfirm`) y pone `reemplazado_en` (conserva `creado_en`). Nunca se borran; las anulaciones (`anula_a`) no llevan comprobante; anular un giro conserva el suyo.
- Escritura (`guardarComprobante`): solo admin con Firebase; 2 reintentos y aviso rojo si falla (no hay cola: se vuelve a adjuntar desde el historial).
- **Historial (Socios → Giros)**: 📎 en las filas con comprobante (abre el visor) y botón "Sin comprobante" que filtra (`GIROS_SOLO_SIN_COMP`, todas las secciones).
- **`socio.html` → "Mis giros"**: "📎 Ver comprobante" solo en sus giros con comprobante (`COMP_IDX_S`), descargado al tocarlo (`verComprobanteSocio`, visor propio).
- **Carga de `socio.html` (`loadLoans`)**: `prestamos`, `giros` y `comprobantes_idx` se piden **en paralelo**. El portal se pinta en cuanto llegan préstamos y giros; el índice **no lo bloquea**: si llega después, `refrescarMisGiros` redibuja solo la tarjeta "Mis giros" (`MIS_GIROS`). Si falla, el portal queda sin 📎. A los 8 s sin respuesta el texto de carga avisa que la conexión está lenta. No volver a encadenar consultas en serie aquí.
- Métricas: `comprobante_adjuntar`, `comprobante_reemplazar`, `comprobante_ver`.

---

## Traspasos y cierre de préstamos

- "✓ Cerrar préstamo" (`showCloseLoanModal`) pone `status='paid'`, `cierre_tipo` (`'finalizado'` o `'traspaso'`) y `fecha_cierre`, y ofrece los mensajes de cierre (cliente y/o socio).
- Un préstamo nuevo puede marcarse como **continuación** de otro (`continuacion_de`) para traspasos de cartera entre socios.
- En `buildClienteImage` una continuación se ve como fila normal, sin etiquetas. En `dayam.html` sí hereda el estado del original.
- En la pestaña Socios (admin) los finalizados muestran "✅ Finalizado" o "✅ Liquidado en su totalidad" (traspaso), sin nombrar al cliente ni al socio entrante. El admin sigue viendo el traspaso con todo su detalle.
- **`socio.html` nunca muestra traspasos**: ningún texto visible con "traspaso", "continuación", "renovación" ni "liquidado", y todos los cerrados se ven como "✅ Finalizado".
- `socio.html` → **meses cobrados** (`mesesCobrados`): el mes de `mes_comienzo` no se cobra, se cobra desde el mes siguiente (un préstamo que empezó este mes suma 0). Se usa en "Rendimiento acumulado", "Generado hasta hoy", "Intereses recibidos"/"meses activos" de los finalizados y en `planProgress` (barra de progreso, meses del plan y "Si llega hasta el final"): un préstamo que empezó este mes muestra 0% con todo el plan por delante.
- `index.html` cuenta **igual que `socio.html`**:
  - `mesesCobrados(mesComienzo, uptoMk)` (misma fórmula) es la base de `loanMesesActivo`/`loanTotalInteresSocio`: pestaña Socios → "Préstamos finalizados" (meses activos e intereses recibidos) y mensajes de WhatsApp de cierre al socio ("CAPITAL FINALIZADO" / "LIQUIDADO EN SU TOTALIDAD"). Un préstamo de marzo a julio suma 4 meses.
  - Resumen (`vDashboard`): "Ganancia mes", "Mi cobrado"/"Mi pendiente" y "En mora" excluyen los préstamos que aún no se cobran (`!isCollectable`). "Capital activo" y "Préstamos activos" sí los incluyen.
  - Cobros (`vCobros`): los préstamos activos con `!isCollectable` van en `nuevos` → `g.nuevos` de su día (nunca en `g.loans`). Se dibujan al final del día con `nuevoCardCobros(ln)`: tarjeta atenuada `.cq-nuevo` con "🆕 Nuevo · primer cobro: <Mes AAAA>" (`nextMk(mes_comienzo)`), socio, cliente y capital, **sin** Pagar/Cobré/Girar ni checkbox del sumador; tocarla abre `showLoanDetail` (para corregir el inicio con Extender). **No cuentan** en Total mes/Cobrado/Mi cuenta, pendientes acumulados, banner de HOY, total del día, "N préstamos · cobrados · girados", "N avisados sin registrar", por girar ni mora (aunque tengan `recibido`). En el encabezado del día van aparte como chip "🆕 N nuevo(s)". Un día con **solo** nuevos (`soloNuevos`) se muestra neutro: sin MORA, sin "✓ Listo", sin total ("🆕 Nuevo") y "🆕 N nuevo(s) · sin cobro este mes". El mensaje multi-día recibe solo los días con `g.loans`.
  - Gráfica de evolución (`vEvolucion`): la barra del mes actual solo suma activos con `isCollectable`; en meses cerrados un préstamo cuenta solo si tiene un interés archivado que no sea el pago simbólico "Mes de apertura" (`esPagoApertura`).
- `socio.html` → lista "Tus préstamos": un préstamo que aún no se cobra (`!isCollectable`: empezó este mes) muestra "🆕 Nuevo" en azul (no "⏳ Pendiente") y debajo "Primer pago: Noviembre 2026" (mes siguiente a `mes_comienzo`). Igual para continuaciones, sin distinguirlas. No cuenta como pendiente en "Estado del mes", "Pendiente de girarte (meses anteriores)" ni "Mis giros".
- `socio.html` → tarjeta **"Movimiento del mes"** (`movimientoMesCard`, debajo de "Tu cartera"), con selector "Este mes" / "Mes anterior":
  - **Nuevos**: todo préstamo con `mes_comienzo` igual al mes, incluidas las continuaciones (`continuacion_de`), sea el original de otro socio o del mismo.
  - **Cerrados**: todo préstamo `status === 'paid'` cuyo mes de cierre es el mes, sea cual sea `cierre_tipo`. Mes de cierre = mes de `fecha_cierre`, o su mes de fin (`loanEndMk`) si no la tiene, igual que "Préstamos finalizados".
  - **Capital neto** = capital de nuevos − capital de cerrados (COP): "entró" o "salió". Si el préstamo sigue con el mismo socio, la continuación (nueva) y el original (cerrado) se compensan solos.
  - Al tocar cada número se ve la lista (día, periodo o fecha de cierre y capital), sin nombres de clientes.

---

## Socios inactivos

- **Definición** (`socioInactivo(name, st2, conAdelanto)` en `index.html`): sin préstamos activos (`st2.count === 0`), sin cuotas por girar (`cuotasPorGirar(name)` vacío, incluye las de préstamos ya cerrados) y sin adelantos por recuperar (`conAdelanto`, armado con `adelantosPorRecuperar()`). Si queda algo por girar o un adelanto pendiente, el socio sigue en la lista de activos.
- **`index.html` → Socios (`vSocios`)**: primero los activos (por capital); los inactivos van al final, ordenados por nombre, en la sección plegada "Socios inactivos (N)" (cerrada por defecto; `SOCIOS_INACTIVOS_OPEN` recuerda si se abrió entre `renderApp()`). Su tarjeta (`renderSocioCard(..., inactivo=true)`) dice "Sin préstamos activos · N finalizados" y "historial"; al abrirla muestra solo Préstamos finalizados, Giros (con comprobantes), Historial mensual y "Copiar link portal". Sin "Registrar giro", lista de activos, "Total a pagarte", imagen resumen ni botones de WhatsApp. No cuentan en totales ni alertas.
- **`socio.html` (`renderPortal`)**: si el socio no tiene préstamos activos (`soloHistorial`), arriba sale "No tienes préstamos activos actualmente. Aquí está tu historial." y debajo una línea "Invertiste $X en N préstamos · Intereses recibidos: $Y" (capital COP de los finalizados; intereses con la fórmula de "Rendimiento acumulado": `mesesCobrados(mes_comienzo, loanEndMk)` × `int_socio` × `montoCOP`). Se ocultan Tu cartera, Movimiento del mes, Estado del mes, Tu cartera con JiTel, el banner de vencimientos y Tus préstamos. Quedan Mis giros (con 📎 comprobantes) y Préstamos finalizados. El acceso no se restringe: el socio sigue entrando con su cuenta.

---

## Convención Julián

Cuando `socio === 'JULIAN'` e `int_socio === 0`, el préstamo es **directo del propietario** sin socio externo. En estos casos:
- `IS(l) === 0` — no hay pago a socio ni giros
- `IP(l) === IM(l)` — toda la ganancia es propia
- El botón "🏦 Pagué socio" se oculta y la cuota se considera girada (`cuotaGirada`)
- La lógica de notificación WhatsApp lo excluye de los mensajes a socios

**Regla**: siempre usar `isJulianLoan(l)` para esta detección. Nunca comparar `l.socio` o `l.int_socio` directamente en otro lugar.

---

## Funcionalidades implementadas

| Funcionalidad | Descripción técnica |
|---------------|---------------------|
| **Pago unificado** | Registrar pago auto-setea `recibido=true` cuando el periodo es el mes actual |
| **Conciliación in-place** | `conciliBtn()` (solo `recibido`) actualiza DOM y `ST.loans` sin `renderApp()` |
| **Giros** | Ver sección "Giros a socios" |
| **Sumador temporal** | Checkboxes en Cobros acumulan selección; barra flotante muestra total; multi-pago en un click |
| **Paginado imágenes** | `buildSocioImage()` y `buildClienteImage()` generan array de dataUrls (10 filas/página) |
| **Google Drive backup** | OAuth2 via GIS; renovación automática de token cuando quedan <30 min; reintento silencioso × 3; `setInterval` cada 45 min |
| **Mensaje multi-día** | `showMensajeMultidia()` en Cobros |
| **Gráfico histórico SVG** | `vEvolucion()` en Resumen |
| **Actualizar/Extender** | Plazo, mes de inicio (con "Primer cobro" en vivo), socio y tasas; los cambios quedan en `nota` |
| **Primer cobro** | En +Nuevo, texto en vivo "Primer cobro: día D de MES AÑO" (mes siguiente al inicio) |

---

## Firebase

**SDK**: v12.16.0 vía CDN, **capa compat** (`firebase-app-compat/auth-compat/firestore-compat.js`). La capa compat mantiene la API namespaced de v8 (`firebase.initializeApp/auth()/firestore()`, `.collection()`, `.doc()`). Config en `FIREBASE_CONFIG` (proyecto `jitel-capital`).

**Objeto global `FB`**: `{ready, app, auth, db, user, perfil}`. `isAdmin()` = `FB.perfil.rol==='admin'`.

**Arranque (boot)**: IIFE al final del script de `index.html`.
- Si el SDK no carga → `bootLegacyLocal()` (localStorage + cache local de giros, sin login — la app nunca queda inutilizable).
- Si carga → `fbInit()`, gatea con `onAuthStateChanged`: sin sesión → `showLoginScreen()`; admin → `bootData()` (`loadSociosMap` + `loadGiros` + `loadSTAsync` → `startApp`); socio/sin perfil → `showSocioPlaceholder()` (los socios usan `socio.html`).

**Lectura**: `loadSTAsync(cb)` → Firestore primero (`loadFromFirestore`), si vacío/falla usa `loadST()` (localStorage), si nada usa `mkInit()`.

**Escritura**: `saveST()` = `saveLocalOnly()` (siempre) + `fbSyncDebounced()` (solo admin, 1.5s → `fbPushAll()`: **solo los `prestamos/{id}` que cambiaron** + `config/global`, en una transacción con control de versión; luego `pushPortalClientes()`). Nunca bloquea ni lanza.

**Control de versión** (`config/global.version`):
- Al cargar desde la nube, `fbAdoptarNube(data)` guarda la versión leída en `FB_VERSION` (0 si el doc aún no tiene `version`). También guarda en `_fbBase` la huella de cada préstamo (`loanHuella`: JSON con claves ordenadas, sin `socio_email`, más el email que le toca), y en `_fbBaseCfg` la de `currMk/archives/snapshots`.
- `fbPushAll` arma los cambios: préstamos nuevos o con huella distinta, préstamos borrados en este dispositivo, o config distinta. Si no hay ninguno, **no escribe ni sube la versión**. Si hay, hace `runTransaction`: lee `config/global` y:
  - si la versión remota es **mayor** que `FB_VERSION`, o este dispositivo no cargó de la nube (`FB_VERSION === null`) y el doc existe → **no escribe nada**. `fbConflictoVersion()` muestra "⚠️ Hay cambios desde otro dispositivo, recarga…" y recarga los datos de la nube; el cambio local que no se guardó se pierde;
  - si coincide → escribe los préstamos cambiados (hasta 450 en la transacción) y `config/global` con `version + 1`. Actualiza `FB_VERSION` y la base. Si hay más de 450 cambios (p.ej. al importar un backup), el resto va en lotes después de la transacción.
- Un guardado a la vez por dispositivo (`_fbPushing` / `_fbPushAgain`), para que dos guardados seguidos del mismo dispositivo no choquen entre sí.
- Los borrados siguen yendo aparte por `fbDeleteDoc`, pero cuentan como cambio, así que suben la versión.
- La regla de `config/{docId}` exige que cada escritura de `config/global` suba `version` en exactamente 1. Así una pestaña con el código viejo, que escribe sin `version`, queda rechazada. Los borrados de préstamos van aparte con `fbDeleteDoc` (reintento + cola `_pendingDeletes`); los giros con `fbCreateGiro` (ver "Giros").

**Estructura Firestore**:
- `usuarios/{uid}` → `{ perfil: { nombre, rol:'admin'|'socio'|'cliente', socioNombre } }`
- `socios/{code}` → `{ email, nombre, createdAt }` — se crea con `jitelSocioAdd(code, email, NOMBRE)`; alimenta `SOCIOS_MAP` (nombre → email)
- `prestamos/{loanId}` → todos los campos de un loan + `socio_email`
- `giros/{giroId}` → giros a socios (solo creación)
- `config/global` → `{ currMk, archives, snapshots, updatedAt, version }`. `version` sube en 1 en cada guardado (ver "Control de versión")
- `comprobantes/{giroId}` / `comprobantes_idx/{giroId}` → comprobantes de giros (ver "Comprobantes de giros")
- `uso/{YYYY-MM}` → métricas de uso del admin: un campo numérico por acción (ver "Métricas de uso")
- `portal_clientes/dayam` → **copia pública** para `dayam.html`, construida por `buildPortalClienteDoc('DAYAM')` en cada sync. Solo campos no sensibles: `id, dia, capital (COP), pago_mensual (COP), mes_fin, año, mes_comienzo, status, continuacion_de, recibido, pagos:[{periodo, fecha}], cuotas_pendientes:[{periodo, recibido}]`. **Nunca** incluir socio, socio_email, tasas, moneda, tasa_cambio, nota ni cierre_tipo.
- `pagos/{loanId}/pagos/{pagoId}` → copia normalizada creada solo por la migración inicial (no se sincroniza en vivo)

**Reglas de Firestore vigentes** — copia en [`firestore.rules`](firestore.rules).

> Las reglas vigentes se publican a mano en Firebase Console. firestore.rules es una copia de referencia: cualquier cambio debe publicarse en la consola y actualizarse aquí en el mismo commit.

- **Admin** (`isAdmin()`): usuario con sesión cuyo `usuarios/{uid}.perfil.rol == 'admin'`.
- **Mi email** (`esMiEmail(x)`): usuario con sesión cuyo correo del token, **en minúsculas**, es igual a `x`. Por eso `socio_email` y `socios.email` deben guardarse en minúsculas (`SOCIOS_MAP` ya los pasa a minúsculas).

| Ruta | Lectura | Escritura |
|------|---------|-----------|
| `usuarios/{uid}` | el propio usuario (`auth.uid == uid`) o admin | solo admin (nadie puede escribir su propio perfil, para que no pueda darse `rol: 'admin'`) |
| `pagos/{loanId}/pagos/{pagoId}` | admin | admin |
| `config/{docId}` | admin | admin; en `config/global`, crear o actualizar solo con `version` = versión actual + 1 (0 si no tiene) |
| `socios/{code}` | admin, o el socio cuyo `email` es su correo | admin |
| `prestamos/{loanId}` | admin, o el socio cuyo `socio_email` es su correo | admin (crear, editar y borrar) |
| `giros/{giroId}` | admin, o el socio cuyo `socio_email` es su correo | **solo crear**, admin; `update` y `delete` prohibidos para todos |
| `comprobantes/{giroId}` | admin, o el socio cuyo `socio_email` es su correo | crear y actualizar, admin (`imagen` string ≤ 307200 caracteres); `delete` prohibido |
| `comprobantes_idx/{giroId}` | admin, o el socio cuyo `socio_email` es su correo | crear y actualizar, admin; `delete` prohibido |
| `uso/{mes}` | admin | admin |
| `portal_clientes/{clienteId}` | **cualquiera, sin sesión** | admin |

Cualquier cambio de código que lea o escriba Firestore debe respetar estas reglas. En particular: desde `socio.html` las consultas a `prestamos`, `giros`, `comprobantes_idx` y `socios` deben filtrar por el correo del socio (`where socio_email == …` / `where email == …`), porque una consulta que pueda devolver documentos ajenos se rechaza completa; nunca hacer `update`/`delete` en `giros`; y todo lo que se escriba en `portal_clientes` es público.

**Funciones de consola** (expuestas en `window`): `migrarAFirebase()`, `jitelBootstrapAdmin('Julián')`, `jitelSocioAdd(code, email, NOMBRE)`, `jitelLogout()`.
- `jitelBootstrapAdmin` **ya no funciona con las reglas vigentes**: escribía el propio `usuarios/{uid}` desde un usuario que aún no era admin, y eso ya no está permitido. El admin ya existe; un perfil nuevo se crea desde Firebase Console o con una sesión admin.
- `migrarAFirebase` escribe `config/global` sin `version`, así que la regla vigente la rechaza. Es una herramienta de la migración inicial, que ya se hizo.
- `jitelSocioAdd` guarda el email con `trim()` y `toLowerCase()`, porque las reglas comparan contra el correo de la sesión en minúsculas.

### Métricas de uso (`registrarUso(clave)`)
- Suma 1 al campo `clave` de `uso/{YYYY-MM}` (mes de la fecha del dispositivo, `usoMkHoy`) con `set({clave: FieldValue.increment(1)}, {merge:true})`. Solo si `FB.ready` e `isAdmin()`; claves `[a-z0-9_]`.
- Va **directo a Firestore, sin esperar**: no pasa por `saveST`, no toca `config/global` ni su `version`, y todo va en `try/catch` + `.catch` → nunca rompe ni bloquea la UI. Sin Firebase (`bootLegacyLocal`) no hace nada.
- Claves y dónde se registran: `tab_<id>` (navbar), `nuevo` (abrir +Nuevo), `nuevo_guardado` / `nuevo_guardado_continuacion`, `nuevo_cancelado` (el modal de +Nuevo se cerró o reemplazó sin guardar, vía `_alCerrarModal`), `simulador` (`showCalculadora`), `nuevo_mes` (abrir `showReset`) y `nuevo_mes_confirmado` (`doReset`), `backup_csv` (`exportBackupCSV`), `backup_drive` (`gdriveManualBackup`; el automático no cuenta), `backup_importar`, `whatsapp` (`openWhatsApp`), `imagen` (`showImageViewer`), `giro` / `giro_anulado` (`registrarGiro`), `cerrar_prestamo` y `traspaso` (cierre con `cierre_tipo='traspaso'`), `extender` (guardar Actualizar/Extender), `corregir_periodo` (`corregirPeriodoPago` exitoso).
- `_alCerrarModal`: callback de un solo uso que `closeModal` ejecuta (también cuando `showModal` reemplaza el modal abierto).
- Vista: 🗄️ Backup (`showBackupModal`) → tabla "📈 Uso de la app" (`usoTablaEl`) con los conteos del mes actual y del anterior; etiquetas en `USO_LBL` (las claves sin etiqueta se muestran tal cual).
- Para agregar una métrica nueva: llamar `registrarUso('clave')` y agregar su etiqueta en `USO_LBL`.

**App Check**: desactivado intencionalmente. La seguridad se gestiona con las reglas de Firestore por rol (`firestore.rules`).

---

## Reglas permanentes

1. **No romper la paginación de imágenes** — `buildSocioImage` y `buildClienteImage` ya funcionan correctamente con `MAX_ROWS=10` y el patrón `onDone(dataUrls[])`. No cambiar su firma ni lógica interna salvo indicación explícita.

2. **No tocar la lógica de pagos** — las funciones `IM`, `IS`, `IP`, `aplicarPagoALoan`, `toggleConcil`, `showPaymentConfirm`, el flujo de registro de pagos en Cobros — solo modificar si se indica explícitamente.

3. **`pagado_socio` solo lo pone un giro** — no agregar ningún otro camino que lo marque o desmarque. Los giros nunca se editan ni se borran: se anulan con `anula_a`.

4. **Mantener `isJulianLoan` consistente** — cualquier feature que filtre o trate diferente los préstamos sin socio externo debe usar esta función, no comparaciones directas.

5. **Evitar `renderApp()` en handlers** — llama a re-render completo y colapsa acordeones. Preferir actualización DOM in-place (ver patrón de `conciliBtn`).

6. **ES5 estricto** — sin `const`/`let`, sin arrow functions `=>`, sin template literals `` ` ``, sin spread `...`, sin destructuring. Todo con `var` y funciones nombradas o anónimas clásicas. No declarar `function` dentro de bloques `for`/`if` (usar `var f=function(){}`).

7. **Sin nombres de clientes** en nada que vea un socio (`socio.html`, imágenes y mensajes de socio), ni datos sensibles en `portal_clientes/dayam`.

8. **Un solo commit al final** — nunca commits intermedios durante una sesión de cambios múltiples. Mensaje de commit descriptivo en español o inglés técnico.

9. **`git push` al finalizar cada tarea** — confirmar push exitoso antes de reportar la tarea como completada.
