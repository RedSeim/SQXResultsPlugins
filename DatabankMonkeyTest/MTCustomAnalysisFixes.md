# MT Custom Analysis — Contrato de caché v2 con el ResultsPlugin DatabankMonkeyTest

> **Propósito:** documento de referencia completo para adaptar el Custom Analysis Java
> (`user/extend/Snippets/SQ/CustomAnalysis/MonkeyTest.java`) de modo que produzca
> los artefactos de caché que el plugin `DatabankMonkeyTest` auto-carga y muestra
> sin necesidad de recalcular.
>
> El Java **no es objeto del proyecto plugin** — este documento es el insumo para
> planificar ese cambio de forma independiente.
>
> Contexto del plugin: [`MonkeyTestImplementationPlan.md`](MonkeyTestImplementationPlan.md)
> (Fases 5–8 son las relevantes para la integración de caché).

---

## 1. Por qué cambia el formato

El plugin usa un **payload de caché mínimo** diseñado para repintar los gráficos al instante
sin recalcular. El CSV trade-level antiguo (22 columnas × N monos × T trades) era excesivamente
pesado: el plugin solo necesitaba 3 columnas y descartaba el resto.

El formato v2 separa responsabilidades:

- **CSV** — solo la matriz voluminosa: ≤ 50 curvas de equity de mono (una fila por curva).
- **meta.json** — todos los escalares KPI + el array completo de N profits (para el histograma).

El plugin detecta v2 por `meta.schemaVersion >= 2`. Si no hay `schemaVersion` en el meta,
trata el par como formato v1 legacy (compatibilidad hacia atrás conservada).

---

## 2. Qué consume el plugin de estos archivos

| Elemento de UI | Dato necesario | Fuente en v2 |
|---|---|---|
| Histograma gaussiano (campana) | Array de N profits de mono + `realReturn` + `zScore` | `meta.monkeyProfits` + escalares |
| Curvas de equity (spaghetti) | ≤ 50 curvas de balance | CSV |
| Línea de umbral azul (PASSED/FAILED) | `monkeyThreshold` + `initialBalance` | `meta` |
| Curva real de la estrategia | `realEquity` | **NO la genera el Java** — el plugin la reconstruye en tiempo real desde `GET_ORDERS` de SQX |
| KPI card "Monkey Threshold" | `monkeyThreshold` | `meta` |
| KPI card "Monkey Test Result (%)" | `rankPercentile` | `meta` |
| KPI card "Z-Score" | `zScore` | `meta` |
| KPI card "Real Profit" | `realReturn` | `meta` |
| Badge PASSED / FAILED / LOW TRADES | `status` | `meta` |
| Badge de origen (Custom Analysis / Plugin calc) | `source` | `meta` |
| Selección de caché más reciente | `generatedAtUtc` | `meta` |

> **Nota importante (Fase 8):** a partir de la Fase 8 del plugin, la pestaña
> "Active Strategy Test" muestra 5 KPI cards: nombre+badge de estado, Z-Score,
> Monkey Threshold, Monkey Test Result (%), Real Profit. Todos estos valores
> vienen de `meta` — si el campo falta o es `null`, la card no se muestra correctamente.

---

## 3. Archivo CSV v2

### 3.1 Ruta y nombre

```
user/extend/ResultsPlugins/DatabankMonkeyTest/cache/[NombreEstrategia]_monkey_simulation_data.csv
```

El nombre de archivo debe usar exactamente el `strategyName` tal como lo conoce SQX
(sin transformaciones de camelCase ni slug).

### 3.2 Formato

```
monkey_id;b0;b1;b2;...;bT
min;1000.00;1012.50;1004.10;...
q01;1000.00;998.00;...
...
q48;1000.00;...
max;1000.00;1050.20;...
```

Reglas estrictas:

| Regla | Detalle |
|---|---|
| Separador de columnas | `;` (punto y coma) |
| Separador decimal | `.` (punto) — nunca coma |
| Primera columna | `monkey_id`: etiqueta cosmética, no se parsea para dibujar |
| Cada fila | Curva de balance completa: `b0` … `bT` (longitud = nº trades + 1) |
| `b0` | `initialBalance` — igual en todas las filas |
| Número de filas | 50 (o N si N < 50) |
| Comillas | Ninguna |
| BOM / encoding | UTF-8 sin BOM |

