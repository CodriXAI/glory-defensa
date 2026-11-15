---
tags: [glory, desarrollado, pausas, back, motor]
ticket: GLORY-216
epica: Pausas (GLORY-196)
tipo: Story
puntos: 3
reviewer: Alexander Huaman Rojas
estado: Done
---
# GLORY-216 — Implementar lógica de ticks en pausas Cooling Break y Half-Time
> Épica: [[Pausas y sustituciones (GLORY-196 y 197)]] · Base de [[GLORY-217 Cambio de posición tras Half-Time|GLORY-217]], [[GLORY-215 Función substitute_player en el motor|GLORY-215]], [[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas|GLORY-218]]

## En una frase
Cuando el partido llega a un tick de pausa, el motor pasa a la fase `PAUSED`, **el reloj se detiene** (`current_tick` no avanza y no se envían `TICK_UPDATE`) durante la pausa, y al terminar vuelve a `IN_PROGRESS` y sigue desde el mismo tick.

## Qué pedía el ticket
- Calendario proporcional a la duración: Cooling Break al 25 % y 75 %, Half-Time al 50 %. 240 ticks → 60/120/180; 600 ticks → 150/300/450.
- Duración: Cooling Break 10 s, Half-Time 20 s.
- Constantes y `pause_schedule_for(total_ticks)` en `metrics.py`; `PAUSED` en `MatchPhase`.
- Para cualquier `total_ticks` distinto de 240 y 600, devolver lista vacía (sin pausas) para no afectar tests viejos.
- Actualización del 16/10: `state.pause` es un objeto **nuevo en cada pausa** (tipo, número, instante de fin y `substituted_team_ids`) y vuelve a `None` al terminar.

## Dónde está en el código
| Qué | Archivo | Detalle |
|---|---|---|
| Calendario y duraciones | `src/engine/_01_to_world/metrics.py` | `PAUSE_PLAN`, `PAUSE_DURATION_SECONDS`, `PauseSlot`, `pause_schedule_for` |
| Fase | `src/engine/_02_to_match/state.py` | `MatchPhase.PAUSED` |
| La pausa en curso | idem | `PauseState(type, number, ends_at, substituted_team_ids)` |
| Bucle del partido | `src/engine/match_runner.py` | `run_match` (variable local `next_pause`) |
| La pausa en sí | idem | `_run_pause` |
| Pasar duración por parámetro | idem y `match_start_service.launch_match` | `pause_durations` |

## Cómo funciona
1. `run_match` calcula `pause_schedule = pause_schedule_for(state.total_ticks)` y lleva `next_pause = 0`.
2. En cada vuelta, **antes** de jugar el tick, si `current_tick` es el de la próxima pausa → `await _run_pause(...)` y `next_pause += 1` (así ninguna pausa se repite).
3. `_run_pause`: `phase = PAUSED`, crea `state.pause` con `ends_at = time.monotonic() + duración`, deja actuar al DT ([[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor|GLORY-297]]), aplica los cambios planificados ([[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]]), avisa `PAUSE_STARTED` ([[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas|GLORY-218]]), `await asyncio.sleep(duración)`, y al final (si es Half-Time) `swap_sides` ([[GLORY-217 Cambio de posición tras Half-Time|GLORY-217]]), `pause = None`, `phase = IN_PROGRESS`, avisa `PAUSE_FINISHED`.

## Decisiones de diseño
- **Aritmética entera** (`total_ticks * porcentaje // 100`): sin decimales ni redondeos raros.
- **`time.monotonic()` y no `time.time()`**: es un reloj que solo avanza y no se ve afectado por cambios de la hora del sistema (ajuste de red, horario de verano), así el tiempo restante nunca da negativo ni salta.
- **`next_pause` es una variable local** de `run_match`, no se guarda en `MatchState`: es un detalle del bucle.
- **`PauseState` nuevo en cada pausa**: así `substituted_team_ids` empieza vacío solo, sin tener que limpiarlo.
- **La duración entra por parámetro** (con los valores de `metrics.py` por defecto): los tests de partido completo pasan duración 0 y no esperan.
- **Solo hay pausas con 240 o 600 ticks** (duraciones válidas 2 y 5 min): otros totales devuelven `[]`.
- **En la pausa no corren los jugadores** (solo el DT, agregado en [[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor|GLORY-297]]).

## Qué pasa si falla
- Falla el aviso de pausa (`notify_pause`) → `_notify_pause_safely` lo registra y el partido sigue.
- Si no se pasa `notify_pause`, el partido corre igual que antes.

## Tests
`tests/unit/test_match_pauses.py`

## Relacionado con
[[GLORY-217 Cambio de posición tras Half-Time|GLORY-217]] · [[GLORY-215 Función substitute_player en el motor|GLORY-215]] · [[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]] · [[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas|GLORY-218]] · [[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor|GLORY-297]]

## Preguntas que me pueden hacer
- ¿En qué ticks hay pausas con un partido de 2 minutos? ¿Y de 5?
- ¿Por qué `time.monotonic()`?
- ¿Qué pasa con `current_tick` y con los `TICK_UPDATE` durante la pausa?
- ¿Qué pasa si `total_ticks` no es 240 ni 600?
- ¿Por qué `pause` es un objeto nuevo en cada pausa?

## Con mis palabras (completar)
- Qué hice:
- Por qué así:
