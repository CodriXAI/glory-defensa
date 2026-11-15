---
tags: [glory, revisado, penales]
---
# R7 — Penales (los tickets que revisé de la épica GLORY-277)
> Volver a [[00 - Inicio]] · Mis tickets de penales: [[Penales (GLORY-277)]]

## La historia
Antes de que el motor pueda jugar penales ([[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]]) hacía falta: una **opción** en la liga ([[R7 - Penales#GLORY-291 — Opción de penales en la liga|GLORY-291]]), un **marcador** de penales ([[R7 - Penales#GLORY-281 — Marcador de penales|GLORY-281]]) y una forma de **mostrar y puntuar** el resultado ([[R7 - Penales#GLORY-293 — Exponer el resultado y adaptar la tabla|GLORY-293]]). Los tres los hizo Noah y los revisé.

| Ticket | Qué hace |
|---|---|
| [[R7 - Penales#GLORY-291 — Opción de penales en la liga\|GLORY-291]] | Opción `penalties_enabled` al crear una liga |
| [[R7 - Penales#GLORY-281 — Marcador de penales\|GLORY-281]] | Marcador `phome` / `paway` |
| [[R7 - Penales#GLORY-293 — Exponer el resultado y adaptar la tabla\|GLORY-293]] | Exponer el resultado de penales y adaptar la tabla de posiciones |

## GLORY-291 — Opción de penales en la liga
- **Código clave**: `League.penalties_enabled` (bool, `default False`, `server_default false`; migración `73f3c6df5cb9`); `CreateLeagueRequest.penalties_enabled`; `league_service.create_league`; `match_start_service.prepare_match` → `build_initial_state(..., penalties_enabled)`.
- **Idea clave**: se guarda **en la liga** y se copia al estado de cada partido. Los **amistosos nunca** tienen penales.
- **Por qué `server_default`**: las ligas que ya existían reciben `false` al migrar.
- **Test**: `test_penalties_enabled.py`.

## GLORY-281 — Marcador de penales
- **Código clave**: `Score.phome` / `Score.paway` (opcionales: `None` si no hubo penales); `match_broadcast.build_tick_update_payload` y `ScoreOut` los incluyen.
- **Idea clave**: `None` y no `0` para distinguir "no hubo penales" de "hubo y nadie convirtió" (cada ronda arranca 0-0 recién cuando empieza la tanda: [[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]]).
- **Test**: `test_score_penalties.py`.

## GLORY-293 — Exponer el resultado y adaptar la tabla
- **Código clave**: `standings_service` (`PENALTY_WIN`; `POINTS` suma la clave `PENALTY_WIN: 2`), `penalty_goals` en el resultado, `FixtureScoreOut(phome, paway)`; columnas `home_penalty_goals` / `away_penalty_goals` en `matches` (migración `bae51cba2fdb`).
- **Reglas**: `POINTS = {WIN: 3, DRAW: 1, LOSS: 0, PENALTY_WIN: 2}`. Ganar en tiempo reglamentario = 3; **ganar por penales = 2**; **perder por penales = 0** (cuenta como derrota); empate sin penales = 1. **Los goles de penales NO cuentan** para goles a favor / en contra. En la tabla, `PENALTY_WIN` se cuenta como **victoria** (`RESULT_FIELD`).
- **Idea clave**: las columnas de la migración se movieron a este ticket porque [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]] estaba bloqueado.
- **Test**: `test_penalties_result.py`.
- **Ojo (inconsistencia conocida)**: un 0-0 cerrado por una preparación fallida en una liga **con** penales cuenta como empate, y el contrato dice que con penales no hay empates ([[05 - Inconsistencias conocidas]]).
