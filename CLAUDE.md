# JiTel Capital — Contexto para Claude

## Descripción del negocio

**JiTel Capital** es una app de gestión de cartera de préstamos informales en Colombia. Permite registrar préstamos a clientes, llevar el cobro mensual de intereses, hacer seguimiento de pagos a socios inversores, generar imágenes para WhatsApp y hacer backups en Google Drive.

Los préstamos generan interés mensual cobrado al cliente (`int_socio + int_propio`). El socio inversor recibe `int_socio × monto` y el propietario retiene `int_propio × monto` como ganancia neta.

---

## Arquitectura

- **Un solo archivo**: `index.html` (~3.300 líneas), sin framework ni build system
- **JS embebido** en `<script>` al final del HTML; estilo ES5 estricto (sin `const`/`let`, sin arrow functions, sin template literals)
- **Persistencia**: en migración a **Firebase** (Auth + Firestore). `localStorage` (`jitel_v1`) se mantiene como red de seguridad / fallback. Ver sección "Firebase" abajo.
- **Despliegue**: GitHub Pages (rama `main`) — `https://julianjimenez91.github.io/jitel-capital/`
- **Sin librerías externas**: SVG nativo, Canvas nativo, Google Identity Services vía CDN para OAuth2

---

## Las 5 pestañas

| Pestaña | Función |
|---------|---------|
| **Cobros** | Vista diaria de cobros agrupados por día. Acordeones colapsables por día. Botón Pagar por grupo, conciliación (✓ Recibí / 🏦 Pagué socio), sumador temporal de selección múltiple, notificaciones WhatsApp, mensaje multi-día. |
| **Resumen** | Dashboard ejecutivo: capital activo, ganancia mensual, mora, KPIs financieros, gráfico histórico SVG de evolución (Ganancia/Capital/Préstamos, últimos 6 meses), preferencias de app, backup Google Drive, reinicio mensual. |
| **Préstamos** | Lista completa de préstamos (activos + pagados) con búsqueda, filtros por estado/socio/cliente. Cada fila abre modal de detalle con historial de pagos, extensión de plazo y cambio de socio. |
| **Socios** | Resumen por socio inversor: capital aportado, interés mensual, detalle de préstamos. Exporta imagen paginada (10 filas/página) para compartir por WhatsApp. |
| **Clientes** | Resumen por cliente/responsable: préstamos activos, cuota total mensual. Exporta imagen paginada (10 filas/página) con estado de cuenta. |

---

## Funciones clave

### Cálculo de intereses
```js
function IM(l)  // Interés mensual total cobrado al cliente = monto × (int_socio + int_propio)
function IS(l)  // Interés del socio inversor              = monto × int_socio
function IP(l)  // Interés propio (ganancia neta)          = monto × int_propio
```

### Detección de préstamos directos
```js
function isJulianLoan(l)
// true si socio==='JULIAN' && int_socio===0
// Préstamo directo del propietario sin socio externo
// SIEMPRE usar esta función — nunca comparar socio o int_socio por separado
```

### Estado de pago del mes actual
```js
function paidMth(l)
// true si el arreglo l.pagos contiene al menos un pago con fecha en el mes/año actuales
```

### Persistencia
```js
function saveST()    // Serializa ST a localStorage — llamar tras toda mutación de ST
function renderApp() // Re-renderiza la vista activa completa — COLAPSA acordeones abiertos
                     // Preferir actualizaciones DOM in-place cuando sea posible
```

---

## Modelo de datos `ST`

```js
var ST = {
  loans: [/* array de objetos préstamo */],
  archives: {
    "2026-04": { loanId: [pagos...] },  // pagos del mes archivado por préstamo
    "2026-05": { ... }
  }
};
```

