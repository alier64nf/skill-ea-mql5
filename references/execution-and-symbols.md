# Ejecución, retcodes y restricciones de símbolo

## CTrade vs MqlTradeRequest

`CTrade` (Standard Library) es la capa cómoda; `MqlTradeRequest`/`OrderSend` es el nivel inferior. `CTrade` internamente construye un `MqlTradeRequest`. Usa `CTrade` cuando su comportamiento por defecto es suficiente; usa `MqlTradeRequest` directamente cuando necesitas control explícito sobre `type_filling`, `deviation`, o el flujo `OrderCheck` → `OrderSend` con interpretación propia del resultado. No mezclar ambos caminos para el mismo tipo de operación dentro del mismo EA sin una razón arquitectónica clara — debe existir un único Execution Gateway.

`action` en `MqlTradeRequest` es fundamental: `TRADE_ACTION_DEAL` (mercado), `TRADE_ACTION_PENDING`, `TRADE_ACTION_SLTP`, `TRADE_ACTION_MODIFY`, `TRADE_ACTION_REMOVE`, `TRADE_ACTION_CLOSE_BY`.

## La regla más importante de esta sección

```
OrderSend() == true
```
**NO significa** `trade executed`. Significa únicamente que la solicitud superó la comprobación básica de envío. La documentación oficial de MQL5 es explícita en esto: hay que analizar el resultado y seguir la ejecución posterior mediante las transacciones correspondientes.

```
OrderCheck() == true
```
Tampoco garantiza ejecución. Es una validación previa (fondos, margen, parámetros básicos), no una promesa de que el servidor aceptará y ejecutará la operación.

La verdad de lo que ocurrió se confirma con: `retcode` de `MqlTradeResult`, los deals/orders/positions observados, y `OnTradeTransaction`/reconciliación — nunca con el `bool` de retorno de `OrderSend`/`CTrade`.

## MqlTradeResult y retcodes

`MqlTradeResult` incluye `retcode`, `deal`, `order`, `volume`, `price`, `comment`, `request_id`, entre otros.

### Retcodes de éxito/completado
- `TRADE_RETCODE_DONE` — la solicitud fue completada.
- `TRADE_RETCODE_DONE_PARTIAL` — solo se completó parcialmente.
- `TRADE_RETCODE_PLACED` — la orden (pendiente) fue colocada.

### Rechazos deterministas
`TRADE_RETCODE_REJECT`, `TRADE_RETCODE_INVALID`, `TRADE_RETCODE_INVALID_VOLUME`, `TRADE_RETCODE_INVALID_PRICE`, `TRADE_RETCODE_INVALID_STOPS`, `TRADE_RETCODE_INVALID_FILL`, `TRADE_RETCODE_INVALID_ORDER`. Cada uno indica un problema distinto — no los trates todos como "OrderSend falló" genérico. Un sistema serio debe poder decir algo como: *"Volume 0.13 is invalid for NAS100. Allowed step = 0.10"*, no solo `Print("Trading error")`.

### Errores de mercado / ambiguos
`TRADE_RETCODE_MARKET_CLOSED`, `TRADE_RETCODE_PRICE_OFF`, `TRADE_RETCODE_PRICE_CHANGED`, `TRADE_RETCODE_REQUOTE`, `TRADE_RETCODE_NO_MONEY`, `TRADE_RETCODE_TIMEOUT` (tratar como `UNKNOWN`, no como fallo confirmado).

## OrderCheck() antes de enviar

```
TradeRequest → OrderCheck → OrderSend
```
`MqlTradeCheckResult` da información sobre fondos, margen, equity, balance y un retcode preliminar. Sigue siendo solo una comprobación previa: `CHECK ≠ EXECUTION`.

## GetLastError() no sustituye al retcode de trading

`GetLastError()` reporta errores del entorno de ejecución de MQL5, no el resultado de la operación de trading en sí. Captúralo inmediatamente después de la llamada que pueda producirlo (no más tarde, porque puede ser sobrescrito). Nunca lo uses como sustituto de `MqlTradeResult.retcode`.

## Filling modes: FOK, IOC, RETURN

Tabla de compatibilidad por modo de ejecución del símbolo (para **órdenes de mercado**):

| Execution mode | FOK | IOC | RETURN |
|---|---|---|---|
| Request | ✓ | ✓ | ✓ |
| Instant | ✓ | ✓ | ✓ |
| Market | según símbolo | según símbolo | ❌ |
| Exchange | según símbolo | según símbolo | ✓ |

