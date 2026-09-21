# Risk Engine, Exposure, Circuit Breakers, Kill Switch y Emergency

## Risk ≠ Margin ≠ Volume ≠ Exposure

Cuatro dimensiones distintas que no deben confundirse:
- **Risk** (dinero que se está dispuesto a perder si el SL se ejecuta).
- **Margin** (capital reservado por el broker para mantener la posición abierta) — calculado con `OrderCalcMargin()`.
- **Volume** (lotaje) — resultado, no punto de partida.
- **Exposure** (magnitud total de mercado bajo riesgo, agregada por símbolo, dirección o grupo).

## Risk Engine — el pipeline correcto

```
Equity → Risk % → Risk Money → SL (distancia) → Risk per Lot → Raw Volume → Normalized Volume
```

- Usa `OrderCalcProfit()` para obtener el valor monetario real por lote dado un SL, en vez de una fórmula genérica de "precio × lotaje × constante" (el valor por punto varía por instrumento — crítico en índices como NAS100/USA30).
- Normaliza el volumen con `SYMBOL_VOLUME_STEP`, respetando `SYMBOL_VOLUME_MIN`/`MAX`/`LIMIT`. Prefiere redondear hacia abajo (`MathFloor`) sobre el step en vez de `MathRound`, para no exceder el riesgo objetivo.
- Si el volumen mínimo permitido ya arriesga más de lo aceptado por el usuario, el Risk Engine debe **rechazar** la operación explícitamente (con razón), no silenciosamente subir el riesgo.
- Usa `OrderCalcMargin()` como control adicional independiente — puede que haya presupuesto de riesgo pero no margen suficiente, o viceversa.
- El Risk Engine debe devolver un **objeto de decisión** (`RiskDecision`), no solo un `double lot`, para que el sistema pueda explicar por qué SÍ o por qué NO opera.

## Exposure y Portfolio Risk

Riesgo de una operación individual ≠ riesgo agregado de cartera. Un `CExposureManager`/`CPortfolioRiskManager` debe:

- Escanear posiciones abiertas y órdenes pendientes (ambas cuentan para exposición).
- Separar `known_risk` (posiciones con SL definido, ownership resuelto) de `unknown_risk` (posiciones sin SL, o con ownership no resuelto). **Nunca trates riesgo desconocido como cero.**
- Distinguir **gross exposure** (suma de volumen en valor absoluto) de **net exposure** (suma neta con signo) — netting y hedging cambian radicalmente cuál es relevante.
- Agrupar por `Risk Group` cuando símbolos están correlacionados (ej. varios pares de un mismo commodity, o índices relacionados) — el riesgo agregado por grupo puede superar límites aunque cada símbolo individualmente esté dentro de su límite. No inventar una matriz de correlación compleja si no es necesario — empezar con grupos explícitos configurados es suficiente en la mayoría de los casos.
- `SYMBOL_VOLUME_LIMIT` es un límite de volumen **por dirección** — no confundir con el límite de riesgo de la propia estrategia.

### Risk Reservation

Para evitar que dos señales casi simultáneas gasten el mismo presupuesto de riesgo antes de que la primera se confirme ejecutada:

1. **Reservar antes de ejecutar** — nunca reservar después de que la orden ya se envió.
2. La reserva **no es una posición real** ni sustituye al riesgo real una vez confirmado por reconciliación.
3. Estados de una reserva: creada → consumida (tras confirmación) → liberada (si la orden fue rechazada/cancelada) → expirada (timeout) → **retenida ante UNKNOWN** — nunca liberar una reserva automáticamente ante un resultado `UNKNOWN`, porque la exposición real podría haberse creado igualmente.
4. `Reserve()` debe ser lo más atómico posible; considera locks entre múltiples EAs de la misma cuenta si aplica (`GlobalVariableSetOnCondition()` es una herramienta relevante para coordinación simple entre procesos MT5).

## Circuit Breakers

Un EA puede tener una señal válida y una decisión de riesgo aprobada, y **aun así no debe operar** si algún breaker está activo. Nunca modeles el estado de seguridad como un único `bool trading_allowed` — usa un conjunto de razones explícitas.

