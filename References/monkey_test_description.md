# Monkey Test: Descripción Científica e Implementación en QuantLab

Este documento describe el funcionamiento, lógica matemática y criterios del **Monkey Test** tal como está implementado en la aplicación de rule extraction **QuantLab** (versión 3.1).

---

## 1. Concepto y Objetivos

El **Monkey Test** es una prueba de robustez estadística diseñada para responder a la siguiente pregunta: 
> *¿El rendimiento histórico obtenido por una regla es fruto de una ventaja real (Edge) o podría haberse obtenido simplemente operando al azar con la misma frecuencia y duración?*

En trading cuantitativo, un error común consiste en evaluar reglas basadas puras en optimización que, por pura probabilidad o "suerte", encuentran rachas ganadoras en el histórico. Para descartar la suerte, el Monkey Test simula cientos de "monos" que operan de forma aleatoria, estableciendo una distribución de probabilidad de resultados aleatorios para compararla con el rendimiento real.

---

## 2. Nomenclatura en el Código

En el código de la aplicación, el test se implementa bajo dos componentes principales:
* **Filtro de Robustez (Algoritmo):** La función [vectorized_monkey_test_ultra](file:///C:/Users/jorge/Downloads/rule_extractor-main-v3.1%20(1)/rule_extractor-main/unified_rule_miner.py#L717-L731) en `unified_rule_miner.py`.
* **Métrica de Puntuación (Z-Score):** La variable `Monkey_Score` calculada dinámicamente en la función [evaluate_period](file:///C:/Users/jorge/Downloads/rule_extractor-main-v3.1%20(1)/rule_extractor-main/unified_rule_miner.py#L1220-L1256) y mostrada en la tabla visual de `app_dashboard.py`.

---

## 3. Lógica del Algoritmo: Desplazamiento Circular (Circular Shift)

A diferencia de otros métodos que generan operaciones aleatorias (como lanzar una moneda al aire en cada vela), el Monkey Test de esta aplicación utiliza la técnica de **desplazamiento circular (Circular Shifting / Time-Shift Permutation)** sobre el vector de señales.

### Implementación Matemática en Python

El código vectorizado en [unified_rule_miner.py](file:///C:/Users/jorge/Downloads/rule_extractor-main-v3.1%20(1)/rule_extractor-main/unified_rule_miner.py#L717-L731) es el siguiente:

```python
def vectorized_monkey_test_ultra(signals, returns, n_sims=100, percentile=95):
    """MONKEY TEST ULTRA VECTORIZADO"""
    real_return = np.sum(returns[signals])
    n_bars = len(returns)
    if n_bars < 10 or np.sum(signals) == 0: return False, 0
    
    # 1. Generación de desplazamientos aleatorios independientes para cada simulación
    shifts = np.random.randint(1, n_bars, size=n_sims)
    
    # 2. Creación de una malla de índices base
    indices = np.arange(n_bars)
    
    # 3. Aplicación del desplazamiento circular mediante broadcasting y el operador módulo (%)
    shifted_indices = (indices[None, :] - shifts[:, None]) % n_bars
    
    # 4. Re-mapeo del vector de señales originales en los nuevos índices temporales
    signals_matrix = signals[shifted_indices]
    
    # 5. Cálculo del beneficio acumulado de cada mono
    monkey_returns = (signals_matrix * returns[None, :]).sum(axis=1)
    
    # 6. Cálculo del percentil de control
    threshold_val = np.percentile(monkey_returns, percentile)
    
    # Retorna si el beneficio real supera el umbral y el valor del umbral
    return real_return > threshold_val, threshold_val
```

### Paso a Paso del Funcionamiento:
1. **Rendimiento Real:** Se calcula sumando los retornos del activo (`returns`) cuando la señal de la regla está activa (`signals`).
2. **Generación de Desplazamientos:** Para cada simulación de mono ($N$ sims), se elige un desfase aleatorio entero (`shifts`) entre $1$ y la longitud total del histórico (`n_bars`).
3. **Desplazamiento Circular:** Mediante la operación matemática `(indices - shift) % n_bars`, las posiciones de las señales se desplazan uniformemente. Si una señal se desplaza hacia atrás del principio (índice $< 0$), el operador módulo la devuelve circularmente al final del dataset.
4. **Matriz de Monos:** `signals_matrix` es una matriz bidimensional de tamaño `(n_sims, n_bars)` donde cada fila es una línea temporal de señales idéntica a la original, pero desplazada aleatoriamente en el tiempo.
5. **Evaluación de Resultados:** Se multiplican las señales de cada mono por los retornos reales y se calcula el rendimiento final de cada uno.

---

## 4. Reglas de Modificación de las Entradas (Trades)

La simulación de los "monos" sigue tres reglas de restricción estrictas que aseguran la validez científica del test:

* **Regla 1: Preservación de la Frecuencia (Número de Trades)**
  Cada simulación de mono realiza **exactamente la misma cantidad de operaciones** que la estrategia real. No se evalúan monos hiperactivos ni monos pasivos, lo que garantiza una comparación justa (manzanas con manzanas).
  
* **Regla 2: Preservación de la Distribución y Agrupación (Clustering)**
  Al desplazar el vector completo, se mantiene la estructura temporal de las operaciones. Si la regla tiende a operar en ráfagas (por ejemplo, 5 operaciones seguidas y luego un largo periodo de inactividad), los monos harán lo mismo. Esto preserva las propiedades de autocorrelación del sistema.

* **Regla 3: Preservación del Tiempo en Mercado (Duración/Exposición)**
  Dado que el vector `returns` ya asume la duración del trade precalculada (`exposicion_dias`), cada operación de cada mono dura exactamente el mismo número de velas que el trade de la regla original.

---

## 5. Criterios de Selección y Métricas

La aplicación de dashboard expone e implementa dos formas de evaluar las reglas contra la simulación:

### A. Filtro de Control de Calidad (Criterio Binario)
Una regla solo se considera robusta (`Monkey_OK = True`) si su rentabilidad supera el umbral matemático del percentil configurado. 
* **Configuración Endurecida (v3.1):** 
  * `n_monkey_sims = 500` (Número de monos simulados).
  * `monkey_percentile = 99.9` (Percentil de exigencia). Esto significa que la regla real debe **batir al 99.9% de los monos** para ser aceptada. Equivale a un nivel de significación estadística del $\alpha = 0.1\%$ en un test de una cola.

### B. Métrica de Desviación (`Monkey_Score`)
El `Monkey_Score` representa la distancia en desviaciones estándar ($\sigma$) entre la regla y la masa aleatoria. Se calcula como un Z-score:

$$\text{Monkey Score} = \frac{\text{Beneficio Real} - \mu_{\text{monos}}}{\sigma_{\text{monos}}}$$

Donde:
* $\mu_{\text{monos}}$ es la media de los beneficios de todos los monos.
* $\sigma_{\text{monos}}$ es la desviación estándar de los beneficios de los monos.

* **Interpretación práctica en QuantLab:**
  * `Monkey_Score < 1.0` $\rightarrow$ La regla se comporta como el azar.
  * `Monkey_Score` entre `1.0` y `1.5` $\rightarrow$ Zonas de incertidumbre.
  * `Monkey_Score > 1.5` o `2.0` $\rightarrow$ La regla tiene un Edge indudable y sistemático sobre el comportamiento aleatorio.

---

## 6. Extensión Teórica: Monkey Test con SL y TP Dinámicos (Porcentuales)

### Concepto y Motivación
En sistemas de trading que emplean órdenes de salida por límite de pérdidas (Stop Loss, SL) y toma de beneficios (Take Profit, TP), evaluar el test del mono simplemente sumando retornos prefijados (como se hace con la duración fija por holding period) no es suficiente, ya que el momento en el que se cierran las operaciones depende del camino recorrido por el precio (Path Dependency).

Además, al operar activos con fuertes tendencias a largo plazo (por ejemplo, el Oro, Bitcoin o índices bursátiles), los precios absolutos cambian drásticamente entre el pasado lejano y el presente. Si calculamos el SL y TP en valores absolutos (como dólares o pips fijos), el perfil de riesgo se distorsiona: una distancia fija de $10 representa un riesgo del 1.0% cuando el precio está a $1000, pero solo del 0.5% cuando cotiza a $2000.

Para resolver esto, proponemos un **Monkey Test con SL y TP Relativos en Porcentaje**, modificando el timing de entrada de las operaciones mediante desplazamiento circular pero preservando la distancia del SL y del TP de dichos trades de manera relativa al precio de entrada del mono.

---

### Algoritmo e Ideación del Funcionamiento

#### Fase 1: Parametrización de los Trades Originales
Para cada trade $i$ ejecutado por la estrategia original en su respectivo índice de vela de entrada $t_i$, extraemos:
1. **Dirección ($D_i$):** $+1$ para Compras (Long) y $-1$ para Ventas (Short).
2. **Distancia Relativa del Stop Loss ($SL_{\%, i}$):**
   $$SL_{\%, i} = \frac{|SL_i - P_{\text{entry}, i}|}{P_{\text{entry}, i}}$$
3. **Distancia Relativa del Take Profit ($TP_{\%, i}$):**
   $$TP_{\%, i} = \frac{|TP_i - P_{\text{entry}, i}|}{P_{\text{entry}, i}}$$
4. **Duración Máxima de Exposición ($MaxBars_i$):** Número máximo de velas que estuvo abierto el trade original si no tocó SL ni TP.

#### Fase 2: Simulación del Mono por Desplazamiento Circular
Para cada una de las simulaciones ($N$ monos), aplicamos un desplazamiento circular de las señales de entrada:
$$t'_i = (t_i - Shift) \pmod{N_{\text{velas}}}$$

#### Fase 3: Evaluación Dinámica de la Ruta (Path Evaluation)
Para cada operación desplazada al índice de entrada del mono $t'_i$, evaluamos los precios históricos reales hacia adelante ($High$, $Low$, $Close$):
1. **Precio de Entrada del Mono ($P'_{t'_i}$):** El precio de apertura o cierre en la vela $t'_i$.
2. **Establecimiento de Niveles Nominales Dinámicos:**
   * **Para Long ($D_i = +1$):**
     $$SL'_{nominal} = P'_{t'_i} \times (1 - SL_{\%, i})$$
     $$TP'_{nominal} = P'_{t'_i} \times (1 + TP_{\%, i})$$
   * **Para Short ($D_i = -1$):**
     $$SL'_{nominal} = P'_{t'_i} \times (1 + SL_{\%, i})$$
     $$TP'_{nominal} = P'_{t'_i} \times (1 - TP_{\%, i})$$
3. **Simulación de la Operación (Paso a paso):**
   Se avanza barra a barra desde la vela $t'_i$ hasta un límite de $MaxBars_i$ velas:
   * Si la barra $t$ cruza o toca $SL'_{nominal}$, la operación se cierra con pérdida del $-SL_{\%, i}$ (más costes de spread/deslizamiento).
   * Si la barra $t$ cruza o toca $TP'_{nominal}$, la operación se cierra con ganancia del $+TP_{\%, i}$.
   * Si se alcanzan las $MaxBars_i$ velas sin tocar ninguno de los dos límites, la operación se cierra a mercado en el precio de cierre de esa vela de salida, calculando la rentabilidad porcentual de salida:
     $$Return_{\%} = D_i \times \frac{P'_{exit} - P'_{t'_i}}{P'_{t'_i}}$$

---

### Pseudocódigo de Simulación en Python

```python
def evaluate_monkey_trade_with_sl_tp(df, entry_idx, direction, sl_pct, tp_pct, max_bars):
    n_bars = len(df)
    entry_price = df['Close'].iloc[entry_idx]
    
    # Calcular límites nominales dinámicos basados en porcentajes relativos
    if direction == 1:  # Long
        sl_price = entry_price * (1.0 - sl_pct)
        tp_price = entry_price * (1.0 + tp_pct)
    else:  # Short
        sl_price = entry_price * (1.0 + sl_pct)
        tp_price = entry_price * (1.0 - tp_pct)
        
    # Evaluar paso a paso en el histórico
    for bar in range(1, max_bars + 1):
        idx = (entry_idx + bar) % n_bars
        high = df['High'].iloc[idx]
        low = df['Low'].iloc[idx]
        close = df['Close'].iloc[idx]
        
        if direction == 1:  # Long
            if low <= sl_price:
                return -sl_pct  # Toca Stop Loss
            if high >= tp_price:
                return tp_pct   # Toca Take Profit
        else:  # Short
            if high >= sl_price:
                return -sl_pct  # Toca Stop Loss
            if low <= tp_price:
                return tp_pct   # Toca Take Profit
                
    # Salida por límite de tiempo (max_bars)
    exit_price = df['Close'].iloc[(entry_idx + max_bars) % n_bars]
    return direction * (exit_price - entry_price) / entry_price
```

### Ventajas Científicas de este Enfoque
1. **Neutralidad de la Tendencia:** Al ajustar el SL y el TP de forma porcentual, las operaciones en zonas de precios altos o bajos arriesgan exactamente la misma proporción relativa de capital, eliminando el sesgo tendencial.
2. **Evaluación de la Dependencia del Camino (Path Dependency):** Captura de forma fidedigna si la estrategia tiene "Edge" en la precisión de sus salidas de seguridad (SL) y de beneficios (TP), y no solo en la dirección general.
3. **Consistencia de la Relación Riesgo-Beneficio (R:R Ratio):** Al mantener el ratio porcentual de la estrategia original para cada mono, los resultados agregados preservan la matemática del sistema original ante diferentes volatilidades.

