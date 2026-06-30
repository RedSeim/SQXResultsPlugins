# Implementation Plan — Integración DatabankMonkeyTest ↔ Custom Analysis MonkeyTest

> Plan de implementación **solo para el ResultsPlugin** (`DatabankMonkeyTest/index.html`).
> El Custom Analysis (Java) ya está resuelto y no es objeto de este plan.
> Contrato de referencia: [MonkeyTestCustomAnalysisInteraction.md](MonkeyTestCustomAnalysisInteraction.md).

---

## 0. Objetivos y principio rector

El plugin debe ganar 4 capacidades **sin romper nada de lo actual**:

1. **Auto-carga (lectura):** al seleccionar una estrategia (`STRATEGY_DATA`), buscar en `user/exports/MonkeyTest/` un CSV+meta cuyo rango temporal coincida con la estrategia, y si existe, **pintar los gráficos sin ejecutar "Run Monkey Test"**.
2. **Persistencia (escritura):** al recalcular con "Run Monkey Test", **escribir/actualizar** el par CSV+meta en esa carpeta, sin dejar duplicados (sobrescribir el archivo existente).
3. **Refresco:** un recálculo manual siempre prevalece sobre lo auto-cargado en la UI.
4. **Botón "Delete Cached Data":** bajo "Run Monkey Test", menos llamativo, que borra los CSV/meta de Monkey Test. El alcance de qué estrategias se borran depende de lo seleccionado en el desplegable **"Perform Test On"** (active/bulk/both).

**Principio rector — aislamiento:** el motor de cálculo actual (Web Worker, `runMonkeySimulation`, `runActiveStrategyTest`, `runBulkTest`, gráficos `paintGaussianHistogram`/`paintEquityCurveComparison`) **no se toca en su lógica**. La integración se añade como una capa nueva (carga/guardado) que produce el **mismo objeto `result`** que ya consume `drawActiveCharts`. Esto está explícitamente diseñado así en el contrato (§8: «se usan sin modificación»).

---

## 1. Hallazgos del código actual (base sobre la que construimos)

### 1.1 Java Custom Analysis — cómo escribe en disco

Revisado `MonkeyTest.java` completo. Hallazgos definitivos:

| Hallazgo | Detalle | Implicación para el plugin |
|---|---|---|
| **Escribe directamente con `java.io.FileWriter`** | `new FileWriter("user/exports/MonkeyTest/" + name + "_monkey_simulation_data.csv")` (línea 183). Es I/O nativo Java dentro del proceso SQX. | **No existe ninguna API PostMessage de SQX para escritura de ficheros.** El Java no usa ningún mecanismo del iframe. |
| **Siempre sobrescribe el archivo base** | El path es siempre `[name]_monkey_simulation_data.csv` sin sufijos `_2`, `_3`. | Los "sufijos de copia" del contrato §2 eran especificación teórica. En práctica Java solo crea un archivo por estrategia. La política del plugin debe ser igual: **sobrescribir el archivo base canónico**. El bucle de descubrimiento puede simplificarse a base + máx 1–2 sufijos de seguridad. |
| **`tMin`/`tMax` del meta.json son epoch ms crudos UTC** | `tMin = o.OpenTime`, `tMax = o.CloseTime` directamente, sin conversión de zona. | El matching temporal compara epoch ms con epoch ms: matching directo sin conversión. |
| **`fmtDate` en Java usa explícitamente UTC** | `sdf.setTimeZone(TimeZone.getTimeZone("UTC"))` (línea 611). | Las fechas en el CSV son UTC. `parseDate` del plugin las parsea igualmente como epoch ms — sin desfase. |
| **El meta.json lo escribe el Java con `realProfit`** | `realProfit` es suma de `orders.get(i).PL` en el período. | El plugin puede mostrar este valor del meta como referencia sin recalcular. |

### 1.2 ResultsPlugin — puntos clave del código

