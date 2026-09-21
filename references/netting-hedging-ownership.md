# Netting vs Hedging y Ownership

Esta es una de las diferencias que más cambia la arquitectura de un EA. Trátala explícitamente en cualquier diseño o revisión, y pregunta siempre por el `ACCOUNT_MARGIN_MODE` / modo de cuenta objetivo si no se ha especificado.

## Netting

Un símbolo tiene **una posición agregada** por Magic/estrategia (según política de ownership). Operaciones opuestas reducen, cierran o revierten esa posición agregada — nunca coexisten como posiciones independientes.

```
BUY 1.50 + SELL 0.50 = BUY 1.00          (reducción)
BUY 1.50 + SELL 2.00 = SELL 0.50         (reversal)
```

Un reversal en netting es en realidad **dos eventos económicos** (cerrar 1.50 + abrir 0.50 en la dirección contraria) aunque MT5 lo represente como una operación fluida. No sumes/restes volúmenes de deals ciegamente para reconstruir el lifecycle: usa `DEAL_ENTRY` (`DEAL_ENTRY_IN`, `DEAL_ENTRY_OUT`, `DEAL_ENTRY_INOUT` para reversals) para clasificar correctamente cada deal.

`PositionSelect(_Symbol)` en netting devuelve la posición neta del símbolo — esto puede ser razonable en netting puro, pero es peligroso si hay más de un Magic/estrategia operando el mismo símbolo (ver Ownership abajo).

## Hedging

Pueden existir **múltiples posiciones independientes** en el mismo símbolo simultáneamente, cada una con su propio ticket. `PositionSelect(_Symbol)` **no** identifica una posición única — es ambiguo y peligroso en hedging. Usa siempre el ticket explícito (`PositionSelectByTicket()`) para operaciones de gestión (modificar SL/TP, cerrar, partial close).

`Close By` (`TRADE_ACTION_CLOSE_BY`) permite cerrar dos posiciones opuestas entre sí — es una operación específica de hedging, no reproducible directamente en netting.

## Position Ticket vs Position Identifier

- **POSITION_TICKET**: identificador de la posición en un momento dado — puede no ser estable a través de todo el lifecycle en algunos escenarios (particularmente en netting, tras partial closes/reversals).
- **POSITION_IDENTIFIER**: el eje histórico correcto para reconstruir el lifecycle completo de una posición a través de `HistorySelectByPosition()`. Prefiere `POSITION_IDENTIFIER` sobre `POSITION_TICKET` cuando reconstruyas historial; no los trates como intercambiables.

## No confundir SL/TP absolutos con relativos

`SYMBOL_VOLUME_LIMIT` es un límite de volumen abierto (posiciones + pendientes) **por dirección**, distinto de `SYMBOL_VOLUME_MAX` por operación individual.

## Ownership: por qué Magic Number no es suficiente

`POSITION_MAGIC == MyMagic` **no es una respuesta universal de ownership**. En netting, dentro del mismo lifecycle de una posición agregada puede haber:
- Deals del propio EA (`OUR DEAL`)
- Deals manuales del usuario (`MANUAL DEAL`)
- Deals de otro EA con Magic distinto (`OTHER EA DEAL`)

Todos mezclados en la misma posición neta. Un EA que asume `magic == mío → toda la posición es mía` puede modificar o cerrar exposición que no le pertenece.

### Clasificación de ownership

Clasifica siempre en cuatro estados, nunca solo binario "es mía / no es mía":

- **OURS** — evidencia suficiente (histórico de deals reconstruido) de que la posición es del EA.
- **FOREIGN** — evidencia suficiente de que es de otro actor (manual o otro EA/Magic).
- **MIXED** — evidencia de contribución de más de un actor dentro del mismo lifecycle (típico en netting).
- **UNKNOWN** — no hay evidencia suficiente para decidir. **Nunca conviertas UNKNOWN en FOREIGN ni en NONE por defecto** — trátalo con la misma cautela que un `UNKNOWN` de ejecución (ver `state-idempotency-recovery.md`).

### Cómo reconstruir ownership

Usa `HistorySelectByPosition(POSITION_IDENTIFIER)`, recorre los deals y clasifica cada uno por `DEAL_MAGIC`, `DEAL_REASON`, `DEAL_ENTRY`. Ninguna propiedad aislada es suficiente por sí sola:
- `DEAL_MAGIC` puede no estar presente o ser 0 para operaciones manuales.
- `DEAL_REASON` (`DEAL_REASON_EXPERT`, `DEAL_REASON_MOBILE`, `DEAL_REASON_CLIENT`, etc.) da pistas pero no es autoridad absoluta.
- `POSITION_REASON` tiene limitaciones similares.
- El `comment` de una orden/deal es metadata útil pero **nunca** debe ser la única fuente de identidad u ownership — puede ser sobrescrito o ausente.