### 3.3 Selección de las ≤ 50 curvas

No vale "las primeras 50 monos". Se eligen sobre el array de profits **ordenado ascendentemente**:

- `min` → mono de **menor** profit (índice 0 del array ordenado)
- `max` → mono de **mayor** profit (índice N-1)
- 48 intermedios uniformes por percentil: índices `round(k × (N-1) / 49)` para `k = 1..48`
  (deduplicar si N es pequeño)

Para generar estas curvas, el Java debe conservar en memoria la curva de equity de cada
mono candidato (o el seed + offset para regenerar solo esos 50 en una segunda pasada).

---

## 4. Archivo meta.json v2

### 4.1 Ruta y nombre

```
user/extend/ResultsPlugins/DatabankMonkeyTest/cache/[NombreEstrategia]_monkey_simulation_data.meta.json
```

### 4.2 Estructura completa

```json
{
  "schemaVersion": 2,
  "strategyName": "Strategy 1.3.65",
  "period": "FULL",
  "tradeFromMs": 1609459200000,
  "tradeToMs": 1640995200000,
  "numTrades": 512,
  "numMonkeys": 1000,
  "percentile": 95,
  "initialBalance": 10000.0,
  "realProfit": 1234.5,
  "monkeyThreshold": 800.0,
  "meanMonkey": 100.0,
  "stdMonkey": 50.0,
  "zScore": 2.1,
  "rankPercentile": 97.3,
  "status": "PASSED",
  "monkeyProfits": [-1200.0, -1100.5, ..., 1800.2],
  "generatedAtUtc": 1700000000000,
  "source": "CustomAnalysis"
}
```

### 4.3 Campos — definición y cómo calcularlos

| Campo | Tipo | Descripción y cómo computarlo |
|---|---|---|
| `schemaVersion` | int | **`2`** fijo. Marca que activa la rama v2 del lector. **Obligatorio.** |
| `strategyName` | string | Nombre exacto de la estrategia (igual al de SQX). |
| `period` | string | `"FULL"` / `"IS"` / `"OOS"`. Mayúsculas. |
| `tradeFromMs` | long (epoch ms UTC) | Mínimo de `OpenTime` de los trades reales del período. **Epoch ms UTC sin conversión de zona.** |
| `tradeToMs` | long (epoch ms UTC) | Máximo de `CloseTime` de los trades reales del período. **Epoch ms UTC sin conversión de zona.** |
| `numTrades` | int | Nº de trades reales del período (excluye balance ops y trades con `Open==Close && PL==0`). |
| `numMonkeys` | int | N monos simulados en este cálculo. |
| `percentile` | number | Percentil de aprobación usado (ej: 95). |
| `initialBalance` | number | Balance inicial = `b0` común a todas las curvas del CSV. |
| `realProfit` | number | Profit real de la estrategia en el período. |
| `monkeyThreshold` | number | `monkeyProfits[floor(N × percentile / 100)]`. Línea azul en los gráficos. **Debe derivarse de TODOS los N monos, no de los 50 guardados.** |
| `meanMonkey` | number | Media de los N profits de mono. |
| `stdMonkey` | number | Desviación estándar **muestral** de los N profits (`÷ (N-1)`). |
| `zScore` | number | `stdMonkey > 0 ? (realProfit - meanMonkey) / stdMonkey : 0`. |
| `rankPercentile` | number | `(nº de monos cuyo profit < realProfit) / N × 100`. |
| `status` | string | Ver §4.4. |
| `monkeyProfits` | array\<number\> | **Los N profits, ordenados ascendentemente.** Es lo que dibuja el histograma gaussiano. |
| `generatedAtUtc` | long (epoch ms) | Timestamp de generación. El plugin usa el más reciente para elegir entre varios candidatos. |
| `source` | string | **`"CustomAnalysis"`** — el plugin muestra un badge de origen diferenciado. No cambiar. |

