# Plan de Mejora del Custom Analysis: Simulación Avanzada de Monkey Test (Contrato v3)

> **Propósito:** Documento de especificación técnica y de referencia para actualizar el código del Custom Analysis Java (`user/extend/Snippets/SQ/CustomAnalysis/MonkeyTest.java`) de StrategyQuant X. El objetivo es alinear el motor de simulación en Java con las nuevas funcionalidades implementadas en el ResultsPlugin `DatabankMonkeyTest`, asegurando la compatibilidad de lectura, escritura y el formato del contrato de caché.

---

## 1. Introducción y Objetivos

Para que los análisis de robustez ejecutados en segundo plano mediante tareas automatizadas (Custom Analysis en Java) arrojen resultados idénticos a los del simulador en el ResultsPlugin, el motor de cálculo en Java debe incorporar los mismos parámetros de control:
1.  **Modos de Replicación de Operaciones:** Salida tradicional por SL/TP, salida por duración fija promedio o salida por duración individual del trade original.
2.  **Modos de Desplazamiento Circular:** Desplazamiento temporal constante de todo el bloque de operaciones o desplazamiento temporal aleatorio e independiente por cada operación.
3.  **Matemáticas de Robustez:** Evitar divisiones por diferencias de precios infinitesimales o nulas y gestionar correctamente la duración media de operaciones.

---

## 2. Parámetros de Configuración a Integrar

El snippet de Java debe definir o recibir en sus estructuras de configuración los siguientes parámetros de control (con sus valores predeterminados correspondientes):

| Parámetro Java | Tipo / Valores | Valor por Defecto | Descripción |
| :--- | :--- | :--- | :--- |
| `replicationMode` | `String` (`"SLTP"`, `"AvgBars"`, `"IndivBars"`) | `"IndivBars"` | Controla las condiciones de salida y exposición del trade. |
| `shiftingMode` | `String` (`"Constant"`, `"Random"`) | `"Random"` | Controla la forma de aplicar el desplazamiento temporal. |

---

## 3. Especificación de los Modos de Replicación (Replication Mode)

