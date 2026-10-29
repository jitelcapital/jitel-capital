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
| **Cobros** | Vista del mes agrupada por día de cobro (acordeones). Pago rápido/por grupo, toggle "Cobré", botón "🏦 Pagué socio" (abre el modal de giro), sumador temporal, mensaje multi-día. Sección "📦 Pendientes del mes anterior" (cuotas arrastradas) y aviso "Cobrado sin girar hace más de 3 días". |
| **Resumen** | Dashboard ejecutivo: capital activo, ganancia mensual, mora, KPIs, gráfico histórico SVG (últimos 6 meses), preferencias, backup Google Drive, "Nuevo mes" (cierre mensual). |
| **Préstamos** | Lista completa (activos + cerrados) con búsqueda y filtros. Cada fila abre `showLoanDetail`: historial de pagos, registrar pago (con periodo), Actualizar/Extender (plazo, mes de inicio, socio, tasas) y cerrar préstamo. |
| **Socios** | Por socio: capital, interés mensual, préstamos, finalizados, sección **Giros** por mes, historial mensual (`ST.snapshots`), botón "🏦 Registrar giro", imagen paginada para WhatsApp. Aviso "Cobrado sin girar" arriba. |
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
  3. **Pagos de meses posteriores**: si algún préstamo activo tiene en `pagos` un pago con periodo > `mk` (`pagosPosterioresA`), se listan y se bloquea; el cierre los archivaría bajo `mk` y los quitaría del préstamo.
- Antes de confirmar muestra las cuotas sin cobrar o sin girar del mes (`loansPendientesParaCierre`), las **cobradas sin girar hace más de 3 días** por socio, y un **recordatorio para exportar el backup** (botón "💾 Exportar backup ahora").
- **No se puede deshacer** desde la app: la única vuelta atrás es restaurar el backup exportado justo antes.
- Hacerlo desde **un solo dispositivo** y recargar la app en los demás: una sesión abierta con datos viejos reescribe `prestamos` y `config/global` completos en su siguiente `saveST()`.
- Al confirmar (`doReset`), en este orden:
  1. `generarArrastrePendientes(mk)`: cada cuota sin resolver del mes pasa a `loan.cuotas_pendientes` con su `recibido`/`pagado_socio`.
  2. `ST.snapshots[mk] = buildSociosSnapshot()`.
  3. `ST.archives[mk]` = pagos de todos los préstamos.
  4. Préstamos activos: `pagos=[]`, `recibido=false`, `pagado_socio=false`. Los cerrados conservan sus pagos.
- Una cuota arrastrada está resuelta cuando tiene pago para su periodo **y** (`pagado_socio` o `isJulianLoan`). Las sin resolver se ven en Cobros → "Pendientes del mes anterior".

---

## Giros a socios (colección `giros`)

Cada pago a un socio se registra como un **giro**. Arranca en **octubre de 2026** (`GIROS_DESDE`); no se importaron giros anteriores.

- **Sin giro no hay "Pagué socio"**: `pagado_socio` (del mes y de `cuotas_pendientes`) solo se pone en `true` al guardar un giro (`registrarGiro`) y vuelve a `false` solo al anular el giro que la cubría. `toggleConcil`, `conciliBtn` y `conciliBtnCuota` ignoran el campo `pagado_socio`; el botón es `giroBtn(loan, periodo)`.
- **Documento**: `{id, socio, socio_email, fecha, monto (COP), modalidad, modalidad_detalle, referencia, tipo, cuotas:[{loanId, periodo, monto}], nota, creado_en, anula_a?}`.
  - `modalidad`: `'transferencia' | 'efectivo' | 'abono_tarjeta' | 'otro'`.
  - `tipo`: `'intereses' | 'devolucion_capital' | 'aporte_capital'`. En capital, `cuotas` lleva los préstamos asociados con `periodo: null`.
  - `anula_a` solo existe en anulaciones.
- **Obligatorios**: referencia en transferencia y abono a tarjeta; a quién se entregó o nota en efectivo; `modalidad_detalle` en otro.
- **Nunca se editan ni se borran**. Para corregir: "Anular giro" pide motivo y crea un giro nuevo con `anula_a = <id>`. Un giro está anulado si existe otro con `anula_a` igual a su id (`giroVigente`). Las cuotas del anulado vuelven a pendientes de girar (si el mes ya estaba cerrado, se agrega una cuota arrastrada).
- **Estado girado de una cuota** (`cuotaGirada(l, periodo)`): la cuota arrastrada de ese periodo si existe; si el periodo no está archivado, `l.pagado_socio`; si está archivado y sin arrastre, se considera resuelta.
- **Cuotas por girar** (`cuotasPorGirar(socio)`): con pago del cliente registrado y sin girar (mes en curso + arrastradas). El aviso (`alertasSinGirar`) muestra las de más de 3 días.
- **Escritura**: fuera de `fbPushAll`. `fbCreateGiro` hace `set` con 2 reintentos (antes de reintentar comprueba si el doc ya existe, porque un segundo `set` sería un update y las reglas lo rechazan). Si falla, aviso rojo visible y cola `_pendingGiros`, que `flushPendingGiros` reintenta en cada `fbPushAll`.
- **Carga**: `loadGiros` al iniciar sesión de admin (Firestore + cache local `jitel_giros_v1`). En modo sin Firebase solo la cache local.
- **Vistas**: Socios → sección "Giros" por mes (generado = interés de las cuotas del periodo, girado = cuotas de ese periodo en giros vigentes, diferencia; anulados tachados con motivo). `socio.html` → "Mis giros" con el mismo cálculo. Después de guardar se ofrece el mensaje de WhatsApp (`waGiroMsg`, sin nombres de clientes).
- **Backup CSV**: fila especial `__GIROS__` (JSON), como `__ARCHIVES__`. Al importar solo se agregan los que falten; no se suben a la nube.
- El backup CSV (`exportBackupCSV` y `gdriveBuildCSV`) también incluye las fotos mensuales en la fila `__SNAPSHOTS__` (JSON de `ST.snapshots`). Al importar reemplaza `ST.snapshots`; un backup viejo sin esa fila conserva las fotos actuales.
- Compatibilidad: cuotas marcadas antes de los giros siguen marcadas (se ven "✓ Girado" sin giro asociado).

