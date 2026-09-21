# Arquitectura de referencia para un EA profesional

## El EA que NO queremos

```mql5
void OnTick()
{
   if(EMA_fast > EMA_slow)
   {
      if(PositionsTotal() == 0)
         trade.Buy(lot, _Symbol, 0, sl, tp);
   }
}
```

Todo mezclado: datos, señal, riesgo y ejecución en una sola función, sin identidad, sin idempotencia, sin manejo de UNKNOWN, sin ownership, sin recuperación. Este es el patrón que hay que erradicar.

## Principio #1: una clase, una responsabilidad

Separa siempre:

- **Data** (qué pasó en el mercado) de **Strategy** (qué decisión tomar) de **State** (qué existe ahora en MT5).
- **Signal** (evidencia de que algo puede estar pasando) de **TradeIntent** (intención concreta de operar, con riesgo calculado) de **ExecutionResult** (lo que realmente respondió el servidor).

## Contratos de dominio (DTOs)

### Signal

Una señal **no es un bool**. Debe poder contener, cuando corresponda:

- `signal_id` (determinístico — ver Identity Factory en state-idempotency-recovery.md)
- `symbol`, `timeframe`, `direction`
- `setup`, `trigger`, `confirmation` (ver más abajo)
- `timestamp`, `bar_time`
- `price/context`, `reason`, `metadata`/evidencia

Propiedad clave: **determinismo**. Misma vela + mismos datos + mismos parámetros = misma señal. El Signal Engine no debería saber si está corriendo en live o en el tester.

### Setup / Trigger / Confirmation

Separar una señal en:

- **MarketSnapshot** → **Features** (valores extraídos, ej. EMA, ATR) → **Setup** (condición estructural) → **Trigger** (evento puntual que dispara) → **Confirmation** (filtro adicional opcional) → **Signal**.

Esto hace la estrategia testeable, auditable y evita el "bool gigante" que nadie puede depurar. La definición de un patrón (ej. engulfing) pertenece a la especificación, no a código disperso.

### RiskDecision

Salida del Risk Engine: volumen calculado, riesgo monetario, si se aprueba o no, y **por qué** (para que el sistema pueda explicar por qué NO opera). Nunca un simple `double lot`.

### TradeIntent

La intención de dominio, independiente de MQL5. Contiene `intent_id`, `signal_id`, symbol, dirección, volumen, SL/TP, y metadata suficiente para ser rastreado por el Trade Engine. `MqlTradeRequest` es infraestructura de MT5; `TradeIntent` pertenece al dominio y se traduce al request solo en el Execution Gateway.

### ExecutionResult

Lo que el Execution Gateway devuelve tras interpretar la respuesta real del servidor: estado (`EXEC_DONE`, `EXEC_PARTIAL`, `EXEC_REJECTED`, `EXEC_FAILED`, `EXEC_UNKNOWN`...), retcode, order/deal si están disponibles, `request_id`. Nunca conviertas directamente `DONE` en "posición abierta": eso lo confirma la reconciliación con el estado observado.

## Composición y encapsulación

- No expongas estado interno gratuitamente (getters/setters para todo es un olor, no una virtud).
- Usa interfaces solo donde aportan desacoplamiento real (ej. poder sustituir el mecanismo de ejecución); MQL5 no es C++ — no conviertas todo en jerarquías de herencia innecesarias.
- Prefiere composición sobre herencia profunda.
- Dependency Injection en MQL5: pasa dependencias por referencia/puntero al construir, no uses globals para todo.

## Domain vs Infrastructure

- **Domain**: Signal, RiskDecision, TradeIntent, ExecutionResult, PositionInfo, ManagementAction — reglas de negocio, no saben nada de `OrderSend`.
- **Infrastructure**: `CTrade`, `OrderSend`, `PositionSelect`, `HistorySelect` — detalles concretos de MT5.