Modelo recomendado: **contribution ledger** — reconstruir el flujo de contribución (no solo una fotografía del estado actual) recorriendo el histórico de deals, y asignar un nivel de confianza (`OwnershipConfidence`: EXACT / STRONG / AMBIGUOUS / UNKNOWN) en vez de un booleano.

### Política de ownership

Ante ambigüedad, define explícitamente una política (documentada, no implícita en el código):
- `EXCLUSIVE` — el EA solo actúa si tiene certeza absoluta de ownership exclusivo.
- `BY_MAGIC` — usa Magic como heurística principal (más simple, más riesgo).
- `LEDGER` — usa el contribution ledger reconstruido como autoridad.
- `ACCOUNT_COORDINATED` — coordina explícitamente con otros EAs/Magic conocidos en la misma cuenta.

Regla de seguridad: **nunca "adoptes" automáticamente una posición foreign o MIXED** (no significa "tomar posesión" — significa empezar a gestionarla sin autorización explícita del usuario/política). Ante MIXED, la acción por defecto debe ser conservadora: bloquear nuevas entradas equivalentes y, si es necesario gestionar, hacerlo con la política más restrictiva disponible (ej. no reducir volumen que no se puede atribuir con certeza al EA).

## Consecuencia para PositionManager, RiskEngine y TradeEngine

- **PositionManager**: nunca ejecutes Break-Even/Trailing/Partial Close sobre una posición sin antes resolver su ownership. En hedging, opera siempre por ticket explícito, nunca por símbolo.
- **RiskEngine/ExposureManager**: separa `known_risk` de `unknown_risk` en el portfolio snapshot. Una posición con ownership `UNKNOWN` no debe contar como riesgo cero — cuenta como riesgo no atribuible, y la política de exposición debe decidir explícitamente cómo tratarlo (normalmente, de forma conservadora).
- **TradeEngine**: nunca abras una nueva posición asumiendo que el símbolo está "libre" solo por no ver posiciones con tu Magic — en hedging puede haber posiciones foreign en el mismo símbolo que no bloquean nada, pero en netting una posición existente (aunque sea MIXED) sí cambia la interpretación de una nueva orden en dirección contraria.

## Trade Journal y P&L real (CTradeJournal)

No trabajes con `HistoryDealGet...()` disperso por todo el EA — centraliza la reconstrucción de historial en un `CTradeJournal` dedicado.

- `HistorySelect()` no es "la base de datos completa" — solo selecciona un rango; siempre selecciona explícitamente el rango relevante antes de leer.
- Usa `HistorySelectByPosition(POSITION_IDENTIFIER)` para reconstruir el lifecycle completo de una posición específica, no recorras todo el historial de la cuenta cuando solo necesitas una posición.
- Una posición puede tener **muchos deals** (entradas parciales, salidas parciales, reversal) — nunca asumas 1 order = 1 deal ni 1 order = 1 trade completo.
- `DEAL_PROFIT` no es el resultado neto completo — hay que sumar también comisión y swap (`DEAL_SWAP`) para el P&L real; no mezcles "profit" bruto con "net profit".
- Precio medio de entrada/salida: cuando hay múltiples deals de entrada o salida parcial, el precio medio debe calcularse ponderado por volumen — no tomar solo el primer o último deal.
- Cuidado con deals `INOUT` (reversal): no son un simple `+volume` — representan cierre de la posición anterior y apertura de la nueva dirección en la misma transacción; deben descomponerse antes de calcular P&L o riesgo.
- Esto permite reconstruir una operación completa después de reiniciar el EA (auditoría tras un crash) y detectar inconsistencias entre lo que el EA cree y lo que MT5 realmente registró — no dependas de que el EA "recuerde" el resultado si puede reconstruirlo desde el historial.

## Regla que debes aplicar al revisar código de Claude/IA

Cuando revises código generado por IA, busca específicamente:
- `PositionSelect(_Symbol)` usado como si identificara una única posición en un contexto que podría ser hedging.
- Cualquier lógica que trate `POSITION_MAGIC == MyMagic` como prueba suficiente de ownership completo sin reconstrucción del histórico.
- Lógica de netting implementada asumiendo semántica de hedging (o viceversa) sin verificar el `ACCOUNT_MARGIN_MODE` real de la cuenta objetivo.
- Cierre o modificación de una posición MIXED sin política explícita.