| Elemento | Ubicación | Nota para el plan |
|---|---|---|
| `handleMessage` | [index.html:1780](index.html#L1780) | `STRATEGY_DATA` (1799) + `ORDERS_RESPONSE` (1814). Aquí engancha la auto-carga. |
| `ORDERS_RESPONSE` → `activeStrategy.trades` | [index.html:1814-1860](index.html#L1814-L1860) | Normaliza órdenes. **Punto donde disparar la verificación temporal.** |
| `activeResult` (ref) | [index.html:1133](index.html#L1133) | Alimenta todos los charts activos. La auto-carga rellena este mismo ref. |
| `COMPLETE_SINGLE` → `drawActiveCharts` | [index.html:2215-2227](index.html#L2215-L2227) | Patrón a replicar en la auto-carga. |
| `exportMonkeyData(res)` | [index.html:2348-2683](index.html#L2348-L2683) | Genera el CSV completo y dispara descarga (`link.click()` línea 2681). **Reutilizar para construir el string CSV**; el destino cambia. |
| `runMonkeyTest` | [index.html:2685](index.html#L2685) | Punto de enganche para persistencia post-cálculo. |
| `settings.testTarget` | [index.html:1122](index.html#L1122) | `'active'` / `'bulk'` / `'both'`. Controla qué afecta Delete Cached Data. |
| Botón Run/Cancel | [index.html:764-772](index.html#L764-L772) | Bajo este bloque va el nuevo "Delete Cached Data". |
| `bulkStrategies` (ref) | Ver contexto: `bulkStrategies.value` usado en `runBulkTest` | Lista de estrategias bulk. Delete bulk itera sobre sus nombres. |

### 1.3 Ventaja clave de la auto-carga vs. cálculo manual

El cálculo manual exige `priceData.value.length > 0` ([index.html:2304](index.html#L2304), [:2696](index.html#L2696)). La auto-carga **no necesita precios**: lee del CSV exportado + `GET_ORDERS`. Funciona con solo seleccionar la estrategia en el databank. Es la principal ventaja a comunicar en la UI.

---

## 2. Resultados de los feature-tests (COMPLETADOS)

### Contexto descubierto
- SQX sirve el plugin en `http://localhost:8080/plugins/DatabankMonkeyTest/index.html`
- El servidor expone **solo** la carpeta del plugin (`/plugins/DatabankMonkeyTest/**`) y sus subcarpetas
- `user/exports/MonkeyTest/` no está expuesto bajo ninguna ruta HTTP
- HTTP PUT y DELETE devuelven 404 — el servidor no soporta escritura ni borrado remotos

### Solución adoptada: carpeta `cache/` dentro del plugin ✅ IMPLEMENTADO

Se usa una **carpeta real** `cache/` dentro del plugin como ubicación canónica de todos los datos de Monkey Test:

```
user/extend/ResultsPlugins/DatabankMonkeyTest/cache/
  [StrategyName]_monkey_simulation_data.csv
  [StrategyName]_monkey_simulation_data.meta.json
```

**El Java (`MonkeyTest.java`) ya ha sido actualizado** para escribir en esta carpeta (líneas 181-182 y 374). El servidor SQX la sirve en `http://localhost:8080/plugins/DatabankMonkeyTest/cache/[archivo]`, accesible con `fetch('cache/[archivo]')` desde el plugin.

**Ventaja de portabilidad:** al compartir el plugin con otro usuario, la carpeta `cache/` viaja con él. No requiere ninguna configuración adicional ni junctions.

`BASE_EXPORT` en el plugin: `'cache/'`.

### Capacidades resultantes

| Capacidad | Estado | Mecanismo |
|---|---|---|
| **Lectura (auto-carga)** | ✅ VIABLE | `fetch('cache/[nombre].csv')` via junction |
| **Escritura desde plugin** | ❌ NO VIABLE | HTTP PUT no soportado por SQX |
| **Borrado desde plugin** | ❌ NO VIABLE | HTTP DELETE no soportado por SQX |

### Consecuencias para las fases
- **Fase 2 (auto-carga):** completamente viable.
- **Fase 3 (persistencia):** modo degradado — al recalcular con "Run Monkey Test", se descargan CSV + meta.json y se muestra instrucción al usuario para moverlos a `user/exports/MonkeyTest/`.
- **Fase 4 (Delete Cached Data):** no viable via HTTP. El botón se muestra deshabilitado con tooltip explicativo.

---

## 3. Arquitectura de la nueva capa

Toda la lógica nueva vive en un bloque cohesionado dentro del `setup()`, claramente separado del motor de cálculo. Módulos:

```
[MonkeyCache layer]
 ├─ paths/fetch         : BASE_EXPORT, fetchMetaAndCsv, discoverCandidates
 ├─ temporal match      : computeStrategyRange(x3), matchesMeta, matchesFallback, selectBestCandidate
 ├─ reconstruction      : parseCsvToMonkeyData, buildRealEquity, buildKpis → result
 ├─ persistence (write) : buildCsvString (refactor de exportMonkeyData), buildPluginMeta, writeCacheFiles
 ├─ delete              : deleteCachedData (respeta settings.testTarget)
 └─ state/UI            : cacheStatus, autoLoadedResult flags, badges, mensajes i18n
```

### 3.1 Estado nuevo (refs en `setup`)
```javascript
const cacheStatus    = ref('idle'); // 'idle'|'loading'|'loaded'|'approx'|'none'|'mismatch'|'error'
const cacheInfo      = ref(null);   // { period, generatedAtUtc, source, approximate }
const strategyRanges = ref(null);   // { full, is, oos } — calculado en ORDERS_RESPONSE
const lastOrdersRaw  = ref(null);   // órdenes crudas (sin normalizar) para buildRealEquity + meta
const cacheError     = ref('');
```

---

## 4. Fase 1 — Refactor habilitante (sin cambio de comportamiento)

Objetivo: preparar el terreno sin alterar la salida actual. Mergeable solo.

### 4.1 Extraer `buildCsvString(res, settings, priceData)` de `exportMonkeyData`
Mover toda la generación de filas ([index.html:2351-2672](index.html#L2351-L2672)) a una función pura que **devuelve el string CSV**. `exportMonkeyData` pasa a:
```javascript
const csv = buildCsvString(res, settings, priceData.value);
triggerDownload(csv, `${res.name}_monkey_simulation_data.csv`);
```
Verificar que la descarga sigue idéntica (mismo separador `;`, `\r\n`, formato de celdas con comillas dobles).

### 4.2 Añadir claves i18n
En `LOCALES` inline ([index.html:980](index.html#L980)) y en `locales/en.json` + `locales/es.json`:

| Clave | EN | ES |
|---|---|---|
| `loadingSavedData` | "Loading saved Monkey Test data…" | "Cargando datos guardados del Monkey Test…" |
| `dataLoadedFromExport` | "Loaded from cache (period: {period} · generated: {date} · source: {source})" | "Cargado desde caché (periodo: {period} · generado: {date} · origen: {source})" |
| `dataLoadedApprox` | "Loaded from cache (approximate — no fingerprint)" | "Cargado desde caché (aproximado — sin fingerprint)" |
| `noMatchingData` | "No cached Monkey Test data matches this strategy's period. Run the Custom Analysis or click 'Run Monkey Test'." | "No hay datos en caché que coincidan con el periodo de esta estrategia. Ejecuta el Custom Analysis o pulsa 'Ejecutar Monkey Test'." |
| `temporalMismatch` | "Cached data does not match the current strategy period (strategy may have been re-backtested)." | "Los datos en caché no coinciden con el periodo actual (puede que la estrategia haya sido re-backtesteada)." |
| `fetchError` | "Error loading cached data: {error}" | "Error al cargar datos en caché: {error}" |
| `sourceBadgeCA` | "Custom Analysis" | "Custom Analysis" |
| `sourceBadgePlugin` | "Plugin calculation" | "Cálculo del plugin" |
| `deleteCachedData` | "Delete Cached Data" | "Borrar datos en caché" |
| `deleteConfirm` | "Delete cached Monkey Test data for {scope}? This cannot be undone." | "¿Borrar datos en caché del Monkey Test para {scope}? Esta acción no puede deshacerse." |
| `cacheDeleted` | "Cached data deleted." | "Datos en caché borrados." |
| `cacheSaved` | "Results saved to cache." | "Resultados guardados en caché." |
| `cacheSaveFailed` | "Could not save to cache: {error}" | "No se pudo guardar en caché: {error}" |
| `cacheWriteManual` | "Move the downloaded files to user/exports/MonkeyTest/ to enable auto-loading." | "Mueve los archivos descargados a user/exports/MonkeyTest/ para habilitar la carga automática." |
| `cacheDeleteUnavailable` | "File deletion is not available in this SQX version." | "El borrado de archivos no está disponible en esta versión de SQX." |

**Criterio de aceptación Fase 1:** cálculo manual y descarga CSV producen exactamente el mismo resultado que antes.

---

## 5. Fase 2 — Auto-carga / LECTURA (depende del Test A)

**Alcance:** solo la pestaña **Active Strategy** (`settings.testTarget === 'active'` o `'both'`). La pestaña Bulk usa archivos subidos manualmente — auto-load bulk queda fuera de esta iteración.

### 5.1 Disparo (en `ORDERS_RESPONSE`)
Tras poblar `activeStrategy.value.trades` ([index.html:1859](index.html#L1859)):
```javascript
// Guardar órdenes crudas para buildRealEquity y meta
lastOrdersRaw.value = data.orders;
// Calcular rangos temporales (epoch ms, matching directo con meta.json)
strategyRanges.value = {
  full: computeStrategyRange(data.orders, 'FULL'),
  is:   computeStrategyRange(data.orders, 'IS'),
  oos:  computeStrategyRange(data.orders, 'OOS')
};
// Disparar auto-carga (async, no bloquea)
cacheStatus.value = 'loading';
attemptAutoLoad(activeStrategy.value.name, strategyRanges.value);
```

`computeStrategyRange` usa `parseDate` existente en `o.OpenTime`/`o.CloseTime` (que ya devuelve epoch ms) y excluye tipos balance (Type 9/10/11) exactamente como describe el contrato §7.1.

### 5.2 Descubrimiento de candidatos
Implementar `discoverCandidates(strategyName)` (contrato §6) sondeando sufijos `''`, `_2`…`_5` (5 es suficiente; el Java nunca crea `_2+`, pero el plugin podría en el futuro). Primer 404 → cortar el bucle.

```javascript
const BASE_EXPORT = 'cache/';
```

### 5.3 Verificación temporal
- `TOL_MS = 3_600_000` (1h) — margen para redondeos de milisegundos. Todos los valores son epoch ms UTC, sin desfase de zona horaria: el matching es directo.
- `matchesMeta(meta, ranges)` — compara `meta.tradeFromMs`/`tradeToMs` con el rango del periodo correspondiente.
- `matchesFallback(csvText, ranges)` — para CSVs sin meta: deriva rango de la columna `"Open time"` usando `parseDate` existente (las fechas en el CSV son UTC, y `parseDate` sobre epoch ms es pass-through).
- `selectBestCandidate(candidates, ranges)` — prefiere candidato con `generatedAtUtc` más reciente.

### 5.4 Reconstrucción del objeto `result`
- `parseCsvToMonkeyData(csv)` → `{ monkeyProfits, monkeyEquities }` con `try/catch` → candidato inválido si falla.
- `buildRealEquity(lastOrdersRaw.value, period)` → `{ realEquity, realReturn }`.
- `buildKpis(monkeyProfits, realReturn, meta)` → KPIs.
- Ensamblar `result` con `fromExport: true`, `name = strategyName`. **No setear `result.trades`** (el CSV contiene trades de monos, no reales; dejaría huérfano `exportMonkeyData` que requiere trades reales + priceData).

### 5.5 Pintado
```javascript
activeResult.value = result;
cacheStatus.value = result.approximate ? 'approx' : 'loaded';
cacheInfo.value = { period: result.period, generatedAtUtc: result.generatedAtUtc, source: result.source };
nextTick(() => drawActiveCharts());
```

### 5.6 UI de estado
Badge sobre el panel activo (junto a `kpi-row`, visible incluso si no hay `activeResult`):

| `cacheStatus` | Estilo | Texto |
|---|---|---|
| `'loading'` | gris, spinner | `loadingSavedData` |
| `'loaded'` | verde | `dataLoadedFromExport` con periodo, fecha, fuente (CA o Plugin) |
| `'approx'` | amarillo | `dataLoadedApprox` |
| `'none'` | info | `noMatchingData` |
| `'mismatch'` | naranja | `temporalMismatch` |
| `'error'` | rojo | `fetchError` |
| `'idle'` | oculto | — |

Si `cacheStatus === 'loaded'` pero **no hay `priceData`**, deshabilitar/ocultar el botón "Data" de export ([index.html:800](index.html#L800)) con tooltip explicativo.

**Criterio de aceptación Fase 2:** con un CSV+meta del Custom Analysis en la carpeta y rango temporal coincidente, al hacer doble click en la estrategia los gráficos aparecen **sin** pulsar Run y **sin** cargar precios. Rango no coincidente → mensaje correcto, no gráficos.

---

## 6. Fase 3 — Persistencia / ESCRITURA (File System Access API)

HTTP PUT no es viable. En su lugar se usa la **File System Access API** (`window.showDirectoryPicker()`), disponible en Chromium (motor de SQX). El usuario elige la carpeta de destino; el plugin escribe directamente ahí con `FileSystemFileHandle.createWritable()`.

### 6.1 Botón "Save Test Cache Data"

Texto fijo: **"Save Test Cache Data"**  
Icono: símbolo **ⓘ** a la derecha del texto, con tooltip CSS que incluye:
- Qué guarda (CSV de simulación + metadata JSON)
- Para qué sirve (reanuda los gráficos sin recalcular)
- Ruta recomendada: `user/extend/ResultsPlugins/DatabankMonkeyTest/cache/`

Aparece en **dos lugares**:
| Pestaña | Acción del botón |
|---|---|
| **Active Strategy** | Guarda CSV + meta.json de la estrategia activa en la carpeta elegida |
| **Bulk Databank** | Guarda CSV + meta.json de **todas** las estrategias calculadas en la carpeta elegida |

Los botones "Data" individuales por fila en Bulk se mantienen sin cambios (descargan solo el CSV de esa estrategia).

### 6.2 Construcción de archivos

#### CSV (`buildCsvString` — Fase 1)
```javascript
const csvText = buildCsvString(res, settings, priceData.value);
```
Para bulk se itera sobre `bulkResults.value`, buscando en `bulkStrategies.value` los `trades` de cada estrategia por nombre.

#### meta.json (`buildPluginMeta`)
```javascript
function buildPluginMeta(strategyName, trades, result, settings) {
  const range = computeStrategyRange(trades, result.period === 'Full' ? 'FULL' : result.period);
  return {
    strategyName,
    period: (result.period || 'FULL').toUpperCase(),
    tradeFromMs: range?.fromMs || 0,
    tradeToMs:   range?.toMs   || 0,
    numTrades:   trades.filter(t => !(t.OpenTime === t.CloseTime && t.ProfitLoss === 0)).length,
    realProfit:  result.realReturn,
    percentile:  settings.percentile,
    numMonkeys:  settings.numMonkeys,
    generatedAtUtc: Date.now(),
    source: 'Plugin'
  };
}
```

### 6.3 Escritura con File System Access API

```javascript
const saveToCache = async (filePairs) => {
  // filePairs: [{ name, csvText, metaJson }, ...]
  const dirHandle = await window.showDirectoryPicker({ mode: 'readwrite' });
  let saved = 0;
  for (const { name, csvText, metaJson } of filePairs) {
    const base = `${name}_monkey_simulation_data`;
    const csvFile  = await dirHandle.getFileHandle(`${base}.csv`,       { create: true });
    const metaFile = await dirHandle.getFileHandle(`${base}.meta.json`, { create: true });
    const w1 = await csvFile.createWritable();
    await w1.write(csvText); await w1.close();
    const w2 = await metaFile.createWritable();
    await w2.write(metaJson); await w2.close();
    saved++;
  }
  return saved;
};
```

Si el usuario cancela el selector de carpeta, la `Promise` rechaza con `AbortError` — se captura y no se muestra error.

### 6.4 Habilitación del botón

| Condición | Active Strategy | Bulk |
|---|---|---|
| Sin resultado calculado | deshabilitado | deshabilitado |
| Cálculo en progreso | deshabilitado | deshabilitado |
| Resultado disponible | habilitado | habilitado (≥1 estrategia calculada) |
| `fromExport: true` (auto-cargado desde cache) | deshabilitado (ya está guardado) | — |

### 6.5 Claves i18n nuevas

| Clave | EN | ES |
|---|---|---|
| `saveTestCacheData` | "Save Test Cache Data" | "Guardar datos en caché" |
| `saveTestCacheDataTip` | "Saves monkey simulation CSV + metadata to disk.\nLoad charts later without recalculating.\nRecommended path: …/DatabankMonkeyTest/cache/" | "Guarda el CSV de simulación + metadatos en disco.\nPermite cargar los gráficos sin recalcular.\nRuta recomendada: …/DatabankMonkeyTest/cache/" |
| `savingCache` | "Saving…" | "Guardando…" |
| `cacheSaveSuccess` | "Saved {count} strategy(ies) to selected folder." | "Guardadas {count} estrategia(s) en la carpeta seleccionada." |

**Criterio de aceptación Fase 3:** con un resultado calculado, clic en "Save Test Cache Data" abre el selector de carpeta; al elegir `cache/`, los archivos aparecen allí; la próxima vez que se seleccione la estrategia se auto-carga sin recalcular.

---

## 7. Fase 4 — Botón "Delete Cached Data"

### 7.1 Alcance según `settings.testTarget`

| `testTarget` | Qué se borra |
|---|---|
| `'active'` | Archivos de `activeStrategy.value.name` (`base.csv`, `base.meta.json`) |
| `'bulk'` | Archivos de cada nombre en `bulkStrategies.value` (itera todos) |
| `'both'` | Active + Bulk (unión) |

### 7.2 UI
Bajo el bloque Run/Cancel ([index.html:764-772](index.html#L764-L772)):
```html
<button v-if="!isCalculating"
  class="btn btn-sm delete-cache-btn"
  disabled
  :title="t('cacheDeleteUnavailable')">
  {{ t('deleteCachedData') }}
</button>
```
CSS: color `var(--text-muted)`, sin background de color, tamaño reducido, siempre `disabled`. El tooltip explica que el borrado debe hacerse manualmente en `user/exports/MonkeyTest/`.

> HTTP DELETE no está soportado por SQX. El botón se incluye como marcador de posición para futuras versiones o si SQX expone esta capacidad. No implementar lógica de borrado.

### 7.3 Lógica

No se implementa lógica de borrado (HTTP DELETE no disponible). El botón queda siempre `disabled` con tooltip `cacheDeleteUnavailable`.

**Criterio de aceptación Fase 4:** el botón aparece bajo "Run Monkey Test", visualmente subordinado, deshabilitado con tooltip explicativo. El usuario puede borrar archivos manualmente desde `user/exports/MonkeyTest/`.

---

## 8. Plan de pruebas

| # | Escenario | Resultado esperado |
|---|---|---|
| 1 | **Regresión manual:** Run Monkey Test con precios cargados | Charts y descarga CSV idénticos a hoy |
| 2 | **Auto-carga:** CSV+meta del Custom Analysis, rango coincide, sin precios cargados | Gráficos, badge verde, origen "Custom Analysis" |
| 3 | **Rango no coincide:** CSV de otra sesión/período | Sin gráficos, mensaje `temporalMismatch` |
| 4 | **Sin meta (legacy):** CSV sin `.meta.json`, rango coincide vía Open time | Gráficos, badge amarillo `approx` |
| 5 | **Recálculo manual sobre auto-cargado:** Run Monkey Test | Charts se actualizan al nuevo cálculo; caché sobrescrita |
| 6 | **Delete — active:** `testTarget='active'`, borrar | Solo archivos de la estrategia activa eliminados |
| 7 | **Delete — bulk:** `testTarget='bulk'`, borrar | Archivos de todas las estrategias bulk eliminados |
| 8 | **Delete — both:** `testTarget='both'`, borrar | Active + bulk eliminados |
| 9 | **Sin datos previos** | Mensaje `noMatchingData`, sin gráficos |
| 10 | **CSV corrupto** | Candidato descartado, no crashea, mensaje `fetchError` |
| 11 | **`parseDate` preservada:** cálculo manual funciona igual que antes | Sin regresión numérica en el worker |

---

## 9. Orden de ejecución y dependencias

```
[Test A: fetch read]    [Test B: HTTP PUT/DELETE]    ← PRIMERO, en paralelo
        │                         │
        ▼                         │
    Fase 1 (refactor buildCsvString + i18n)   ← segura, sin riesgo
        │                         │
        ▼                         ▼
    Fase 2 (lectura)     Fase 3 (escritura) → Fase 4 (delete)
        │                         │
        └──────────────────────────┘
                    ▼
            Fase de pruebas (§8)
```

- **Fase 1** se puede iniciar ya, no depende de los tests.
- **Fase 2** solo si Test A pasa. Si falla → no se implementa, documentar como limitación conocida.
- **Fase 3** solo si Test B confirma HTTP PUT. Si no → modo degradado (descarga doble + instrucción).
- **Fase 4** (Delete) solo si Test B confirma HTTP DELETE. Si no → botón visible pero deshabilitado con tooltip.

---

## 10. Decisiones cerradas (resueltas durante la preparación del plan)

| Decisión | Resolución |
|---|---|
| Política anti-duplicados | **Nombre canónico único** (`base.csv` siempre), sin sufijos `_2`. Confirmado por el Java. |
| Sufijos de descubrimiento | Sondear hasta `_5` como red de seguridad (el Java nunca los crea, pero el plugin podría). |
| Función `parseDate` | ~~No modificar la existente.~~ **REVISADO (Fase 6):** cambiar `new Date(y,m,d,h,min,s)` → `Date.UTC(y,m,d,h,min,s)` en los 5 ramos de formato cadena. El Bulk CSV exportado por SQX usa fechas UTC; interpretarlas como LOCAL introduce un offset = timezone del browser. |
| `TOL_MS` para matching | ~~`3_600_000` ms (1 h).~~ **REVISADO (Fase 6):** `7_200_000` ms (2 h) — absorbe offset de timezone (UTC+1/+2) y offsets de broker. Falso positivo es prácticamente imposible ya que el matching también compara `strategyName`. |
| Alcance de Delete | **Sigue `settings.testTarget`:** active / bulk / both. |
| Auto-carga bulk | ~~Fuera de esta iteración.~~ **REVISADO (Fase 6):** implementada mediante `bulkAutoLoadCache()`. |
| API de escritura de SQX | **No existe API PostMessage ni HTTP PUT/DELETE.** Java escribe con `FileWriter` nativo en `cache/` dentro del plugin. El plugin lee desde `cache/` via `fetch`. |

---

## 11. Decisiones aún abiertas (requieren confirmación o test en SQX)

1. **Test A (lectura):** ¿El iframe usa `file://` o `http://localhost`? Bloqueante para Fase 2.
2. **Test B (escritura/borrado):** ¿SQX sirve los archivos vía HTTP local con PUT/DELETE? Bloqueante para Fases 3 y 4.
3. **Formato de `OpenTime`/`CloseTime` en `ORDERS_RESPONSE`:** Verificar en consola del browser que llegan como epoch ms enteros (esperado). Si por alguna razón llegasen como strings, `parseDate` los maneja igualmente con su branch de regex — sin impacto en el matching.

---

## 12. Fase 5 — Rediseño del payload de caché (CSV mínimo + guardado instantáneo)

> **Esta fase SUSTITUYE el formato de caché definido en Fase 1 (§4.1 `buildCsvString`) y Fase 3 (§6.2).**
> El export por fila "Data" (CSV completo formato SQX) **se mantiene aparte** y no cambia (ver §12.7).
> Requiere un cambio espejo en el Custom Analysis (Java), documentado en
> [MTCustomAnalysisFixes.md](MTCustomAnalysisFixes.md).

### 12.1 Problema que resuelve

El CSV de caché actual replica el export de trades de SQX: **22 columnas × (N monos × T trades)**. Para N=1000 monos y T=500 trades son 11M de filas. Tiene dos costes graves:

1. **Guardado lento:** `buildCsvString` **no serializa lo ya calculado** — re-simula trade a trade cada mono (con nuevos `Math.random()`), repitiendo el trabajo pesado del worker en el hilo principal. Por eso "Save" tarda aunque el cálculo ya esté hecho.
2. **Peso en disco/RAM desproporcionado** para lo que los gráficos realmente necesitan.

**Verificación en código:** el lector de caché `parseCsvToMonkeyData` ([index.html:2021](index.html#L2021)) solo lee 3 columnas (`monkey id`, `balance`, `profit/loss`). Las otras 19 columnas **nunca se leen**. Son peso muerto.

### 12.2 Qué necesitan exactamente los gráficos (auditado)

| Gráfico / UI | Campo del `result` | Origen real |
|---|---|---|
| Histograma gaussiano (`paintGaussianHistogram`) | `monkeyProfits` (los **N** completos), `realReturn`, `zScore` | distribución completa → necesita los N profits (escalares) |
| Equity comparado (`paintEquityCurveComparison`) | `monkeyEquities` (**≤50** curvas), `realEquity`, `monkeyThreshold`, `realEquity[0]` | 50 curvas + umbral + balance inicial |
| Curva real de la estrategia | `realEquity` | **reconstruida en vivo** desde `GET_ORDERS` (`buildRealEquity`) — NO del CSV |
| Tarjetas KPI / tabla bulk | `realReturn`, `monkeyThreshold`, `zScore`, `rankPercentile`, `status`, `meanMonkey`, `stdMonkey` | escalares, calculados del **total de N monos** |

**Conclusión:** lo único voluminoso e imprescindible son **50 curvas de equity**. Todo lo demás son **N escalares de profit** (baratos) + un puñado de escalares KPI.

### 12.3 Nuevo formato de caché (schemaVersion 2)

Separación limpia: **CSV = la matriz voluminosa (50 curvas)**; **meta.json = todo lo demás** (escalares KPI + los N profits para el histograma).

#### CSV mínimo (`[name]_monkey_simulation_data.csv`)
Matriz "wide": una fila por curva, separador `;`, **decimales con punto** (sin comillas, formato máquina):

```
monkey_id;b0;b1;b2;...;bT
min;1000;1012.5;1004.1;...
q02;1000;998.0;...
...
max;1000;1050.2;...
```

- 50 filas (o N si N<50). `monkey_id` es etiqueta cosmética; el lector solo usa la serie de balances.
- **Selección de las 50 curvas** (mejora sobre el "primeros 50" actual): mono de **menor** profit (`min`), mono de **mayor** profit (`max`) y **48 intermedios** repartidos uniformemente por percentil de rank sobre el array ordenado: índices `round(k·(N-1)/49)` para `k=1..48`, deduplicados. Así el spaghetti muestra la envolvente real.

#### meta.json (`[name]_monkey_simulation_data.meta.json`)

```json
{
  "schemaVersion": 2,
  "strategyName": "Strategy 1.3.65",
  "period": "FULL",
  "tradeFromMs": 0,
  "tradeToMs": 0,
  "numTrades": 0,
  "numMonkeys": 1000,
  "percentile": 95,
  "initialBalance": 1000.0,
  "realProfit": 1234.5,
  "monkeyThreshold": 800.0,
  "meanMonkey": 100.0,
  "stdMonkey": 50.0,
  "zScore": 2.1,
  "rankPercentile": 97.3,
  "status": "PASSED",
  "monkeyProfits": [ /* N floats ordenados asc */ ],
  "generatedAtUtc": 1700000000000,
  "source": "Plugin"
}
```

- `monkeyProfits` (N escalares) → alimenta el histograma con fidelidad **idéntica** a hoy. Para N=1000 son ~8 KB; para N=10000 ~80 KB. Frente a los megas del CSV trade-level actual, sigue siendo una victoria enorme.
- `monkeyThreshold`, `meanMonkey`, `stdMonkey`, `zScore`, `rankPercentile`, `status` → **calculados del total de N monos** (cumple el requisito explícito: el umbral PASSED/FAILED NO sale de los 50 guardados).
- `initialBalance` → ancla las 50 curvas y la línea de umbral azul.

### 12.4 Cambios en el motor de cálculo (worker)

`runMonkeySimulation` ([index.html:~2470](index.html#L2470)) ya calcula `monkeyProfits` (N) y los KPIs del total. Cambios mínimos:

1. **Capturar el `shift` de cada mono** en un array `shifts[]` (N enteros, barato) durante el bucle.
2. Tras ordenar profits y calcular KPIs, **seleccionar los 50 índices** (min/max/48 por percentil) y **regenerar solo esas 50 curvas** con sus `shift` guardados (re-simulación de 50 monos = milisegundos). Así el worker **nunca retiene N curvas** — pico de RAM acotado a 50.
   - *Alternativa más simple si se prefiere:* retener las N curvas durante esa estrategia y podar a 50 al final (pico transitorio mayor, pero solo en el worker y liberado de inmediato).
3. El worker devuelve un **`cachePayload`** listo para serializar:
   ```javascript
   {
     curves: [[b0,...,bT], ...],   // ≤50, ya seleccionadas
     monkeyProfits: [...],         // N
     initialBalance, monkeyThreshold, meanMonkey, stdMonkey,
     zScore, rankPercentile, status, realReturn, numMonkeys
   }
   ```
   Sigue devolviendo `realEquity` para el pintado inmediato (no se serializa).

### 12.5 Memoria retenida tras el cálculo (huella mínima)

En `activeResult` / `bulkResults[i]` se retiene solo el `cachePayload` + `realEquity` + `name` + `period`. **NO se retiene `res.trades`** (hoy sí se retiene, §index.html:2671 y :2706). Justificación auditada: los trades fuente siguen disponibles en `activeStrategy.value.trades` y en `strategiesMap`/`bulkStrategies` (bulk), de modo que el export "Data" por fila puede tomarlos de ahí (§12.7). Esto recorta lo más pesado retenido hoy (T objetos × ~30 campos × nº estrategias).

> Decisión a confirmar al implementar: si algún punto no auditado leyera `res.trades`, optar por la variante de bajo riesgo (mantener `res.trades` retenido). El guardado instantáneo y el CSV mínimo se consiguen igual; solo se pierde el recorte de RAM de los trades.

### 12.6 Guardado instantáneo

`saveAllCacheData` deja de llamar a `buildCsvString`. Pasa a:

```javascript
const serializeCacheCsv = (payload) => {
  const head = 'monkey_id;' + payload.curves[0].map((_, i) => 'b' + i).join(';');
  const rows = payload.curves.map((c, i) => labelFor(i, payload) + ';' + c.join(';'));
  return [head, ...rows].join('\r\n');
};
// meta: JSON.stringify de los escalares + monkeyProfits ya en memoria
```

Todo es O(50·T) de construcción de string sobre datos ya en RAM → **instantáneo**, sin `priceData` ni `trades` ni re-simulación. `showDirectoryPicker()` sigue siendo el primer `await` (gesto de usuario).

### 12.7 Export "Data" por fila (se mantiene, NO es caché)

El botón "Data" de cada fila bulk ([index.html:~3175](index.html#L3175)) genera el **CSV completo trade-level formato SQX** vía `buildCsvString` — es un export de detalle on-demand, distinto del CSV de caché. Se mantiene tal cual (puede seguir re-simulando: es de una sola estrategia y bajo demanda). Si se aplica §12.5 (no retener `res.trades`), este export toma los trades de `activeStrategy.trades` / `strategiesMap` por nombre.

### 12.8 Lector de caché — soporte de ambos formatos

`attemptAutoLoad` / `parseCsvToMonkeyData` deben ramificar por `meta.schemaVersion`:

| Caso | Detección | Parseo |
|---|---|---|
| **v2 (nuevo)** | `meta.schemaVersion >= 2` | curvas desde CSV wide → `monkeyEquities`; `monkeyProfits` + escalares desde meta; `realEquity` en vivo |
| **v1 (legacy)** | sin `schemaVersion` | `parseCsvToMonkeyData` actual (trade-level, 3 columnas) — **se conserva intacto** para la transición |

Los 5 pares ya existentes en `cache/` son v1 y seguirán cargando. Cuando el Java se actualice (ver MTCustomAnalysisFixes.md) emitirá v2.

**Manejo del `monkeyThreshold` (decisión cerrada):** el umbral se calcula en el worker con los **N monos**, se retiene como **escalar en memoria** dentro del `cachePayload`, y al guardar se **vuelca tal cual** a `meta.monkeyThreshold`. El lector v2 lo **lee directamente del meta** y **nunca lo recomputa** desde las 50 curvas guardadas. Igual para `meanMonkey` y `stdMonkey` (funciones puras de los N profits): se vuelcan y se leen directos. Así el umbral PASSED/FAILED y la línea azul reflejan siempre los N monos reales del test, no una muestra de 50.

> `zScore` y `rankPercentile` dependen del `realReturn`. Como en la carga v2 el `realReturn` se reconstruye en vivo desde las órdenes, se recomputan con ese `realReturn` en vivo (contra `meanMonkey`/`stdMonkey` y `monkeyProfits` del meta) para mantener coherencia con la curva real mostrada. Los valores escalares guardados sirven de referencia/sanity.

### 12.9 i18n

Sin claves nuevas. `saveTestCacheDataTip` puede ajustarse para mencionar que el guardado es instantáneo (opcional).

### 12.10 Criterios de aceptación Fase 5

1. **Guardado instantáneo:** con resultados calculados (activa y/o bulk), "Save Test Cache Data" escribe en <1 s sin importar N ni T.
2. **Gráficos idénticos:** tras guardar y recargar desde caché v2, histograma y equity se ven **igual** que recién calculados (mismo histograma, misma envolvente con min/max visibles, misma línea de umbral azul).
3. **Umbral correcto:** `monkeyThreshold` proviene del total de N monos, no de los 50 guardados (verificable comparando con el cálculo en vivo).
4. **Peso reducido:** el CSV de caché pasa de N×T×22 celdas a ≤50×(T+1).
5. **Compatibilidad:** los CSV v1 existentes siguen auto-cargando sin error.
6. **Sin regresión:** el export "Data" por fila sigue produciendo el CSV completo formato SQX.

### 12.11 Orden de implementación Fase 5

```
1. Worker: capturar shifts + selección 50 (min/max/48) + emitir cachePayload   [núcleo]
2. Retención: guardar cachePayload en activeResult/bulkResults (§12.5)
3. Guardado: serializeCacheCsv + meta v2 → saveAllCacheData instantáneo (§12.6)
4. Lector: ramificar v2/v1 en attemptAutoLoad/parseCsvToMonkeyData (§12.8)
5. (Espejo, fuera de este repo) Java emite v2 → MTCustomAnalysisFixes.md
6. Pruebas §12.10
```

---

## 13. Fase 6 — Bugfixes post-implementación ✅ IMPLEMENTADO

### 13.1 Fix B — `parseDate` UTC + `TOL_MS` ampliado

**Problema:** `buildPluginMeta` guarda `tradeFromMs`/`tradeToMs` calculados desde los trades del Bulk CSV, parseados por `parseDate` con `new Date(y,m,d,h,min,s)` (hora LOCAL). `attemptAutoLoad` los compara con las mismas fechas según `GET_ORDERS` (epoch ms UTC puros de SQX). Si el CSV exportado usa fechas UTC y el browser está en UTC+1/UTC+2, el offset (1–2h) excede `TOL_MS` en verano → mismatch espurio.

**Fix aplicado en `DatabankMonkeyTest/index.html`:**
- Los 5 `new Date(y, mon, d, h, min, sec).getTime()` de `parseDate` cambiados a `Date.UTC(y, mon, d, h, min, sec)`.
- `TOL_MS` cambiado de `3_600_000` (1h) a `7_200_000` (2h).

**Regresión:** la rama de epoch numérico (10–13 dígitos) no se toca — era ya correcta. Los meta.json del Custom Analysis usan epoch ms directamente → no pasan por `parseDate` → sin impacto.

### 13.2 Fix A — `bulkAutoLoadCache()`: auto-carga caché en Bulk tab

**Problema:** la pestaña Bulk Databank Test no intentaba leer la caché al cargar el CSV de trades. El grid quedaba vacío hasta ejecutar "Run Monkey Test" manualmente.

**Fix aplicado en `DatabankMonkeyTest/index.html`:**
- Nueva función `bulkAutoLoadCache()` que itera `bulkStrategies.value`, llama a `discoverCandidates` + `matchesMeta` para cada estrategia, y precarga el grid con los resultados que coincidan.
- Llamada al final de la carga del Bulk CSV (justo después de `validateSqxFiles()`).
- Reutiliza íntegramente: `discoverCandidates`, `matchesMeta`, `selectBestCandidate`, `parseCacheCsvV2`, `parseCsvToMonkeyData`, `buildRealEquity`, `buildKpisFromMeta`, `buildKpis`, `computeStrategyRange`.

**Comportamiento:** grid se pre-rellena con resultados cacheados al cargar el CSV. "Run Monkey Test" sigue recalculando todo desde cero y sobrescribe. Estrategias sin caché no aparecen en el grid hasta calcular.

### 13.3 Criterios de aceptación Fase 6

| # | Escenario | Resultado esperado |
|---|---|---|
| B1 | Ejecutar test en Bulk, guardar caché → ir a Active Strategy | Badge verde "Loaded from cache", sin mismatch |
| B2 | Datos de agosto (UTC+2 verano) | Sin mismatch (cubierto por TOL_MS = 2h) |
| B3 | Caché del Custom Analysis (Java, epoch ms en meta) | Sin regresión: sigue auto-cargando correctamente |
| A1 | Con caché en cache/, cargar Bulk Trades CSV | Grid se pre-rellena sin pulsar "Run" |
| A2 | Sin caché, cargar Bulk CSV | Grid vacío; "Run Monkey Test" habilitado |
| A3 | Resultados de caché visibles, pulsar "Run Monkey Test" | Recálculo sobrescribe los cacheados |
| A4 | Guardar desde Bulk tab → verificar en Active Strategy | Active tab carga la caché sin mismatch |

---

## 14. Fase 7 — Sincronización bidireccional entre pestañas ✅ IMPLEMENTADO

### 14.1 Problema

Tres bugs de flujo de datos entre las pestañas Active Strategy y Bulk Databank Test:

**Bug 1 — `testTarget` bloquea auto-carga en Active tab**: La condición `if (settings.testTarget === 'active' || settings.testTarget === 'both')` en `ORDERS_RESPONSE` impedía llamar a `attemptAutoLoad` cuando `testTarget === 'bulk'`. Al hacer doble clic en una estrategia del databank de SQX, `STRATEGY_DATA` disparaba → `activeResult = null` → pero sin llamar `attemptAutoLoad` → Active tab mostraba "No strategy selected" permanentemente.

**Bug 2 — Sin fallback en memoria**: Incluso si se elimina la condición anterior, `attemptAutoLoad` buscaba SOLO en disco (`cache/`). Si no había archivo guardado (el usuario había calculado en Bulk pero no guardado), el resultado de Bulk no se mostraba en Active tab.

**Bug 3 — Sin sincronización cruzada de resultados**: `activeResult` y `bulkResults` eran refs completamente independientes. Calcular en un tab no actualizaba el otro, causando que el grid Bulk mostrara datos obsoletos tras recalcular en Active, y viceversa.

### 14.2 Fixes aplicados en `DatabankMonkeyTest/index.html`

#### Fix 1 — Eliminar condición `testTarget` de `ORDERS_RESPONSE`

`attemptAutoLoad` se llama SIEMPRE al recibir `ORDERS_RESPONSE`, independientemente de `testTarget`. El selector "Perform Test On" solo controla qué calcula el botón "Run Monkey Test", no si se intenta auto-cargar.

```javascript
// ANTES:
if (settings.testTarget === 'active' || settings.testTarget === 'both') {
  cacheStatus.value = 'loading';
  attemptAutoLoad(activeStrategy.value.name, list);
}

// DESPUÉS:
cacheStatus.value = 'loading';
attemptAutoLoad(activeStrategy.value.name, list);
```

#### Fix 2 — Fallback a `bulkResults` en `attemptAutoLoad`

Nueva función `applyBulkFallback(entry)` (definida antes de `attemptAutoLoad`): aplica una entrada de `bulkResults` como `activeResult` sin badge de caché (resultado en memoria, no en disco).

En `attemptAutoLoad`, los dos puntos de retorno anticipado ahora comprueban primero si hay un resultado coincidente en `bulkResults`:

```javascript
// Sin archivos en cache/:
if (candidates.length === 0) {
  const bulkFallback = bulkResults.value.find(r => r.name === strategyName);
  if (bulkFallback) { applyBulkFallback(bulkFallback); return; }
  cacheStatus.value = 'none'; return;
}

// Archivos existen pero no coincide el período:
if (matching.length === 0) {
  const bulkFallback = bulkResults.value.find(r => r.name === strategyName);
  if (bulkFallback) { applyBulkFallback(bulkFallback); return; }
  cacheStatus.value = 'mismatch'; return;
}
```

#### Fix 3 — Sincronización bidireccional

**Bulk → Active** (handler `PROGRESS` del worker bulk):
```javascript
if (result) {
  bulkResults.value.push(result);
  if (activeStrategy.value?.name === result.name) {
    activeResult.value = result;
    nextTick(() => drawActiveCharts());
  }
}
```

**Active → Bulk** (handler `COMPLETE_SINGLE` del worker active):
```javascript
activeResult.value = result;
const bulkIdx = bulkResults.value.findIndex(r => r.name === result.name);
if (bulkIdx !== -1) {
  bulkResults.value[bulkIdx] = { ...result };
  nextTick(() => drawBulkPie());
}
```

**Cache Bulk → Active** (`bulkAutoLoadCache`, al finalizar la carga):
```javascript
if (loaded.length > 0) {
  bulkResults.value = loaded;
  const activeMatch = loaded.find(r => r.name === activeStrategy.value?.name);
  if (activeMatch) applyBulkFallback(activeMatch);
}
```

#### Fix 4 — Redibujado de Active charts al cambiar de tab

**Problema:** cuando Bulk calcula y termina con la pestaña Bulk visible (activa), el handler `PROGRESS` llama a `drawActiveCharts()`. Pero el canvas del tab Active está oculto fuera del DOM (`v-show="currentTab === 'active"`), así que tiene dimensiones 0 → el dibujo no se ve. Al cambiar a Active tab, el canvas ya tiene dimensiones pero nadie lo redibuja → imagen vacía hasta hacer doble clic en la estrategia.

**Fix:** `watch(currentTab)` que, al cambiar a `'active'` tab, redibuja automáticamente si ya hay `activeResult` en memoria.

Cambios:
1. Añadir `watch` al import de Vue (junto a `createApp`, `ref`, etc.)
2. Insertar el watch antes de `onMounted`:

```javascript
watch(currentTab, (newTab) => {
  if (newTab === 'active' && activeResult.value) {
    nextTick(() => drawActiveCharts());
  }
});
```

### 14.3 Nota sobre diferencia de Z-score entre tabs

Cuando se calculan en sesiones distintas (Bulk en un momento, Active en otro), los Z-scores pueden diferir ligeramente porque la simulación usa `Math.random()` (distinto seed en cada ejecución). Con N=1000 la varianza es pequeña pero existe. No es un bug — es varianza estadística esperada. Tras los fixes de esta fase, el Z-score siempre muestra el del ÚLTIMO cálculo que afecta a esa estrategia, de modo que ambas pestañas muestran el mismo valor en todo momento.

### 14.4 Criterios de aceptación Fase 7

| # | Escenario | Resultado esperado |
|---|---|---|
| C1 | `testTarget='bulk'`, click estrategia en SQX databank → pestaña Active | Muestra gráficos (desde caché o desde bulkResults en memoria) — NO "No strategy selected" |
| C2 | Calcular Bulk con estrategia X incluida → cambiar a Active → click estrategia X | Gráficos de la estrategia X del último cálculo Bulk se muestran en Active tab |
| C3 | Calcular Active para estrategia X → ver pestaña Bulk | Grid Bulk muestra el Z-score actualizado para estrategia X |
| C4 | Cargar CSV Bulk con caché existente → estrategia activa entre las cargadas | Active tab se actualiza automáticamente con el resultado de caché |

---

## 15. Fase 8 — Mejoras de UI en pestaña Active Strategy Test ✅ IMPLEMENTADO

Dos cambios menores de presentación en la fila de KPI cards de la pestaña Active Strategy Test.

### 15.1 Cambio 1 — Badge de estado PASSED/FAILED con color de fondo

**Problema:** El estado (PASSED/FAILED/LOW TRADES) se muestra con texto de color (`text-success`/`text-danger`) pero sin fondo destacado, a diferencia de la pestaña Bulk que usa `.badge-success`/`.badge-danger`/`.badge-warning` con fondo de color.

**Archivo:** `DatabankMonkeyTest/index.html`

**Ubicación:** líneas 886–888 (dentro del primer `kpi-card` del tab `active`).

**Antes:**
```html
<div class="kpi-val" :class="activeResult.status === 'PASSED' ? 'text-success' : 'text-danger'">
  {{ t(activeResult.status.toLowerCase()) }}
</div>
```

**Después:**
```html
<div class="kpi-val">
  <span class="badge"
    :class="activeResult.status === 'PASSED' ? 'badge-success' : (activeResult.status === 'LOW TRADES' ? 'badge-warning' : 'badge-danger')">
    {{ t(activeResult.status.toLowerCase().replace(' ', '')) }}
  </span>
</div>
```

La lógica de clases es idéntica a la del grid Bulk (línea 1009). El `t()` lookup sigue el mismo patrón: `'passed'`, `'failed'`, `'lowtrades'` (fallback al key sin traducción para LOW TRADES, igual que Bulk).

### 15.2 Cambio 2 — Añadir KPI cards de Monkey Threshold y Monkey Test Result

**Problema:** La fila de KPI cards de Active tab muestra solo 3 métricas (nombre+estado, Z-Score, Real Profit). Faltan `monkeyThreshold` y `rankPercentile`, que sí aparecen en el grid Bulk y son esenciales para interpretar el resultado.

**Archivo:** `DatabankMonkeyTest/index.html`

**Ubicación:** después de la kpi-card de Z-Score y antes de cerrar `</div>` del `kpi-row` — es decir, entre la card de Z-Score (líneas 895–900) y la card de Real Profit (líneas 901–906).

Insertar dos nuevas cards:

```html
<div class="kpi-card">
  <div class="kpi-card-content">
    <div class="kpi-label">{{ t('monkeyThreshold') }}</div>
    <div class="kpi-val">${{ activeResult.monkeyThreshold.toFixed(2) }}</div>
  </div>
</div>
<div class="kpi-card">
  <div class="kpi-card-content">
    <div class="kpi-label">{{ t('monkeyTestResult') }}</div>
    <div class="kpi-val">{{ activeResult.rankPercentile.toFixed(1) }}%</div>
  </div>
</div>
```

El orden final de las 5 cards será:
1. Nombre + badge PASSED/FAILED (con mini pie chart y botón "Data")
2. Z-Score
3. **Monkey Threshold** ← nueva
4. **Monkey Test Result (%)** ← nueva
5. Real Profit

Las claves `monkeyThreshold` y `monkeyTestResult` ya existen en el objeto LOCALES inline (líneas 1120–1121 en, 1202–1203 es) y en los archivos `locales/en.json` y `locales/es.json` — no requieren cambios en locales.

### 15.3 Criterios de aceptación Fase 8

| # | Escenario | Resultado esperado |
|---|---|---|
| D1 | Estrategia con status PASSED en Active tab | Badge verde con texto "PASSED" / "APROBADA" |
| D2 | Estrategia con status FAILED en Active tab | Badge rojo con texto "FAILED" / "FALLIDA" |
| D3 | Estrategia con status LOW TRADES en Active tab | Badge amarillo con texto "LOW TRADES" / "POCOS TRADES" (o fallback "lowtrades") |
| D4 | Resultado con `monkeyThreshold = 850.00` visible | Card "Monkey Threshold" muestra "$850.00" |
| D5 | Resultado con `rankPercentile = 97.3` visible | Card "Monkey Test Result" muestra "97.3%" |
| D6 | Sin resultado activo | KPI row oculta (v-if="activeResult") — sin cambio |
| C5 | Calcular Bulk → estrategia activa entre las calculadas → sin cambiar de estrategia | Active tab muestra resultados en tiempo real conforme avanza el cálculo Bulk |
