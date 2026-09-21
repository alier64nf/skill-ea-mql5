# Debugging profesional de EAs

## Cuatro niveles que no deben confundirse

```
COMPILACIÓN → ¿El código es sintácticamente válido?
RUNTIME     → ¿El programa puede ejecutarse sin errores críticos?
LÓGICA      → ¿Hace lo que debería?
TRADING     → ¿El comportamiento financiero es correcto?
```

Un EA puede compilar perfectamente, ejecutar sin errores críticos, incluso producir ganancias en el Tester, y aun así estar lógicamente equivocado. No confundas ninguno de estos cuatro niveles con "está bien".

## Clasificación de bugs (usa esto primero, antes de proponer un fix)

1. **Bug de datos** — dato equivocado, dato viejo, dato incompleto, shift equivocado, serie no sincronizada.
2. **Bug temporal/evento** — vela incorrecta, lookahead, evento duplicado, evento perdido.
3. **Bug de estado** — el sistema cree que está `READY` cuando en realidad está en `RECOVERY`, o posición esperada ≠ posición observada.
4. **Bug de ejecución** — request enviado ≠ operación realmente ejecutada.
5. **Bug de ownership** — posición atribuida al EA ≠ posición realmente del EA (ver `netting-hedging-ownership.md`).
6. **Bug de idempotencia** — misma intención lógica produce dos órdenes.
7. **Bug de recuperación** — tras desconexión→reconexión el EA asume un estado en vez de reconciliar (`reconexión → asume estado` cuando debería ser `reconexión → RECONCILE → comparar → recuperar`).
8. **Bug de sincronización/race** — reentrancia, ticks rápidos, `OnTradeTransaction` procesado fuera de orden.
9. **Bug de normalización de precio/volumen** — comparaciones `==` sobre `double`, redondeo hacia el lado equivocado, ignorar tick size/volume step.
10. **Discrepancia tester/live** — comportamiento que solo aparece o solo desaparece en el Strategy Tester (ver `testing-validation.md` sobre modos de modelado y por qué esto ocurre).

## El debugger no busca el bug por ti

No hagas simplemente `Run → algo sale mal → Print() → Run → otro Print() → Run → más Print()`. Eso funciona para problemas triviales. Para un EA complejo:

```
Symptom → Reproduce → Localize Layer → Capture Evidence
   → Hypothesis → Instrument → Verify → Fix → Regression Test
```

- **Localize Layer**: usa la arquitectura en capas (Data/Strategy/Risk/Execution/State/Reconciliation) para acotar dónde puede estar el problema antes de instrumentar código.
- **Capture Evidence**: usa logging estructurado existente (`signal_id`, `intent_id`, `request_id`, retcode) en vez de añadir `Print()` genéricos que se descartan después.
- **Regression Test**: cada bug corregido debería convertirse en un escenario de prueba permanente (ver lista de escenarios en `testing-validation.md`), no solo verificarse una vez manualmente.

Herramientas del MetaEditor disponibles: debugger integrado, breakpoints, ejecución paso a paso (Step Over vs Step Into), inspección de variables, call stack, `DebugBreak()` (útil para pausar condicionalmente en un escenario específico), y depuración directa sobre datos históricos vía Strategy Tester.

## GetLastError() no sustituye al retcode de trading

Recordatorio crítico también para debugging: si estás depurando un problema de ejecución y solo miras `GetLastError()`, estás mirando la capa equivocada. Mira `MqlTradeResult.retcode` primero.

## Debugging de idempotencia y sincronización

Estos son los más difíciles de reproducir porque dependen de timing. Para depurarlos:
- Revisa los logs por `intent_id`/`request_id`, no por timestamp aislado — reconstruye la secuencia completa de eventos para un mismo intent.
- Verifica si `OnTradeTransaction()` está haciendo trabajo pesado dentro del handler (causa común de comportamiento errático bajo ticks rápidos).
- Simula explícitamente el escenario de doble tick / señal duplicada en el Tester si es posible, en vez de solo confiar en observación pasiva.