### 4.4 Valores exactos del campo `status`

El plugin usa el string de `status` directamente para lógica de badge y clasificación.
Los tres únicos valores válidos son:

| Valor | Condición | Badge en UI |
|---|---|---|
| `"PASSED"` | `numTrades >= 20 && realProfit > monkeyThreshold` | Verde |
| `"FAILED"` | `numTrades >= 20 && realProfit <= monkeyThreshold` | Rojo |
| `"LOW TRADES"` | `numTrades < 20` | Amarillo |

> **Importante:** el string `"LOW TRADES"` lleva **espacio**, no guion ni underscore.
> Cualquier otra variante romperá la lógica de clasificación del plugin.

---

## 5. Fórmulas de referencia (espejo exacto del worker del plugin)

Para que Java y plugin den números idénticos, replicar exactamente:

```
monkeyProfits.sort(asc)

mean      = sum(profits) / N
variance  = sum((p - mean)^2) / (N - 1)      // desviación muestral
std       = sqrt(variance)

zScore    = std > 0 ? (realProfit - mean) / std : 0

threshIdx = floor(N * percentile / 100)        // clamp a [0, N-1]
threshold = monkeyProfits[threshIdx]

beaten    = count(p in monkeyProfits where p < realProfit)
rankPct   = beaten / N * 100

status    = numTrades < 20 ? "LOW TRADES"
          : realProfit > threshold ? "PASSED" : "FAILED"
```

> La construcción de la curva de cada mono (shift temporal, manejo de SL/TP/EndTest,
> salida por barras) **ya existe en el Java actual** y no cambia: solo cambia qué
> se persiste y en qué formato.

---

## 6. Cómo el plugin selecciona y valida la caché (algoritmo de matching)

Cuando SQX selecciona una estrategia, el plugin:

1. **Descubre candidatos** en `cache/`: busca archivos `*_monkey_simulation_data.meta.json`
   cuyo nombre empiece por el `strategyName`.

2. **Valida temporalmente** cada candidato comparando su `tradeFromMs`/`tradeToMs` con el
   rango de fechas de los trades reales (obtenidos de `GET_ORDERS`):

   ```
   TOL_MS = 7_200_000   // 2 horas — absorbe offset de broker + redondeo
   match = |meta.tradeFromMs - strategy.tradeFromMs| <= TOL_MS
        && |meta.tradeToMs   - strategy.tradeToMs|   <= TOL_MS
        && meta.period == strategy.period   // 'FULL' / 'IS' / 'OOS'
   ```

   > **Por qué 2h:** el plugin parsea fechas del CSV exportado por SQX con UTC estricto
   > (`Date.UTC()`), pero algunos brokers y versiones de SQX pueden desplazar los timestamps
   > hasta ±1h. El margen de 2h cubre UTC±1 con holgura.

3. **Elige el mejor** candidato entre los que pasan la validación: el de `generatedAtUtc`
   más reciente.

4. **Carga CSV + meta** y pinta los gráficos sin recalcular.

**Implicación para el Java:** los campos `tradeFromMs` y `tradeToMs` en meta.json **deben
ser epoch ms UTC** (no local time del servidor Java). Si el Java corre en un servidor con
UTC+1 o UTC+2, usar `sdf.setTimeZone(TimeZone.getTimeZone("UTC"))` antes de parsear.

---

## 7. Resumen de cambios necesarios en `MonkeyTest.java`

| # | Cambio | Detalle |
|---|---|---|
| 1 | Dejar de escribir el CSV trade-level de 22 columnas | Sustituir por CSV wide de §3 |
| 2 | Implementar selección de 50 curvas | min/max/48 uniformes por percentil (§3.3) |
| 3 | Ampliar meta.json a formato v2 | Añadir `schemaVersion: 2`, `initialBalance`, `meanMonkey`, `stdMonkey`, `zScore`, `rankPercentile`, `status`, `monkeyProfits[]` (§4) |
| 4 | No incluir la curva real en ningún archivo | El plugin la reconstruye desde `GET_ORDERS` |
| 5 | Garantizar timestamps UTC en meta.json | `tradeFromMs` / `tradeToMs` deben ser epoch ms UTC |
| 6 | Mantener rutas y nombres de archivo | `cache/[Nombre]_monkey_simulation_data.csv` y `.meta.json` |
| 7 | Escribir `"source": "CustomAnalysis"` en meta.json | El plugin muestra un badge de origen diferenciado |

