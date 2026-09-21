# Eventos, Identidad, Idempotencia, UNKNOWN, Reconciliación y Recovery

Este es el núcleo de robustez del framework. La mayoría de los bugs "misteriosos" en producción vienen de ignorar algo de este archivo.

## Modelo de eventos MQL5

- `OnInit()` — no significa "empieza a tradear". Debe ser: Validate → Initialize → Recover → Reconcile → Evaluate health → Set state. Devuelve `int`: `INIT_FAILED` provoca la descarga del EA y genera después el evento de desinicialización correspondiente — no ignores el valor de retorno como mecanismo de control de arranque.
- `OnTick()` — evento de tick, **no** de vela. Detectar "nueva vela" con `IsNewBar()`-equivalente no siempre es suficiente por sí solo (verificar también `BarsCalculated()` de los indicadores relevantes antes de confiar en sus buffers).
- `OnTimer()` — útil para trabajo periódico (reconciliación programada, housekeeping) desacoplado del flujo de ticks; `EventKillTimer()` debe usarse correctamente durante shutdown.
- `OnTrade()` — notificación amplia de cambios de trading; puede corresponder a una o varias transacciones y se produce después de las llamadas correspondientes a `OnTradeTransaction()`. No la uses como correlación uno-a-uno con una solicitud específica.
- `OnTradeTransaction()` — el evento detallado. Debe ser **pequeño y rápido** (marcar "dirty"/encolar, no ejecutar lógica de negocio pesada dentro del handler). Una sola solicitud puede generar **múltiples transacciones**, y el orden de llegada **no está garantizado** — nunca programes asumiendo una secuencia intuitiva fija.
- `OnDeinit()` — no significa "cerrar cuenta". Puede dispararse por múltiples razones del ciclo de vida (recompilación, cambio de símbolo/timeframe, remoción manual, cierre de terminal, etc.). Por defecto debe: detener timers, liberar handles/recursos, volcar diagnóstico, persistir el estado necesario, limpiar UI — **no** `CloseAllPositions()` salvo requisito explícito y justificado documentado.

## Identidad determinística (CIdentityFactory)

Identidad ≠ ticket. La cadena de identificadores:

```
SIGNAL_ID → INTENT_ID → RESERVATION_ID → REQUEST_ID → ORDER → DEAL → POSITION_IDENTIFIER
```

- **Nunca uses `TimeCurrent()`** para construir un `signal_id`/`intent_id` — no es determinístico entre reinicios ni reproducible. Usa el tiempo de la vela relevante (`bar_time`) combinado con symbol/timeframe/setup/dirección.
- Usa hashing (ej. SHA-256 conceptual) sobre los campos relevantes para producir un id compacto y determinístico; **no incluyas** campos no deterministas como `created_at` en el hash de identidad.
- **Identity vs Equivalence** son preguntas distintas: *Identity* — ¿es exactamente el mismo evento lógico? *Equivalence* — ¿es operacionalmente la misma operación aunque no comparta el mismo id (ej. dos señales distintas que producirían exposición equivalente)? Un fingerprint normalizado (no basado en `NormalizeDouble()` ingenuo de precios/volúmenes) sirve para detectar equivalencia, no identidad.
- `comment` de la orden/deal es metadata útil para diagnóstico, **nunca** la fuente de identidad — puede perderse, truncarse o ser sobrescrito por el broker/terminal.
- `request_id` ≠ `intent_id` ≠ `position_ticket` ≠ `position_identifier`. No sustituyas uno por otro.

## Idempotencia — tres niveles de defensa

1. **signal_id** — bloquea que la misma señal lógica (misma vela + mismos datos + mismos parámetros) se procese dos veces.
2. **intent_id** — bloquea que la misma intención de trading se envíe dos veces (ej. reentrancia por un segundo tick antes de que el primero termine de procesarse).
3. **Exposición equivalente (`HasEquivalentExposure()`)** — bloquea abrir una posición/orden que sería operacionalmente redundante con una ya existente, aunque provenga de una señal distinta. "Equivalente" no significa "mismo símbolo" — debe considerar dirección, volumen aproximado (con tolerancia), y contexto (setup).

Idempotencia **no significa "nunca repetir una operación"** — significa que repetir el mismo evento lógico no debe producir efectos duplicados no intencionados. La identidad temporal importa: la misma señal en una vela distinta es legítimamente una nueva operación.

## CIntentRegistry

El Registry **no es la fuente de verdad** — MT5 lo es. El Registry es un journal operacional que ayuda a decidir rápido sin tener que reconciliar en cada tick.

Estados típicos de un Intent:
```
CREATED → VALIDATED → RISK_APPROVED → SUBMITTED → IN_FLIGHT
   → PARTIAL / COMPLETED / REJECTED / UNKNOWN / CANCELLED → RECOVERED
```

- `IN_FLIGHT` y `UNKNOWN` deben **bloquear** nuevos intents equivalentes.
- `REJECTED` es distinto de `UNKNOWN`: un rechazo determinista (retcode claro) libera el bloqueo de idempotencia; `UNKNOWN` no.
- El Registry debe ser reconstruible tras un restart (ver Recovery abajo) — no depender solo de memoria en tiempo de ejecución.
- `PositionsTotal()` **no sirve** como mecanismo de idempotencia (no distingue intents en vuelo antes de que exista una posición).

## UNKNOWN → RECONCILE (la regla más importante del framework)

Cuando el resultado de una operación es incierto (timeout, desconexión durante el envío, respuesta ambigua):

```
UNKNOWN
  ↓
RECONCILE
  ↓
OBSERVE REAL STATE (positions, orders, history/deals)
  ↓
DECIDE
```

