# Anti-patrones — lo que esta skill nunca debe hacer ni recomendar

## Sobre MQL5 como plataforma
- Inventar APIs de MQL5 que no existen o cuyo comportamiento no está verificado.
- Asumir que MQL5 funciona exactamente como C++ (comparte sintaxis, no toda la semántica ni la biblioteca estándar).
- Afirmar comportamiento específico de la plataforma sin verificarlo cuando hay incertidumbre — prioriza la documentación oficial (https://www.mql5.com/en/docs/) si tienes acceso a búsqueda.

## Sobre ejecución
- Asumir que `OrderSend() == true` significa "operación ejecutada".
- Asumir que `OrderCheck() == true` garantiza ejecución.
- Asumir que una orden genera un único evento de transacción.
- Usar `GetLastError()` como sustituto de `MqlTradeResult.retcode`.
- Hardcodear el filling mode en vez de resolverlo desde `SYMBOL_FILLING_MODE` (bitmask).
- Comparar `filling_mode == SYMBOL_FILLING_FOK` en vez de usar AND bit a bit.
- Usar `==` para comparar precios o volúmenes (`double`) sin tolerancia basada en tick size/volume step.
- Normalizar precio solo con `NormalizeDouble(price, _Digits)` sin considerar `SYMBOL_TRADE_TICK_SIZE`.
- Normalizar volumen solo comparando MIN/MAX sin respetar `SYMBOL_VOLUME_STEP`.
- Ignorar `SYMBOL_TRADE_STOPS_LEVEL`/`SYMBOL_TRADE_FREEZE_LEVEL`.

## Sobre arquitectura
- Que Strategy envíe órdenes directamente.
- Que Risk ejecute operaciones.
- Que Reconciler abra posiciones.
- Múltiples módulos con caminos independientes hacia `OrderSend`/`CTrade` (más de una "puerta de ejecución").
- Meter toda la lógica de negocio dentro de `OnTick()`.
- Meter lógica de negocio pesada dentro de `OnTradeTransaction()` (debe ser pequeño y rápido).
- Sustituir el estado real de MT5 por variables internas cacheadas sin reconciliación.
- Confundir una respuesta booleana de una función con "ejecución final" confirmada.
- Crear una arquitectura monolítica innecesaria para un EA simple, o el extremo contrario: 50 archivos desde el primer día para un EA trivial.

## Sobre netting/hedging/ownership
- Asumir que un símbolo identifica una única posición (falso en hedging).
- Asumir que `Magic Number` identifica ownership completo.
- Usar `PositionSelect(_Symbol)` para operaciones que deberían afectar a un ticket específico en hedging.
- Confundir `POSITION_TICKET` con `POSITION_IDENTIFIER`.
- Convertir ownership `UNKNOWN` en `FOREIGN` o en `NONE` por defecto.
- Implementar lógica de netting asumiendo semántica de hedging, o viceversa, sin verificar el modo real de la cuenta.
- Tratar el `comment` de una orden/deal como fuente de identidad u ownership.

## Sobre riesgo y seguridad
- Calcular lotaje con una fórmula genérica en vez de `OrderCalcProfit()`/`OrderCalcMargin()`.
- Confundir riesgo con margen, o riesgo individual con exposición agregada de cartera.
- Tratar riesgo desconocido (`unknown_risk`) como cero.
- Reservar presupuesto de riesgo después de ejecutar, en vez de antes.
- Liberar una reserva de riesgo automáticamente ante un resultado `UNKNOWN`.
- Modelar el estado de seguridad como un único `bool` en vez de razones explícitas y permisos diferenciados (`allow_new_entries` vs `allow_management`, etc.).
- Circuit breakers con un solo umbral (sin histéresis/latching) que pueden oscilar.
- Creer que `ExpertRemove()` es un Kill Switch de trading.
- Cerrar posiciones en `OnDeinit()` sin justificación explícita.

## Sobre estado, idempotencia y recuperación
- Usar `TimeCurrent()` para construir identificadores deterministas (`signal_id`/`intent_id`).
- Usar `PositionsTotal()` como mecanismo de idempotencia.
- Usar un simple `bool order_sent`/`bool busy` para prevenir reentrancia.
- Ante `UNKNOWN`, hacer retry ciego en vez de reconciliar primero.
- Ir de `reconnected` directo a `READY` sin reconciliación profunda.
- Comparar estado esperado vs observado solo con un booleano (`position_exists`) en vez de una comparación estructurada con tolerancia.
- Tratar el estado persistido en archivo/Global Variable como más autoritativo que el estado observado en MT5.
- Múltiples componentes escribiendo el estado global del sistema sin una única autoridad.

## Sobre datos
- Confundir shift 0 (vela en formación) con vela cerrada.
- Asumir que shift 1 de un timeframe es comparable a shift 1 de otro timeframe sin mapear por tiempo.
- No comprobar el valor de retorno de `CopyRates()`/`CopyBuffer()` (pueden devolver menos datos de los solicitados).
- No verificar `BarsCalculated()` antes de confiar en buffers de indicadores.
- Ignorar el problema de sincronización multi-símbolo (datos stale de un símbolo secundario).

## Sobre testing y validación
- Asumir que backtest rentable = EA robusto.
- Asumir que optimización = validación.
- Volver a ajustar parámetros usando datos Out-of-Sample (contamina la validación).
- Buscar el pico de una métrica en vez de una meseta estable al optimizar.
- Probar una estrategia intrabar con un modo de modelado insuficiente (ej. Open prices only).
- Cambiar la lógica financiera del EA entre Tester y Live.
- Interpretar percentiles de Monte Carlo (ej. P95) como garantías del futuro.
- Confundir Monte Carlo de trades (shuffle/bootstrap) con simulación de nuevas trayectorias de mercado.
- Saltar directamente de "backtest positivo" a Live sin Demo ni auditoría.

## Sobre comunicación y código generado
- Explicar sintaxis elemental de programación o conceptos básicos de trading al usuario de esta skill.
- Decir "esto está mal" sin explicar el escenario, el impacto y la corrección concreta.
- Afirmar en un comentario que algo "funciona" cuando el código no lo garantiza estructuralmente.
- Asumir que el código es correcto porque compila o porque el backtest dio ganancia.