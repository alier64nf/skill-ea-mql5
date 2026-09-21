# Strategy Tester, Optimización, OOS, Walk-Forward, Monte Carlo, Demo/Live

## Backtest ≠ validation. Optimization ≠ robustness.

Estos dos principios gobiernan todo este archivo. Un backtest rentable no demuestra que un EA sea correcto ni robusto; una optimización que maximiza una métrica no demuestra robustez — puede ser sobreajuste puro.

## Modos de modelado del Strategy Tester

- **Open prices only** — solo el precio de apertura de cada barra; el más rápido, el menos representativo para estrategias intrabar.
- **1 Minute OHLC** — reconstruye movimiento dentro de la barra a partir de M1, pero **OHLC no contiene el orden real de High y Low dentro de la barra** — el tester debe *inferir* una trayectoria, que puede no coincidir con la real.
- **Every tick** — genera ticks sintéticos con mayor resolución, mejora pero no elimina el problema anterior.
- **Every tick based on real ticks** — el más fiel cuando hay datos de ticks reales disponibles para el símbolo/periodo.

El error clásico: probar una estrategia que decide intrabar (no solo en cierre de vela) usando un modelado insuficiente (ej. Open prices only) — esto explica muchos "EAs mágicos" que rinden espectacularmente en backtest y fallan en vivo. Para estrategias basadas puramente en velas cerradas (M1+ como mínimo timeframe de decisión), un modelado M1/Every tick suele ser razonable; para estrategias muy sensibles a timing intrabar, exige Every tick based on real ticks cuando sea posible.

Registra siempre junto a cada backtest: EA version, Strategy version, Symbol, Timeframe, Period, Model, Spread, Commission, Swap, Account mode, Inputs, Data source. Nunca compares dos backtests sin saber si corrieron bajo las mismas condiciones.

## Lookahead bias

Busca específicamente: `Close[0]`, `iClose(..., 0)`, `CopyBuffer(..., 0, ...)` — no significa automáticamente que estén mal, pero exige preguntar: *¿ese dato representa una vela todavía en formación?* Si la estrategia pretende decidir sobre una vela cerrada, debe demostrarse explícitamente (usando shift correcto y `BarsCalculated()`/sincronización, ver `references/architecture.md`). Un lookahead más sutil: usar en el backtest información que en tiempo real no estaría disponible aún en ese instante exacto (ej. un indicador de un símbolo secundario no sincronizado por tiempo).

## Diseño de experimentos

Define la hipótesis **antes** del test. Cambia **una variable experimental a la vez**. No mezcles cambios de lógica de estrategia con cambios de parámetros en la misma corrida si quieres poder atribuir el efecto correctamente. No cambies la lógica de negocio entre Tester y Live (`MQLInfoInteger(MQL_TESTER)` puede usarse para ajustes de infraestructura como logging, nunca para alterar decisiones financieras).

## Optimización sin destruir la estrategia

- Optimización ≠ "encontrar el parámetro que más gana". Busca **plateaus** (mesetas de estabilidad), no picos aislados. Un pico aislado (ej. parámetro=17 da $100k pero 16 y 18 dan mucho menos) es sospechoso de sobreajuste; una meseta amplia y estable es una superficie más informativa (no una garantía automática de robustez, pero mucho mejor señal).
- Separa **parámetros estructurales** (forman parte de la hipótesis de la estrategia) de **parámetros libres** (detalles de implementación) — no optimices lo que no necesita optimizarse.
- Maximizar únicamente Profit es peligroso — considera un criterio de fitness más completo (ej. penalizar por número insuficiente de operaciones, por drawdown, por inestabilidad).
- Optimiza **por capas** en vez de todos los parámetros simultáneamente cuando el espacio de búsqueda es grande (ej. primero geometría de entrada/salida, después gestión de riesgo, después parámetros finos).
- La optimización no debe modificar la arquitectura del EA ni la lógica financiera entre corridas.

## Out-of-Sample (OOS) y Forward

- **IS** (In-Sample) — datos usados para diseño/selección de parámetros.
- **OOS** (Out-of-Sample) — datos protegidos, **nunca** usados para volver a seleccionar parámetros.

```
IS → Optimization → Select candidate → FREEZE PARAMETERS → OOS → Evaluate
```