### A. Modo `SLTP` (SL & TP Distance Replication)
*   **Comportamiento:** Réplica exacta del comportamiento clásico de los monos en StrategyQuant.
*   **Lógica:**
    *   Para cada trade $i$, se calculan las distancias absolutas de Stop Loss ($distSL = |OpenPrice - SLLevel|$) y Profit Target ($distTP = |OpenPrice - PTLevel|$).
    *   Si el trade original no tenía SL o TP, se asume distancia infinita (o un límite muy alto).
    *   En el trade simulado (que empieza en una barra aleatoria $t'$):
        *   Si es Buy: $slPrice = entryPrice - distSL$, $tpPrice = entryPrice + distTP$.
        *   Si es Sell: $slPrice = entryPrice + distSL$, $tpPrice = entryPrice - distTP$.
    *   Se recorren las barras secuencialmente desde $t'$. El trade se cierra en la primera barra donde el precio toque el $slPrice$ (usando el mínimo de la vela) o el $tpPrice$ (usando el máximo de la vela).
    *   Si no toca ninguno, el trade se cierra por límite de tiempo al transcurrir el número máximo de barras (`BarsInTrade`) del trade original.

### B. Modo `AvgBars` (Fixed Average Exposure)
*   **Comportamiento:** Evalúa el impacto de la exposición al mercado basándose únicamente en el tiempo de permanencia promedio. Ignora por completo los SL y TP de la estrategia.
*   **Lógica:**
    *   **Paso 1:** Calcular la duración promedio en barras de todos los trades del backtest original que pertenecen al periodo bajo análisis:
        $$\text{AvgBars} = \text{round}\left(\frac{1}{M}\sum_{i=1}^{M} \text{BarsInTrade}_i\right)$$
        *Donde $M$ es el número de operaciones válidas (excluyendo canceladas o depósitos).*
    *   **Paso 2:** En la simulación del mono, para cualquier operación simulada iniciada en la barra $t'$, se ignora el SL y el TP original.
    *   **Paso 3:** La operación se mantiene abierta durante exactamente $\text{AvgBars}$ barras. Se cierra al precio `Close` de la barra $t' + \text{AvgBars} - 1$.

### C. Modo `IndivBars` (Individual Trade Exposure) - *POR DEFECTO*
*   **Comportamiento:** Réplica temporal exacta trade a trade. Conserva el perfil de tiempo exacto que la estrategia estuvo en el mercado en cada operación, eliminando la influencia del SL y TP.
*   **Lógica:**
    *   Para cada operación original $i$, se lee su duración exacta en barras: $maxBars_i = \text{BarsInTrade}_i$.
    *   La operación simulada iniciada en la barra $t'$ ignora el SL y el TP original.
    *   La operación se mantiene abierta por exactamente $maxBars_i$ barras y se cierra en el precio de cierre (`Close`) de la barra $t' + maxBars_i - 1$.

---

## 4. Especificación de los Modos de Desplazamiento Circular (Shifting Mode)

En la simulación del Monkey Test, las operaciones se reubican en el tiempo de forma circular utilizando la base de datos de precios históricos cargada.

### A. Modo `Constant` (Constant Global Shift)
*   **Concepto:** Mantiene la correlación temporal y el espaciado original entre todas las operaciones. Todo el bloque de operaciones se desplaza en bloque por el mismo desfase.
*   **Algoritmo:**
    1.  Para cada una de las corridas de simulación del mono (ej. del 1 al 1000):
    2.  Se genera un **único número entero aleatorio** de desplazamiento global:
        $$Shift_{global} = \text{randomInt}(0, \text{totalBars} - 1)$$
    3.  Para cada operación $i$ en la corrida actual:
        *   Identificar el índice de la barra de apertura original: $t_{orig}$.
        *   Calcular el índice de la barra simulada $t'$ aplicando el desplazamiento de forma circular:
            $$t' = (t_{orig} + Shift_{global}) \pmod{\text{totalBars}}$$

### B. Modo `Random` (Per-Trade Random Shift) - *POR DEFECTO*
*   **Concepto:** Destruye cualquier correlación temporal entre operaciones para comprobar la robustez puramente estadística de las entradas a nivel individual.
*   **Algoritmo:**
    1.  Para cada simulación del mono (ej. del 1 al 1000):
    2.  Para cada operación $i$ de la estrategia:
        *   Se genera un **desplazamiento aleatorio independiente** para esa operación específica:
            $$Shift_{i} = \text{randomInt}(0, \text{totalBars} - 1)$$
        *   Calcular el índice de la barra simulada $t'$ aplicando el desplazamiento circular:
            $$t' = (t_{orig} + Shift_{i}) \pmod{\text{totalBars}}$$

---

## 5. Lógica Matemática del Motor de Cálculo

Al implementar esta lógica en Java, se deben incorporar las siguientes protecciones de estabilidad matemática que fueron introducidas en el Web Worker del plugin:

### 5.1 Prevención de Desbordamiento y Desplazamiento Circular (Wraparound)
Dado que los precios históricos tienen un tamaño finito (`totalBars`), al realizar el desplazamiento circular $t'$ de una operación con duración $D$ barras, puede ocurrir que el trade termine "desbordando" el final de la serie de precios.
*   **Solución:** Si la barra de cierre simulada $t'_{exit} = t' + D - 1$ excede `totalBars - 1`, se debe resolver el precio de salida de forma circular o truncar/ajustar para evitar errores de índice fuera de rango (`ArrayIndexOutOfBoundsException`).
*   **Modelo del Worker:** Para evitar que un trade quede fraccionado en el fin de semana o fuera de los límites de los datos de precio, el worker limita la barra de entrada para que el trade quepa antes del fin de los datos históricos. Si no, se ajusta circularmente en la barra de origen.

### 5.2 Estabilidad del Multiplicador de Pips (`pipMult`) contra Breakevens
En los Monkey Tests de SQX, el beneficio real de la operación se escala a la simulación mediante un multiplicador de pips (`pipMult`).
*   **Cálculo clásico:**
    $$\text{pipDiffOriginal} = |\text{ClosePrice}_{orig} - \text{OpenPrice}_{orig}|$$
    $$\text{pipMult} = \frac{\text{ProfitLoss}_{orig}}{\text{pipDiffOriginal}}$$
*   **Problema de Inestabilidad:** Si una operación se cerró a breakeven o con una diferencia de precio extremadamente pequeña, el divisor $\text{pipDiffOriginal}$ tiende a cero, haciendo que `pipMult` explote exponencialmente. En simulaciones libres (sin SL/TP) como `IndivBars` o `AvgBars`, una fluctuación normal de mercado multiplicada por este `pipMult` gigante genera saldos absurdos en los monos.
*   **Protección Obligatoria en Java:**
    Antes de dividir, verificar si la diferencia en pips original es menor a un umbral de seguridad (ej. $10^{-7}$ o $0.0000001$). Si la diferencia de precio original es insignificante, se debe forzar:
    $$\text{pipMult} = 0.0$$
    Esto evita la inestabilidad de la distribución en "U" (donde las simulaciones fallan masivamente acumulándose en los extremos del histograma debido a saldos infinitos).

---

## 6. Esquema de Caché e Integración JSON (Contrato v3)

El ResultsPlugin `DatabankMonkeyTest` es capaz de auto-cargar los resultados precalculados por el Custom Analysis si se colocan los archivos correspondientes en la carpeta:
`user/extend/ResultsPlugins/DatabankMonkeyTest/cache/`

El Custom Analysis Java debe escribir dos archivos por estrategia calculada:
1.  **CSV de Equities:** `[StrategyName]_monkey_simulation_data.csv` (formato compacto de 50 curvas).
2.  **Metadata JSON:** `[StrategyName]_monkey_simulation_data.meta.json` (formato con parámetros v3).

### 6.1 Estructura del Metadata JSON (`.meta.json`) - Contrato v3
Para soportar y validar las nuevas configuraciones, el JSON debe actualizarse con la versión de esquema `3` y guardar los parámetros con los que se realizó la simulación:

```json
{
  "schemaVersion": 3,
  "strategyName": "EURUSD_H1_Breakout",
  "period": "FULL",
  "tradeFromMs": 1451872800000,
  "tradeToMs": 1704067200000,
  "numTrades": 348,
  "numMonkeys": 1000,
  "percentile": 95.0,
  "initialBalance": 10000.0,
  "realProfit": 2450.50,
  "monkeyThreshold": 1820.00,
  "meanMonkey": 1560.20,
  "stdMonkey": 480.30,
  "zScore": 1.85,
  "rankPercentile": 92.4,
  "status": "PASSED",
  "replicationMode": "IndivBars",
  "shiftingMode": "Random",
  "meanHoldingPeriod": 14.5,
  "monkeyProfits": [
    -1200.50,
    -950.00,
    -800.10,
    "...",
    1550.00,
    1820.00,
    2980.40
  ],
  "generatedAtUtc": 1783890000000,
  "source": "CustomAnalysis"
}
```

*   **`replicationMode` y `shiftingMode`:** Indispensables para que el plugin verifique si la caché disponible corresponde a las configuraciones activas del usuario en la UI. Si cambian las opciones en el plugin, este ignorará la caché y recalculará en vivo.
*   **`monkeyProfits`:** Contiene los $N$ beneficios finales de los monos ordenados ascendentemente (requerido para construir el histograma exacto sin re-simular).
*   **`monkeyThreshold`:** El percentil exacto de aprobación (ej. Z-Score u percentil 95%) calculado a partir de la totalidad de las simulaciones.

### 6.2 Estructura del CSV (`.csv`)
Debe seguir el formato compacto "wide" introducido en la versión 2:
*   **Primera fila (Cabecera):** `monkey_id;b0;b1;b2;...;bT` (donde $T$ es el número total de transacciones originales en el backtest).
*   **Filas siguientes (≤50 curvas):**
    *   Fila `min`: La curva de balance del peor mono (menor profit).
    *   Fila `max`: La curva de balance del mejor mono (mayor profit).
    *   48 filas representativas (`q01` a `q48`): Muestras distribuidas uniformemente por percentil sobre las curvas calculadas y ordenadas de peor a mejor.

---

## 7. Directrices para la implementación en Java

1.  **Lectura del historial de precios en Java:** Utilizar las clases nativas de SQX para obtener la serie de barras de precio (`History`) asociadas al símbolo y temporalidad de la estrategia.
2.  **Identificación de barras de trades:** Mapear cada `OpenTime` de las órdenes reales a su índice de barra correspondiente en la serie histórica para tener el punto de partida $t_{orig}$ exacto.
3.  **Generación de números aleatorios:** Usar `java.util.Random` o `java.security.SecureRandom` para la generación de desplazamientos aleatorios en el bucle principal de los monos.
4.  **Escritura JSON:** Usar la biblioteca JSON integrada en StrategyQuant X para serializar de forma nativa la metadata. Asegurar que las rutas de guardado apunten directamente a:
    `user/extend/ResultsPlugins/DatabankMonkeyTest/cache/`