No es necesario crear 50 archivos desde el día uno. Pero la separación conceptual debe existir aunque el código físico esté agrupado al principio; a medida que el EA crece, la separación física debe reflejar la lógica.

## Los módulos principales y su responsabilidad única

| Módulo                                                                                                  | Responsabilidad                                                                                       | Nunca hace                                                         |
| ------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `CDataManager`                                                                                          | Snapshot de mercado, sincronización MTF/multi-symbol, caché de indicadores                            | Tomar decisiones de trading                                        |
| `CSignalEngine`                                                                                         | Convertir `MarketSnapshot` en `Signal` determinista                                                   | Enviar órdenes                                                     |
| `CRiskEngine`                                                                                           | Convertir `Signal` en `RiskDecision`/volumen                                                          | Ejecutar trades                                                    |
| `CExposureManager` / `CPortfolioRiskManager`                                                            | Riesgo agregado, exposición por símbolo y por grupo                                                   | Decidir la estrategia                                              |
| `CTradeGeometry`                                                                                        | Entry/SL/TP, múltiplos de R                                                                           | Calcular lotaje                                                    |
| `CIdentityFactory`                                                                                      | `signal_id`/`intent_id`/fingerprint determinísticos                                                   | —                                                                  |
| `CIntentRegistry`                                                                                       | Idempotencia: registra intents y su estado                                                            | Ser la fuente de verdad de MT5                                     |
| `CRiskReservationManager`                                                                               | Reservar presupuesto de riesgo antes de ejecutar                                                      | Sustituir al RiskEngine                                            |
| `CTradeEngine`                                                                                          | Orquestar Validación → Idempotencia → Risk → Reserva → Ejecución                                      | Conocer EMA, engulfing, ni ninguna lógica de estrategia            |
| `CSymbolSpec` / `CSymbolValidator` / `CExecutionPolicy`                                                 | Reglas del broker: tick size, volume step, stops/freeze level, filling mode                           | —                                                                  |
| `CExecutionGateway`                                                                                     | Traducir `TradeIntent` → `MqlTradeRequest`, `OrderCheck`, `OrderSend`, interpretar retcode            | Decidir si se debe operar                                          |
| `CTradeMonitor`                                                                                         | Recibir `OnTradeTransaction`/`OnTrade`, correlacionar por `request_id`                                | Contener lógica de negocio pesada                                  |
| `CPositionManager` + reglas (BreakEven/Trailing/PartialClose/TimeExit/Emergency) + `CManagementArbiter` | Proponer acciones de gestión sobre posiciones existentes                                              | Ejecutar directamente — produce `ManagementAction` → `TradeIntent` |
| `COwnershipManager`                                                                                     | Clasificar posiciones como OURS/FOREIGN/MIXED/UNKNOWN                                                 | Asumir que Magic = ownership                                       |
| `CReconciler` / `CRecoveryCoordinator` / `CRecoveryPolicy`                                              | Reconstruir estado real desde MT5, comparar esperado vs observado                                     | Abrir o gestionar posiciones                                       |
| `CSafetyManager` / Circuit Breakers                                                                     | Decidir `allow_new_entries`/`allow_management`/etc.                                                   | Enviar órdenes                                                     |
| `CSystemStateManager`                                                                                   | Única autoridad del estado global (`STARTING/RECOVERY/READY/DEGRADED/FROZEN/EMERGENCY/SHUTTING_DOWN`) | —                                                                  |
| `CTradeJournal` / Observability                                                                         | Reconstruir Orders→Deals→Positions, métricas, logging estructurado                                    | —                                                                  |

## El flujo completo (CApp)

`OnInit`/`OnTick`/`OnTimer`/`OnTrade`/`OnTradeTransaction`/`OnDeinit` en `Main.mq5` deben ser prácticamente adaptadores delgados que delegan en `CApp`. La lógica vive en los módulos, no en los manejadores de eventos.