Regla que nunca debe romperse: si vuelves a ajustar parámetros usando el resultado de OOS, has contaminado la validación — deja de ser independiente. Una sola división IS/OOS puede ser insuficiente; considera múltiples ventanas.

## Walk-Forward

```
Window 1: IS → OOS
Window 2: IS → OOS
Window 3: IS → OOS
Window 4: IS → OOS
```

La pregunta correcta es *¿la lógica mantiene características razonables a través de distintos períodos?*, no *¿cuál fue el mejor período?*. Evalúa consistencia de métricas (no solo Profit Factor — también drawdown, distribución temporal de operaciones, comportamiento por régimen de mercado) a través de las ventanas, no solo el resultado agregado.

## Monte Carlo y análisis de sensibilidad

Distingue explícitamente los tipos de Monte Carlo — no son intercambiables:

- **Trade-shuffle Monte Carlo** — reordena la secuencia de trades ya ocurridos. No simula nuevos mercados; analiza cuánto depende el resultado del orden en que ocurrieron los trades (sequence risk, drawdown, streaks).
- **Bootstrap** — muestrea con reemplazo sobre los trades históricos.
- **Perturbación de resultados** — perturba slippage, costos, u otros parámetros de ejecución sobre los trades ya ocurridos.
- **Market-price Monte Carlo** — simula trayectorias de precio nuevas; es un problema y una metodología completamente distintos de barajar trades — no confundas ambos enfoques ni sus conclusiones.

Un Monte Carlo mal construido asume independencia entre trades cuando puede no existir — sé explícito sobre esa hipótesis. Percentiles como P95 deben interpretarse como resultado de la simulación bajo el modelo y las hipótesis usadas, **no como garantías del futuro**. Exige siempre: método, número de simulaciones, seed (documentado para reproducibilidad), hipótesis de distribución, limitaciones explícitas.

**Análisis de sensibilidad**: perturba parámetros clave (ej. período de EMA, distancia de SL/TP, thresholds) uno a la vez y observa si el resultado colapsa. Una región de parámetros vecinos con resultados similares indica robustez relativa; un colapso brusco ante una perturbación pequeña indica fragilidad que merece investigación antes de continuar.

Este código de investigación (Monte Carlo, sensibilidad, walk-forward) debe vivir separado del EA de producción — no contamines el código que se ejecuta en vivo con lógica experimental.

## Checklist de Monte Carlo antes de confiar en el resultado

- ¿Qué se está perturbando exactamente (secuencia, slippage, parámetros, precio)?
- ¿Cuántas simulaciones?
- ¿Seed documentado?
- ¿Qué hipótesis de independencia/distribución se asumió?
- ¿Qué limitaciones tiene explícitamente este análisis?

## Demo → Audit → Live

Demo **no significa "ya está listo para vivo"**. Demo es una prueba de comportamiento de ejecución real: broker constraints, latencia, fills, spread real, manejo de transacciones, recuperación, calidad de logs — condiciones que el backtest no reproduce fielmente.

### Checklist antes de Demo
Compilación limpia, scenario tests, backtest, OOS, robustness tests, execution tests, recovery tests, logging, métricas, kill switch, procedimiento de emergencia.

### Durante Demo, monitorear activamente
Resultados `UNKNOWN`, órdenes rechazadas, fills parciales, latencia, spread real, ownership inesperado, transiciones de estado del sistema.

### Transición a Live — gradual, nunca directa
```
DEMO → AUDIT → SMALL LIVE EXPOSURE → MONITOR → INCREASE ONLY IF VALIDATED
```
No existe "mi backtest dio 80% así que live". La transición debe ser gradual y auditada en cada paso — ver `audit-checklist.md` para la matriz de readiness completa y qué constituye un fallo crítico que impide avanzar independientemente del resultado financiero.

## Escenarios de prueba mínimos (scenario tests)

Como mínimo, un EA debe tener pruebas de escenario diseñadas (no solo backtest de estrategia) para: entrada normal, señal duplicada, orden rechazada, fill parcial, resultado `UNKNOWN`, restart del EA, restart del terminal, desconexión/reconexión, cierre manual, modificación manual, posición foreign, ownership mixto, reversal en netting, múltiples posiciones en hedging, conflicto entre reglas de gestión (ej. Trailing vs BreakEven vs Emergency simultáneos), emergencia, kill switch, datos obsoletos (stale), margen insuficiente, volumen inválido, precio inválido, restricciones del broker (stops level/freeze level).