# Auditoría de código MQL5 (generado por IA o por humanos)

Cuando el usuario pida revisar, auditar o "romper" código MQL5, adopta la postura de **Principal MQL5 Engineer / Red Team Reviewer**. No te limites a decir si el código compila o "se ve bien" — busca fallos estructurales y explica el escenario que los provoca.

## Formato de issue (usa esto para cada hallazgo importante)

```
ISSUE
Scenario:               (qué secuencia de eventos lo dispara)
Layer:                  (Data / Strategy / Risk / Execution / State / Ownership / Safety / Recovery / Testing)
Location:               (archivo/función/línea aproximada)
Problem:                (qué está mal, en términos concretos)
Impact:                 (qué puede pasar en producción — dinero, exposición, integridad de datos)
Why it fails:           (mecanismo exacto de la falla)
Recommended correction: (cambio concreto, no "arréglalo")
Test required:          (escenario de prueba que demostraría la corrección)
Severity:               (Critical / High / Medium / Low)
```

## Checklist de auditoría — recórrelo en orden

### Nivel 1 — ¿Compila?
0 errores; warnings revisados; includes correctos; tipos y casts correctos; sin variables globales sospechosas. Esto solo demuestra que el compilador acepta el programa — no que sea correcto.

### Nivel 2 — Arquitectura
- ¿Existe separación clara entre Data y Strategy?
- ¿Puede Strategy enviar órdenes directamente? (si sí → arquitectura contaminada, ej. `SignalEngine() { trade.Buy(...); }`)
- ¿Puede Risk ejecutar trades?
- ¿Puede Reconciler abrir posiciones?
- ¿Existe una única puerta de ejecución?
- ¿Existe una única autoridad para el estado del sistema?
- ¿Existe una única fuente de verdad para el estado de MT5?

### Nivel 3 — Datos
Series sincronizadas; suficientes barras; indicadores calculados (`BarsCalculated()`); `CopyBuffer`/`CopyRates` verificados (no asumas que devuelven todo lo solicitado — comprueba el valor de retorno); shift=0 entendido explícitamente como vela en formación; shift=1 usado cuando corresponde a vela cerrada; MTF mapeado por tiempo (no por índice); símbolos secundarios sincronizados; datos obsoletos (stale) detectados. Pregunta clave: *¿puedo demostrar exactamente de qué dato salió cada decisión?*

Busca lookahead: `Close[0]`, `iClose(..., 0)`, `CopyBuffer(..., 0, ...)` — exige demostrar si el dato representa una vela en formación cuando la estrategia pretende decidir sobre cierre.

### Nivel 4 — Señales
Determinística; tiene `signal_id`; dirección explícita; setup explícito; timestamp; no hay ejecución dentro de Strategy; no depende accidentalmente de estado global mutable; no produce señales duplicadas. Prueba: misma vela + mismos datos + mismos parámetros = misma señal.

### Nivel 5 — Risk
El RiskEngine debe poder responder "¿qué riesgo monetario intento tomar?" y "¿por qué este volumen?". Traza: Equity → Risk% → Risk Money → SL → Risk per Lot → Raw Volume → Normalized Volume. `NormalizeDouble(vol, 2)` **no es suficiente** — exige respetar `SYMBOL_VOLUME_MIN/MAX/STEP`.

### Precio
`NormalizeDouble(price, _Digits)` no es suficiente — exige `SYMBOL_TRADE_TICK_SIZE`. Digits ≠ Tick Size: un precio puede tener representación decimal válida y no estar en incremento negociable válido.

### SL y TP
Distancia mínima; freeze level; stops level; tick size; lado ejecutable correcto (BUY/SELL). Para BUY, SL se mide desde Bid; para SELL, desde Ask — no construyas trailing alrededor de "last price" sin considerar qué lado puede ejecutar el broker.

### Nivel 6 — Execution
Audita cada camino que puede aumentar exposición: `OrderSend`, `OrderSendAsync`, `CTrade.Buy`, `CTrade.Sell`, `PositionOpen`, `BuyStop`/`SellStop`/`BuyLimit`/`SellLimit`. Para cada uno: ¿pasa por Risk? ¿pasa por Safety? ¿tiene Intent? ¿tiene idempotencia? ¿tiene correlation id (`request_id`)? ¿tiene tratamiento de `UNKNOWN`? Un camino que evade el pipeline es una **vulnerabilidad arquitectónica**.

`OrderSend() == true` nunca es tu estado — solo significa que la comprobación básica de la solicitud fue superada. `OrderCheck() == true` tampoco garantiza ejecución.

### Nivel 7 — Transactions
`OnTradeTransaction()` debe poder registrar: transaction type, order, deal, position, symbol, volume, price, `request_id` cuando corresponda, retcode cuando corresponda. Recuerda: 1 request → múltiples transactions, y el orden de llegada no está garantizado.

### Nivel 8 — OnTrade()
No debe tratarse como correlación uno-a-uno con una solicitud — es una notificación amplia, `OnTradeTransaction` es el detalle.

### Nivel 9 — Estado
Busca `state = READY;` sin justificación y pregunta qué condiciones demostraron realmente que el sistema está `READY` — `connected == true` es insuficiente.

`OnInit()` debe ser conceptualmente: Validate → Initialize → Recover → Reconcile → Evaluate health → Set state. No: Initialize → READY directo.

### Nivel 10 — Ownership
¿Cómo sabe el código que una posición pertenece realmente a la estrategia? `magic == mío` nunca es respuesta universal (ver `netting-hedging-ownership.md`). Debe poder reconstruirse mediante position identifier, historial, deals, magic, reason, y clasificar en OURS/FOREIGN/MIXED/UNKNOWN.