### Campos de cada préstamo (`ST.loans[i]`)

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | string | UUID generado con `GID()` |
| `año` | number | Año de vencimiento |
| `dia` | number | Día de cobro mensual (1–31) |
| `mes_fin` | string | Mes de vencimiento en español mayúsculas (e.g. `'JULIO'`) |
| `mes_comienzo` | string\|null | Mes de inicio `'YYYY-MM'` — puede ser null |
| `monto` | number | Capital prestado en COP |
| `socio` | string | Nombre del socio inversor (e.g. `'VICTOR'`, `'JULIAN'`) |
| `int_socio` | number | Tasa mensual del socio (e.g. `0.035` = 3.5%) |
| `int_propio` | number | Tasa mensual propia (e.g. `0.015` = 1.5%) |
| `responsable` | string | Nombre del cliente/deudor |
| `status` | string | `'active'` o `'paid'` |
| `pagos` | array | Historial de pagos del mes actual: `[{fecha, monto, tipo, nota}]` |
| `recibido` | boolean | Conciliación: cliente pagó este mes |
| `pagado_socio` | boolean | Conciliación: socio fue pagado este mes |
| `nota` | string | Nota libre del préstamo |
| `created_at` | string | ISO timestamp de creación |

### Tipos de pago (`pagos[i].tipo`)
- `'interes'` — pago mensual de intereses (el más común)
- `'abono'` — abono a capital
- `'cancelacion'` — cancelación total del préstamo

### Archivos mensuales (`ST.archives`)
- Clave: `'YYYY-MM'` (e.g. `'2026-05'`)
- Valor: objeto `{ loanId: pagos[] }` — snapshot de los pagos de ese mes para todos los préstamos
- Se crea al ejecutar "Reinicio mensual" — no modificar su estructura

---

## Convención Julián

Cuando `socio === 'JULIAN'` e `int_socio === 0`, el préstamo es **directo del propietario** sin socio externo. En estos casos:
- `IS(l) === 0` — no hay pago a socio
- `IP(l) === IM(l)` — toda la ganancia es propia
- El botón "🏦 Pagué socio" se oculta en la UI (`isJulianLoan` guard en `conciliBtn`)
- La lógica de notificación WhatsApp lo excluye de los mensajes a socios

**Regla**: siempre usar `isJulianLoan(l)` para esta detección. Nunca comparar `l.socio` o `l.int_socio` directamente en otro lugar.

---

## Funcionalidades implementadas

| Funcionalidad | Descripción técnica |
|---------------|---------------------|
| **Pago unificado** | Registrar pago auto-setea `recibido=true`; botón "💰 Recibí" eliminado |
| **Conciliación in-place** | `conciliBtn()` actualiza DOM y `ST.loans` sin llamar `renderApp()` — el acordeón no colapsa |
| **Sumador temporal** | Checkboxes en Cobros acumulan selección; barra flotante muestra total; multi-pago en un click |
| **Paginado imágenes** | `buildSocioImage()` y `buildClienteImage()` generan array de dataUrls (10 filas/página) con navegador de páginas en el modal |
| **Google Drive backup** | OAuth2 via GIS; renovación automática de token cuando quedan <30 min; reintento silencioso × 3 antes de toast de error; `setInterval` cada 45 min |
| **Mensaje multi-día** | `showMensajeMultidia()` en Cobros: selector de socio + rango de días + mensaje consolidado WhatsApp |
| **Gráfico histórico SVG** | `vEvolucion()` en Resumen: tabs Ganancia/Capital/Préstamos, últimos 6 meses, barras doradas (mes actual) vs azules (anteriores), indicadores de variación y mejor mes |
| **Cambio de socio** | Modal de extensión permite cambiar el socio de un préstamo con recálculo de tasas y confirmación |
| **Ordenación automática** | Préstamos de cada cliente ordenados por `dia` ascendente |
| **Token Drive** | Renovación proactiva: `setInterval` 45 min + guard <30 min restantes |

---

## Firebase (migración en curso — Día 1 hecho)