Para **pending orders**, `RETURN` es la política correcta independientemente del execution mode, porque la orden no pretende ejecutarse en el momento de enviarla.

`SYMBOL_FILLING_MODE` es un **bitmask**: los valores se combinan con OR y se comprueban con AND. Nunca hagas `if(spec.filling_mode == SYMBOL_FILLING_FOK)` — usa `(spec.filling_mode & SYMBOL_FILLING_FOK) != 0`, porque puede haber varios flags simultáneos. No hardcodees el filling mode: resuélvelo desde `SYMBOL_FILLING_MODE` en una capa dedicada (`CSymbolSpec`/`CExecutionPolicy`), no inline en cada llamada de ejecución.

`deviation` (slippage tolerado en puntos) es una política de ejecución, distinta de "slippage real" (la diferencia entre precio esperado y precio realmente ejecutado, que solo se conoce después, comparando con los deals).

## Symbol constraints — nunca asumir, siempre consultar

- `SYMBOL_POINT` **≠** `SYMBOL_TRADE_TICK_SIZE`. `_Digits` tampoco valida el precio: un precio puede tener una representación decimal válida (`NormalizeDouble(price, _Digits)`) y aun así no estar en un incremento negociable válido para el símbolo. El precio debe normalizarse según `SYMBOL_TRADE_TICK_SIZE`, con cuidado con la dirección del redondeo (no uses `MathRound()` ciegamente — redondea hacia el lado seguro según BUY/SELL).
- `SYMBOL_TRADE_STOPS_LEVEL` — distancia mínima permitida entre el precio y SL/TP.
- `SYMBOL_TRADE_FREEZE_LEVEL` — distancia dentro de la cual una orden/posición no puede modificarse ni cancelarse (es un concepto distinto de stops level).
- `SYMBOL_VOLUME_MIN` / `SYMBOL_VOLUME_MAX` / `SYMBOL_VOLUME_STEP` / `SYMBOL_VOLUME_LIMIT` — el volumen debe normalizarse por step (preferir `MathFloor` sobre `MathRound` para no exceder el riesgo calculado), nunca solo comparar contra min/max sin normalizar. `SYMBOL_VOLUME_LIMIT` es un límite de volumen abierto total (posiciones + pendientes), distinto de `VOLUME_MAX` por operación, y es direccional.
- `SYMBOL_TRADE_MODE` (incluye `SYMBOL_TRADE_MODE_DISABLED`, `SYMBOL_TRADE_MODE_CLOSEONLY`, etc.) y `SYMBOL_ORDER_MODE` — qué tipo de operaciones están permitidas ahora mismo.
- Bid/Ask: BUY se ejecuta a Ask, SELL a Bid (y el SL/TP de cada dirección se mide desde el lado contrario apropiado). El precio "actual" para una decisión no es un único número — depende de la dirección de la operación.

`CSymbolSpec` (capa dedicada) debe encapsular todas estas reglas del broker y ser la única fuente para normalización de precio/volumen y validación de distancia mínima — no repitas esta lógica en cada punto de ejecución. No congeles `SYMBOL_*` para siempre: pueden cambiar en runtime (ej. cambios de sesión), así que refréscalos con la periodicidad adecuada, no solo en `OnInit()`.

## Tres niveles de validación

1. **CSymbolValidator** — reglas estáticas del broker (tick size, volume step, stops/freeze level, filling mode, order mode).
2. **OrderCheck()** — validación dinámica del servidor (fondos, margen) antes de enviar.
3. **Interpretación del retcode tras OrderSend()** — la verdad final de lo que el servidor decidió.

Estos tres niveles son complementarios, no sustitutos entre sí.

## Spread, slippage y calidad de ejecución

Registra spread al momento de la señal y al momento de ejecución, deviation configurado vs slippage real observado, y tiempo de ejecución. Esto permite auditar después si el EA "mágico" en backtest simplemente ignoraba condiciones reales de mercado (ver `testing-validation.md`, modos de modelado).

## Margin, equity y exposición

`OrderCalcMargin()` y `OrderCalcProfit()` son las herramientas correctas para calcular margen requerido y riesgo monetario respectivamente — nunca una fórmula genérica de "precio × lotaje × constante" sin pasar por estas funciones, porque el valor por punto varía por instrumento (`SYMBOL_TRADE_CALC_MODE`). Riesgo, margen y exposición son tres dimensiones distintas (ver `risk-safety-circuit-breakers.md`): margen disponible no implica que el riesgo monetario planeado sea aceptable, y viceversa.