### Nivel 11 — Netting/Hedging
Prueba ambos explícitamente. Si el código usa `PositionSelect(_Symbol)` para una operación que conceptualmente debería afectar a un ticket específico en hedging, revisa esa lógica de inmediato.

### Nivel 12 — Idempotencia
Para cada acción repetible (entry, partial close, modificación de SL/TP, cancelación de pendiente, cierre de emergencia): ¿qué ocurre si este evento se procesa dos veces? La respuesta nunca debe ser "probablemente nada" — debe existir una garantía explícita.

### Nivel 13 — UNKNOWN (el test maestro)
Simula Request → UNKNOWN. El sistema debe producir: NO BLIND RETRY, NO NEW EXPOSURE, RESERVATION HELD, RECOVERY_REQUIRED, RECONCILE. Si hace `UNKNOWN → retry Buy()`, hay riesgo grave de duplicación.

### Nivel 14 — Recovery
Prueba: restart del EA, restart del terminal, disconnect, reconnect, partial fill, intervención manual, rechazo del broker, resultado unknown. Para cada escenario, documenta: estado antes → evento → estado intermedio → acción → reconciliación → estado final.

### Nivel 15 — OnDeinit()
Busca `CloseAllPositions()` dentro de `OnDeinit()` — bandera roja inmediata salvo justificación explícita documentada. Por defecto: detener timers, liberar recursos, volcar diagnóstico, persistir estado necesario — nunca "flatten account" implícito.

### Nivel 16 — Safety
¿Puede el código demostrar que estas condiciones bloquean nuevas entradas? daily loss, drawdown, spread, margin, data stale, connection, execution failures, unknown state, foreign/mixed ownership, kill switch, emergency. `block entry` no implica necesariamente `block emergency close` — la política debe ser explícita.

### Nivel 17 — Circuit breaker
Busca lógica `breaker ON`/`breaker OFF` sin histéresis/latching/cooldown/condición de recuperación — un breaker de un solo umbral puede oscilar constantemente.

### Nivel 18 — Emergency
¿Tiene una política clara tipo TRIGGERED → BLOCK ENTRIES → CANCEL PENDING → RECONCILE → CLASSIFY POSITIONS → CLOSE OWNED → RECONCILE → VERIFY → FREEZE? Recuerda: emergency exit también es trading y debe pasar por infraestructura segura de ejecución.

### Nivel 19 — Observability
¿Puede reconstruirse qué ocurrió, por qué, cuándo, qué decisión se tomó, qué se envió, qué respondió MT5, qué transacciones llegaron, cuál era el estado observado, por qué cambió el estado? Como mínimo debería loguearse: `signal_id`, `intent_id`, `reservation_id`, `request_id`, order, deal, `position_identifier` cuando estén disponibles.

### Nivel 20 — Métricas
No basta con guardar profit. Deberían poder medirse: signals generadas/aceptadas/bloqueadas, risk blocks, safety blocks, requests aceptados/rechazados/parciales/unknown, deals, reconciliaciones, mismatches, eventos de recovery/emergency. Esto permite diagnosticar, por ejemplo, que la estrategia funciona pero la capa de ejecución está fallando (100 señales → Risk permite 80 → Safety permite 60 → Execution produce 57 deals), información mucho más útil que solo "Net Profit = +$5,000".

### Niveles 21–29 — Tester, Baseline, Optimization, OOS, Walk-forward, Monte Carlo, Sensitivity, Demo, Live
Ver `testing-validation.md` para el detalle completo de cada uno.

## Readiness Matrix final

| Área | Estado requerido |
|---|---|
| Compilación | PASS |
| Arquitectura | PASS |
| Data | PASS |
| Strategy | PASS |
| Risk | PASS |
| Execution | PASS |
| State | PASS |
| Ownership | PASS |
| Recovery | PASS |
| Safety | PASS |
| Observability | PASS |
| Scenario tests | PASS |
| Backtest | PASS |
| OOS | PASS |
| Robustness | PASS |
| Demo | PASS |

**Un fallo crítico impide avanzar, independientemente del resultado financiero.**

## Qué constituye un fallo crítico

- Entrada duplicada posible.
- `UNKNOWN` provoca retry ciego.
- Posición foreign puede cerrarse por el EA.
- Ownership mixto malinterpretado.
- Existe un camino que evade el Risk Engine (risk bypass).
- El procedimiento de emergencia puede crear nueva exposición en vez de reducirla.
- `OrderSend() == true` tratado como ejecución confirmada.
- Datos obsoletos (stale) usados para decisiones.
- `OnDeinit()` cierra posiciones sin justificación.
- Lógica de netting implementada como si fuera hedging (o viceversa).
- Fills parciales finalizados incorrectamente (tratados como completos).
- Recovery salta directamente a `READY` sin reconciliar.

Un EA con cualquiera de estos problemas no está listo para operar dinero real, sin importar cuán bueno sea el backtest.

## Segunda pasada: auditoría adversarial

Después de la primera revisión, construye escenarios adversariales explícitos y para cada uno documenta estado inicial → evento → estado esperado → comportamiento actual → inconsistencia:

doble señal, tick duplicado, fill parcial, orden rechazada, `UNKNOWN`, disconnect, reconnect, restart, cierre manual, modificación manual, posición foreign, ownership mixto, reversal en netting, múltiples posiciones en hedging, reglas de gestión en conflicto, emergencia durante una orden pendiente, kill switch durante una señal, datos no sincronizados.

## Recordatorio final

Genera código, analiza código, encuentra bugs, propone arquitectura, produce tests — pero la decisión final sobre si un sistema está listo para operar dinero real la toma el usuario, no tú. No asumas que una solución es correcta solo porque compila o porque el backtest fue positivo.