---

## Traspasos y cierre de préstamos

- "✓ Cerrar préstamo" (`showCloseLoanModal`) pone `status='paid'`, `cierre_tipo` (`'finalizado'` o `'traspaso'`) y `fecha_cierre`, y ofrece los mensajes de cierre (cliente y/o socio).
- Un préstamo nuevo puede marcarse como **continuación** de otro (`continuacion_de`) para traspasos de cartera entre socios.
- En `buildClienteImage` una continuación se ve como fila normal, sin etiquetas. En `dayam.html` sí hereda el estado del original.
- En la pestaña Socios (admin) los finalizados muestran "✅ Finalizado" o "✅ Liquidado en su totalidad" (traspaso), sin nombrar al cliente ni al socio entrante. El admin sigue viendo el traspaso con todo su detalle.
- **`socio.html` nunca muestra traspasos**: ningún texto visible con "traspaso", "continuación" ni "liquidado", y todos los cerrados se ven como "✅ Finalizado".
- `socio.html` → tarjeta **"Movimiento del mes"** (`movimientoMesCard`, debajo de "Tu cartera"), con selector "Este mes" / "Mes anterior":
  - **Nuevos**: préstamos con `mes_comienzo` igual al mes. Una continuación (`continuacion_de`) cuenta como nuevo normal si el original **no** es de este socio (no está entre sus préstamos). Si el original **sí** es suyo (renovación propia), no cuenta.
  - **Cerrados**: `status === 'paid'` con `fecha_cierre` dentro del mes, cualquier `cierre_tipo` (un préstamo suyo cerrado por traspaso cuenta como cerrado normal). Excepción: el original de una renovación propia no cuenta, para que la renovación no aparezca como capital que sale.
  - **Capital neto** = capital de nuevos − capital de cerrados (COP): "entró" o "salió".
  - Al tocar cada número se ve la lista (día, periodo o fecha de cierre y capital), sin nombres de clientes.

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

**Escritura**: `saveST()` = `saveLocalOnly()` (siempre) + `fbSyncDebounced()` (solo admin, 1.5s → `fbPushAll()`: docs `prestamos/{id}` + `config/global` en lotes, luego `pushPortalClientes()`). Nunca bloquea ni lanza. Los borrados de préstamos van aparte con `fbDeleteDoc` (reintento + cola `_pendingDeletes`); los giros con `fbCreateGiro` (ver "Giros").

**Estructura Firestore**:
- `usuarios/{uid}` → `{ perfil: { nombre, rol:'admin'|'socio'|'cliente', socioNombre } }`
- `socios/{code}` → `{ email, nombre, createdAt }` — se crea con `jitelSocioAdd(code, email, NOMBRE)`; alimenta `SOCIOS_MAP` (nombre → email)
- `prestamos/{loanId}` → todos los campos de un loan + `socio_email`
- `giros/{giroId}` → giros a socios (solo creación)
- `config/global` → `{ currMk, archives, snapshots, updatedAt }`
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
| `config/{docId}` | admin | admin |
| `socios/{code}` | admin, o el socio cuyo `email` es su correo | admin |
| `prestamos/{loanId}` | admin, o el socio cuyo `socio_email` es su correo | admin (crear, editar y borrar) |
| `giros/{giroId}` | admin, o el socio cuyo `socio_email` es su correo | **solo crear**, admin; `update` y `delete` prohibidos para todos |
| `portal_clientes/{clienteId}` | **cualquiera, sin sesión** | admin |

Cualquier cambio de código que lea o escriba Firestore debe respetar estas reglas. En particular: desde `socio.html` las consultas a `prestamos`, `giros` y `socios` deben filtrar por el correo del socio (`where socio_email == …` / `where email == …`), porque una consulta que pueda devolver documentos ajenos se rechaza completa; nunca hacer `update`/`delete` en `giros`; y todo lo que se escriba en `portal_clientes` es público.

**Funciones de consola** (expuestas en `window`): `migrarAFirebase()`, `jitelBootstrapAdmin('Julián')`, `jitelSocioAdd(code, email, NOMBRE)`, `jitelLogout()`.
- `jitelBootstrapAdmin` **ya no funciona con las reglas vigentes**: escribía el propio `usuarios/{uid}` desde un usuario que aún no era admin, y eso ya no está permitido. El admin ya existe; un perfil nuevo se crea desde Firebase Console o con una sesión admin.
- `jitelSocioAdd` guarda el email con `trim()` y `toLowerCase()`, porque las reglas comparan contra el correo de la sesión en minúsculas.

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