---

## 8. Compatibilidad y migración

- El plugin **conserva la rama de lectura v1** (CSV trade-level sin `schemaVersion`) para archivos
  generados por versiones antiguas del Java. No es urgente migrar; se regeneran al re-ejecutar.
- En cuanto el Java emita v2, los archivos nuevos serán mucho más ligeros y el plugin los
  preferirá por `generatedAtUtc` más reciente.
- **No mezclar formatos en un mismo par:** si `meta.schemaVersion == 2`, el CSV debe ser el
  formato wide (§3); si no hay `schemaVersion`, el CSV debe ser el trade-level legacy.

---

## 9. Estado de la UI del plugin (Fase 8, junio 2026)

Referencia para saber qué campos son visibles al usuario y cómo se usan:

### Pestaña "Active Strategy Test" — fila de KPI cards

| Card | Campo usado | Formato mostrado |
|---|---|---|
| Nombre + badge estado | `status` | Badge verde/rojo/amarillo con texto PASSED/FAILED/LOW TRADES |
| Z-Score | `zScore` | `X.XX σ` |
| Monkey Threshold | `monkeyThreshold` | `$X.XX` |
| Monkey Test Result | `rankPercentile` | `XX.X%` |
| Real Profit | `realReturn` (recalculado de trades) | `$X.XX` |

### Pestaña "Bulk Databank Test" — columnas de la tabla

| Columna | Campo | Formato |
|---|---|---|
| Real Profit | `realReturn` | `$X.XX` |
| Monkey Threshold | `monkeyThreshold` | `$X.XX` |
| Monkey Test Result | `rankPercentile` | `XX.X%` |
| Z-Score | `zScore` | `X.XX σ` |
| Status | `status` | Badge verde/rojo/amarillo |

### Badge de origen (ambas pestañas)

Si el resultado proviene de caché con `source == "CustomAnalysis"`, el badge muestra
"Custom Analysis". Si proviene de un cálculo en vivo del plugin, muestra "Plugin calculation".

---

## 10. Checklist de validación (lado Java)

- [ ] `meta.schemaVersion == 2`
- [ ] CSV tiene cabecera `monkey_id;b0;...;bT` con ≤ 50 filas de datos
- [ ] Separador `;`, decimales con `.`, sin comillas, UTF-8 sin BOM
- [ ] Fila `min` y fila `max` presentes; 48 intermedios seleccionados por percentil uniforme
- [ ] `monkeyProfits` tiene N elementos exactamente, ordenados ascendentemente
- [ ] `monkeyThreshold == monkeyProfits[floor(N × percentile / 100)]`
- [ ] `status` es uno de: `"PASSED"`, `"FAILED"`, `"LOW TRADES"` (con espacio, sin variantes)
- [ ] `status` es coherente con `realProfit` vs `monkeyThreshold` y `numTrades`
- [ ] `initialBalance == b0` de todas las curvas del CSV
- [ ] `tradeFromMs` y `tradeToMs` son epoch ms **UTC** (no local time del servidor)
- [ ] `source == "CustomAnalysis"`
- [ ] `generatedAtUtc` es el epoch ms UTC del momento de generación
- [ ] Ruta del CSV: `.../DatabankMonkeyTest/cache/[NombreEstrategia]_monkey_simulation_data.csv`
- [ ] Ruta del meta: `.../DatabankMonkeyTest/cache/[NombreEstrategia]_monkey_simulation_data.meta.json`
- [ ] Al seleccionar la estrategia en SQX → Active Strategy Test tab → los gráficos se cargan automáticamente sin pulsar "Run Monkey Test"
- [ ] Los valores de KPI (monkeyThreshold, rankPercentile, zScore, status) coinciden con un cálculo en vivo del plugin para los mismos trades y parámetros