Breakers típicos a implementar (lista no exhaustiva — adapta según la estrategia):

1. **Daily loss** — definir explícitamente qué es "inicio de día" (según balance o equity, decisión de diseño) y persistir ese estado (no solo en RAM — un restart no debe resetear el contador de pérdida diaria).
2. **Equity drawdown**.
3. **Margin** — distinto del Stop Out del broker; este breaker es una protección propia más conservadora.
4. **Demasiados rechazos de ejecución** — no todos los retcodes pesan igual; usa una ventana temporal y clasifica error individual vs patrón sostenido de error.
5. **Demasiadas pérdidas consecutivas** — reconstruido desde deals reales, no desde una variable en memoria (cuidado con la diferencia deal vs trade completo).
6. **Spread** anormal — este breaker normalmente NO debería bloquear automáticamente la gestión de posiciones existentes (solo nuevas entradas), a menos que se decida explícitamente lo contrario.
7. **Datos** (stale data, series no sincronizadas, indicadores no calculados).
8. **Estado del sistema desconocido** (`SYSTEM_RECOVERY`, `SYSTEM_FROZEN`) — fail-safe: ante duda, bloquear, no permitir.

### Histeresis y latching

Un breaker que compara contra un único umbral puede oscilar (`spread > X → block`, `spread <= X → allow`, repetido cada tick). Usa **dos umbrales** (uno para activar, otro más conservador para desactivar) o un **latch** (una vez activado, requiere condición explícita/cooldown para liberarse, no solo cruzar el umbral de vuelta).

### Permisos diferenciados, no un solo booleano

```
allow_new_entries
allow_position_modify
allow_partial_close
allow_full_close
allow_pending_orders
```

`BLOCKED` (la arquitectura decidió no enviar la solicitud) es conceptualmente distinto de `REJECTED` (se envió y el broker/MT5 la rechazó) — no los mezcles en el logging ni en las métricas; son capas de falla diferentes.

## Kill Switch, Freeze y Emergency

Cuatro conceptos distintos — no los uses como sinónimos:

- **STOP NEW ENTRIES** — no aumenta exposición, pero no toca lo existente.
- **FREEZE** — detiene operaciones (nuevas y de gestión) por incertidumbre/safety, a la espera de reconciliación.
- **EMERGENCY EXIT** — proceso explícito para reducir/cerrar exposición existente. Esto sigue siendo trading — debe pasar por la infraestructura segura de ejecución (Execution Gateway), no un atajo directo a `OrderSend`.
- **KILL SWITCH** — detiene la generación de nuevas acciones o el sistema completo.

**`ExpertRemove()` nunca es un Kill Switch de trading.** Solo descarga el EA del gráfico — no cierra posiciones ni cancela pendientes. Confundir esto es un error crítico y común.

### Secuencia de emergencia recomendada

```
TRIGGERED → BLOCK ENTRIES → CANCEL PENDING → RECONCILE
   → CLASSIFY OWNERSHIP → CLOSE/REDUCE OWNED EXPOSURE
   → RECONCILE → VERIFY → FREEZE
```

Nunca cerrar ciegamente "todo lo que se vea" sin clasificar ownership primero (ver `netting-hedging-ownership.md`) — el Emergency Exit debe cerrar solo exposición atribuible al EA, salvo decisión explícita en contrario documentada.

### Estado operacional del sistema

Máquina de estados única, con un solo propietario autorizado a cambiarla:

```
STARTING → RECOVERY → READY → DEGRADED → FROZEN → EMERGENCY → SHUTTING_DOWN
```

Nunca permitas que múltiples componentes escriban el estado global arbitrariamente. Registra siempre el motivo de cada transición (`SystemIncident`) — nunca ocultes por qué el sistema cambió de estado.

## Regla de oro de esta sección

Ante conflicto entre rapidez, simplicidad y seguridad operacional, prioriza siempre la seguridad, especialmente cuando hay riesgo de aumentar exposición financiera. `BLOCKED` es siempre preferible a "operar con incertidumbre".