**SDK**: compat v9.0.0 vía CDN (`firebase-app/auth/firestore-compat.js`) cargado en `<head>`. Config en `FIREBASE_CONFIG` (proyecto `jitel-capital`).

**Objeto global `FB`**: `{ready, app, auth, db, user, perfil}`. `isAdmin()` = `FB.perfil.rol==='admin'`.

**Arranque (boot)**: IIFE al final del script.
- Si el SDK no carga → `bootLegacyLocal()` (localStorage, sin login — la app nunca queda inutilizable).
- Si carga → `fbInit()`, gatea con `onAuthStateChanged`: sin sesión → `showLoginScreen()`; admin → `bootData()` (app completa); socio/sin perfil → `showSocioPlaceholder()` (vista de socio es Día 2).
- `startApp()` corre una sola vez (`APP_STARTED`): navbar, botón logout (🚪 en topbar), timers de Drive, `renderApp()`.

**Lectura (Fase 5)**: `loadSTAsync(cb)` → Firestore primero (`loadFromFirestore`), si vacío/falla usa `loadST()` (localStorage), si nada usa `mkInit()`. `loadST` queda intacto como fallback síncrono.

**Escritura**: `saveST()` = `saveLocalOnly()` (siempre) + `fbSyncDebounced()` (best-effort, solo admin, 1.5s debounce → `fbPushAll()` escribe docs `prestamos/{id}` + `config/global` en lotes). Nunca bloquea ni lanza. `saveLocalOnly()` para sembrado sin tocar Firebase.

**Estructura Firestore**:
- `usuarios/{uid}` → `{ perfil: { nombre, rol:'admin'|'socio'|'cliente', socioNombre } }`
- `prestamos/{loanId}` → todos los campos de un loan (incluye array `pagos`)
- `pagos/{loanId}/pagos/{pagoId}` → copia normalizada (se llena en migración; para vistas de socio en Día 2)
- `config/global` → `{ currMk, archives, ... }`

**Funciones de consola** (expuestas en `window`):
- `migrarAFirebase()` — migra localStorage→Firestore con progreso; solo admin.
- `jitelBootstrapAdmin('Julián')` — crea el perfil admin del usuario actual (arranque inicial, uso único).
- `jitelLogout()` — cierra sesión.

**Setup en Firebase Console** (no en código): habilitar Auth correo/contraseña; agregar dominio `julianjimenez91.github.io` a Authorized domains; crear Firestore; crear usuarios; configurar reglas de seguridad por rol (Día 2).

**Pendiente Día 2**: vista/portal de socio dentro de la app, reglas de Firestore por rol, sync de subcolección `pagos` en escrituras en vivo.

## Reglas permanentes

1. **No romper la paginación de imágenes** — `buildSocioImage` y `buildClienteImage` ya funcionan correctamente con `MAX_ROWS=10` y el patrón `onDone(dataUrls[])`. No cambiar su firma ni lógica interna salvo indicación explícita.

2. **No tocar la lógica de pagos** — las funciones `IM`, `IS`, `IP`, `toggleConcil`, `showPaymentConfirm`, el flujo de registro de pagos en Cobros — solo modificar si se indica explícitamente.

3. **Mantener `isJulianLoan` consistente** — cualquier feature que filtre o trate diferente los préstamos sin socio externo debe usar esta función, no comparaciones directas.

4. **Evitar `renderApp()` en handlers** — llama a re-render completo y colapsa acordeones. Preferir actualización DOM in-place (ver patrón de `conciliBtn`).

5. **ES5 estricto** — sin `const`/`let`, sin arrow functions `=>`, sin template literals `` ` ``, sin spread `...`, sin destructuring. Todo con `var` y funciones nombradas o anónimas clásicas.

6. **Un solo commit al final** — nunca commits intermedios durante una sesión de cambios múltiples. Mensaje de commit descriptivo en español o inglés técnico.

7. **`git push` al finalizar cada tarea** — confirmar push exitoso antes de reportar la tarea como completada.