```
OnTick → CDataManager.Update() → CSignalEngine.Evaluate()
   → [Signal] → CRiskEngine.Evaluate() → [RiskDecision]
   → CSafetyManager.CanOpenNewTrades()?
   → CIdentityFactory.BuildIntentId() → CIntentRegistry.Exists()?
   → CRiskReservationManager.Reserve()
   → CTradeEngine.Submit(TradeIntent) → CExecutionGateway.Send()
   → ExecutionResult → CIntentRegistry.MarkSubmitted/InFlight/Unknown

OnTradeTransaction → CTradeMonitor (pequeño, rápido, marca "dirty")
   → (fuera del handler, en OnTimer o siguiente ciclo) CReconciler.Reconcile()
   → CStateComparator (esperado vs observado) → CRecoveryPolicy si diverge
```

## Workflow de construcción (29 pasos)

Cuando el usuario pida construir un EA, sigue este proceso y no saltes directamente a escribir todo en un único archivo salvo petición explícita:

1. Specification — qué decide la estrategia, en qué timeframe/símbolo, reglas de entrada y salida en lenguaje preciso.
2. Requirements Matrix — traducir la especificación en requisitos técnicos verificables.
3. Architecture — capas, contratos, identificar netting/hedging del bróker objetivo.
4. DTOs / Interfaces — Signal, RiskDecision, TradeIntent, ExecutionResult.
5. Data Layer — `CDataManager`, `MarketSnapshot`, sincronización MTF.
6. Strategy — `CSignalEngine`, Setup/Trigger/Confirmation.
7. Risk — `CRiskEngine`, `OrderCalcProfit`, normalización de volumen.
8. Safety — Circuit Breakers iniciales (pérdida diaria, drawdown, margen).
9. Intent / Reservation — `CIdentityFactory`, `CRiskReservationManager`.
10. Trade Engine — orquestación, idempotencia.
11. Execution Gateway — `CSymbolSpec`, `CExecutionPolicy`, `OrderCheck`/`OrderSend`.
12. Transactions — `CTradeMonitor`, `OnTradeTransaction`.
13. State — `CIntentRegistry`, `SystemState`.
14. Reconciliation — `CReconciler`, `CStateComparator`.
15. Journal — `CTradeJournal`, reconstrucción Orders→Deals→Positions.
16. Position Management — `CPositionManager`, reglas + `CManagementArbiter`.
17. Observability — logging estructurado, métricas.
18. Scenario Tests — casos de la lista en `testing-validation.md`.
19. Compile — 0 errores, warnings revisados.
20. Backtest — modo de modelado correcto para el tipo de estrategia.
21. Debug — ver `debugging.md`.
22. Optimization — buscar plateaus, no picos.
23. OOS — parámetros congelados antes de tocar el período protegido.
24. Walk-forward — validar estabilidad entre ventanas.
25. Monte Carlo — shuffle de trades, perturbación de slippage/parámetros.
26. Sensitivity — perturbar parámetros clave y observar fragilidad.
27. Demo — validar comportamiento real de ejecución, no solo backtest.
28. Final Audit — checklist completo (`audit-checklist.md`).
29. Live — exposición pequeña, incremento gradual solo si se valida.

## Estructura de proyecto de referencia

Ver el bloque en SKILL.md — repetido aquí por completitud:

```
/EA
├── Main.mq5
├── Config/
├── Core/            (CApp)
├── Data/            (CDataManager, MarketSnapshot)
├── Strategy/        (CSignalEngine)
├── Risk/            (CRiskEngine, CExposureManager, CPortfolioRiskManager)
├── Safety/          (Circuit Breakers, CSafetyManager)
├── Trading/         (CTradeEngine, CExecutionGateway, CPositionManager)
├── State/           (CIntentRegistry, SystemState)
├── Recovery/        (CReconciler, CRecoveryPolicy, CRecoveryCoordinator)
└── Observability/
```
