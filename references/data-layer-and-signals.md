# Data Layer, Multi-Timeframe, Multi-Symbol y Signal Engine

## "Pedir datos" no es "tener datos"

`CopyRates()` y `CopyBuffer()` **no garantizan** que se reciba todo lo solicitado. Ambas devuelven el número de elementos copiados (o -1 en error) — comprueba siempre ese valor de retorno antes de asumir que el array tiene los datos esperados. Nunca proceses un array de velas/buffer sin verificar tamaño y frescura primero.

`BarsCalculated()` sobre el handle de un indicador debe verificarse antes de confiar en sus buffers — un indicador recién creado (`iMA()`, etc.) puede no tener aún suficientes barras calculadas, especialmente justo tras `OnInit()` o un cambio de símbolo/timeframe. `iMA()` (y funciones equivalentes) no "calculan una EMA" en el sentido de devolver el valor directamente — devuelven un **handle**; hay que leer el buffer con `CopyBuffer()` por separado, y ese buffer también puede fallar o venir incompleto.

## Shift 0 vs shift 1

- Shift 0 = vela en formación (aún no cerrada).
- Shift 1 = última vela cerrada.

Si la estrategia decide sobre "la vela que acaba de cerrar", usa shift 1, no shift 0 — usar shift 0 para una decisión que debería basarse en cierre es uno de los errores más comunes y más peligrosos (introduce repintado/lookahead). Detectar "nueva vela" (comparar el timestamp de shift 0/1 con el ciclo anterior) no es siempre suficiente por sí solo — combínalo con `BarsCalculated()` para los indicadores relevantes.

## El problema del shift entre timeframes (regla de oro MTF)

**Nunca asumas** que `M1 shift 1 == M5 shift 1` — los índices son relativos a cada timeframe y representan barras distintas. El tiempo es la referencia común, no el índice.

Para mapear correctamente entre timeframes, usa `iBarShift()`:

```
timestamp (de la vela M1 relevante)
   ↓
iBarShift(_Symbol, PERIOD_M5, timestamp)
   ↓
shift correspondiente en M5
```

Ejemplo: si la señal se genera al cierre de M1 10:04 (`rates_m1[1].time`), usa `iBarShift(_Symbol, PERIOD_M5, signal_time)` para obtener el shift M5 correspondiente y consultar `rates_m5[ese_shift]` — nunca inventes una relación de índices entre timeframes.

## "Time of Knowledge" — la regla que previene lookahead sutil

Para cada dato de estrategia, pregúntate: *¿en qué momento podía el EA conocer realmente este dato?*

```
DATA → TIMESTAMP → AVAILABILITY → DECISION
```

Ejemplo: un `EMA_H1` calculado sobre la vela que cierra a las 10:00 solo está *disponible* a partir de las 10:00 (o el primer tick tras ese cierre). Una decisión tomada a las 09:30 **no puede** usar ese valor "final" de H1 — si el backtest lo hace, hay lookahead aunque el índice de shift parezca correcto. Esta regla protege contra bugs de sincronización MTF mucho más sutiles que un simple error de shift.

## Market Clock, Strategy Clock, Higher-Timeframe Clock

Separar conceptualmente varios "relojes":
- **Market Clock** — cuándo llega cada tick/vela real del mercado.
- **Strategy Clock** — cuándo la estrategia realmente evalúa una decisión (puede ser solo en nueva vela del timeframe de decisión, no en cada tick).
- **Higher-Timeframe Clock** — cuándo se actualiza cada filtro de timeframe superior (ej. filtro H1 no debe recalcularse en cada tick M1 — solo cuando cierra una nueva vela H1).

No actualices un filtro de timeframe superior innecesariamente en cada tick — es coste computacional inútil y puede introducir inconsistencias sutiles de timing.

## Multi-symbol

- `SymbolSelect()` para asegurar que un símbolo secundario está disponible/visible en el Market Watch antes de pedirle datos.
- Los datos de un símbolo secundario pueden estar **desactualizados (stale)** aunque el símbolo principal reciba ticks constantemente — verifica explícitamente el timestamp de la última vela del símbolo secundario, no asumas sincronía.
- `CopyTicks()` no es simplemente "otra forma de `OnTick()`" — trae ticks históricos de un símbolo específico bajo demanda; no sustituye la necesidad de decidir cuándo reaccionar a un símbolo secundario.
- `CopyRates()` respeta un orden de memoria específico (por defecto, orden de tiempo ascendente o descendente según `ArraySetAsSeries()`) — no asumas el orden sin fijarlo explícitamente con `ArraySetAsSeries()`.
- `CopySeries()` es una herramienta alternativa/complementaria a `CopyRates()`/`CopyBuffer()` para ciertos escenarios de sincronización — evalúa cuál encaja mejor según el caso en vez de usar siempre la misma por costumbre.

## Caché de datos e indicadores

La caché existe para evitar recalcular/repedir datos innecesariamente en cada tick, no solo para "guardar variables". Reglas:
- Invalida la caché de una vela/buffer cuando cambia el `bar_time` relevante, no en cada tick indiscriminadamente.
- Cachea handles de indicadores de forma persistente (crearlos una vez en `OnInit()`, no repetidamente en `OnTick()`).
- No seas excesivamente minimalista tampoco: recalcular todo desde cero en cada tick por simplicidad puede ser aceptable para EAs pequeños, pero no escala ni es auditable para EAs complejos — sé explícito sobre la estrategia de caché elegida.

## MarketSnapshot como frontera

`CDataManager` debe producir un `MarketSnapshot` — una "fotografía" coherente y congelada de los datos relevantes para un ciclo de evaluación, no un conjunto de arrays sueltos que la estrategia consulta en momentos distintos (lo que puede producir incoherencias si el tick avanza a mitad de la evaluación). El Signal Engine consume el `MarketSnapshot`, no accede a `CopyRates`/`CopyBuffer` directamente.

## Signal Engine: de datos a decisión

Una señal **no es un `bool`**. Pipeline recomendado:

```
MarketSnapshot → Features → Setup → Trigger → Confirmation → Signal
```

- **Features** — valores extraídos del snapshot (ej. valor de EMA, ATR, distancia a un nivel).
- **Setup** — condición estructural que define el contexto operable (ej. tendencia alcista según el filtro).
- **Trigger** — evento puntual que dispara la señal dentro de ese setup (ej. cruce, ruptura).
- **Confirmation** — filtro adicional opcional que valida el trigger antes de convertirse en `Signal`.

Esta separación hace la estrategia testeable de forma aislada (puedes probar Setup, Trigger y Confirmation independientemente) y evita el "bool gigante" imposible de depurar. La definición exacta de un patrón (ej. qué constituye un engulfing) pertenece a la especificación de la estrategia, documentada explícitamente — no a una condición dispersa sin justificación en el código.

Propiedad clave: el Signal Engine debe ser **determinista** y no debería saber si está corriendo en live o en el Strategy Tester — el mismo input siempre produce la misma señal, sin importar el entorno de ejecución.

Una señal puede existir legítimamente y **no** convertirse en una operación (ej. bloqueada por Risk o Safety) — Signal ≠ TradeIntent (ver `architecture.md` y `state-idempotency-recovery.md`).