`UNKNOWN` **nunca** significa `RETRY` ciego. Durante `UNKNOWN`:
- No abrir nueva exposición equivalente.
- Mantener la reserva de riesgo asociada (no liberarla automáticamente).
- No asumir que la orden falló ni que se ejecutó.
- Reconstruir el estado consultando posiciones, órdenes y `HistorySelectByPosition()`.
- Resolver el estado del intent solo tras esa reconciliación.

`TRADE_RETCODE_TIMEOUT` debe tratarse como `UNKNOWN`, no como fallo. El peor bug posible en esta categoría es: `UNKNOWN → retry Buy()` sin reconciliar primero — riesgo grave de duplicación de exposición.

## Reconciliación (CReconciler)

Debe activarse por: startup, restart, reconnect, `UNKNOWN`, timeout, partial fill, intervención manual detectada, inconsistencia, ownership ambiguo — no solo "cada N segundos" indiscriminadamente (aunque un ciclo periódico de bajo costo vía `OnTimer()` es razonable como red de seguridad adicional).

Dos niveles:
- **Shallow** — comprobación rápida de consistencia básica (¿el número de posiciones/órdenes coincide con lo esperado?).
- **Deep** — reconstrucción completa: posiciones, órdenes, historial, ownership, riesgo, intents.

Secuencia tras reconexión:
```
RECONNECTED → DEEP RECONCILE
  → (Positions, Orders, History, Ownership, Risk, Intents)
  → COMPARE EXPECTED VS OBSERVED
  → RECOVERY POLICY
  → READY / DEGRADED / FROZEN
```

**Nunca** `reconnected → READY` directamente sin pasar por esta reconstrucción y comparación.

### CStateComparator

Compara estado esperado (derivado del Registry/última vista conocida) vs estado observado (leído en vivo de MT5) y produce un resultado estructurado — nunca comparar solo `bool position_exists`. Casos típicos:
- Esperábamos posición, no existe.
- Existe algo que no esperábamos (posible intervención externa).
- Ambas existen — comparar volumen/precio/SL/TP con tolerancia (nunca `==` directo sobre `double`; usa tolerancia basada en tick size/volume step).

El Comparator **no** cierra posiciones automáticamente — solo produce el diagnóstico; la decisión de acción la toma `CRecoveryPolicy`.

### CRecoveryPolicy — tabla de decisión (resumen)

| Esperado | Observado | Acción típica |
|---|---|---|
| NONE | NONE | Normal, sin acción |
| NONE | Posición nuestra | ADOPT (retomar gestión, no "reabrir") |
| NONE | Posición foreign | No tocar — solo registrar |
| POSITION | POSITION idéntica | Normal |
| POSITION (1.0) | POSITION (2.0) | Divergencia — investigar antes de actuar (posible intervención externa o fill parcial no reconciliado) |
| PENDING | POSITION | La pendiente se ejecutó — actualizar registry |
| PENDING | NONE | Cancelada o expirada — verificar en historial cuál |

Principio **fail closed**: ante divergencia no resuelta con confianza suficiente, bloquear (`FROZEN`/`DEGRADED`) en vez de asumir el escenario más optimista. Recovery no es lo mismo que autorizar trading — `allow_new_trades` y `allow_management` deben poder decidirse por separado tras una recuperación.

### Recovery ≠ Restore

*Restore* implicaría volver exactamente al estado anterior desde una copia guardada, confiando en ella como verdad. *Recovery* es **reconstruir** el estado real actual desde MT5 (observado), usando cualquier dato persistido solo como pista/evidencia, nunca como autoridad. `Observed State > Cached State` siempre.

### CIntentRecovery — reconstruir tras perder la RAM

Tras un restart, el Registry en memoria está vacío. Para reconstruir intents:
- Buscar el deal de entrada relevante en el historial y extraer metadata (magic, comment si existe, timing).
- Asignar un nivel de confianza (`RecoveryConfidence`: EXACT / STRONG / AMBIGUOUS / UNKNOWN) — no tratar toda recuperación como igualmente confiable.
- Nunca inventar un `intent_id` reconstruido con certeza que no se tiene realmente.

### Persistencia

Tres categorías de estado:
1. **Derivable de MT5** — no persistir (se reconstruye vía reconciliación): posiciones, órdenes, historial.
2. **Útil como evidencia pero no autoritativo** — puede persistirse (ej. metadata del Registry, contador de pérdida diaria) pero siempre subordinado al estado observado al recuperar.
3. **Realmente necesario y no derivable** — ej. parámetros de configuración runtime específicos que no viven en MT5.

Herramientas: `GlobalVariables` del terminal (persisten entre reinicios del terminal, no solo del EA — útiles para locks simples entre EAs) vs archivos (más flexibles, requieren tu propio versionado). Nunca hagas que el archivo persistido sea "más importante" que el estado real de MT5 al reconciliar — es solo una pista.

## Concurrencia y reentrancia

- No uses un simple `bool order_sent`/`bool busy` para prevenir reentrancia — no captura estados intermedios (`IN_FLIGHT`, `UNKNOWN`) correctamente.
- Ticks rápidos pueden reentrar `OnTick()` antes de que una operación previa se resuelva — la protección debe basarse en el estado real del Intent Registry, no en un flag simple.
- `OnTradeTransaction()` debe ser tolerante al orden de llegada de eventos y extremadamente rápido — patrón recomendado: marcar "dirty" y dejar que la reconciliación (en `OnTimer()` o el siguiente ciclo apropiado) haga el trabajo pesado, no procesar todo dentro del handler.
- `OrderSendAsync()` hace todavía más evidente la necesidad de este modelo: el envío y la confirmación están desacoplados en el tiempo por diseño.