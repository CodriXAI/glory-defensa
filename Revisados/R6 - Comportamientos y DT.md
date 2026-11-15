---
tags: [glory, revisado, comportamientos, dt]
---
# R6 — Comportamientos de jugador y DT
> Volver a [[00 - Inicio]] · Mis tickets de DT: [[DT (GLORY-282)]]

## La historia
Cada jugador guarda **su propio código de comportamiento** ([[R6 - Comportamientos y DT#GLORY-259 — Guardar el `on_tick` en cada jugador|GLORY-259]]); el usuario puede **cambiarlo en pleno partido** ([[R6 - Comportamientos y DT#GLORY-260 — Cambiar el comportamiento durante el partido|GLORY-260]]); además un equipo de liga puede tener un **DT** que se **elige** ([[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga|GLORY-299]]) y **se ejecuta en el motor** ([[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor|GLORY-297]]), usando las primitivas que hice en [[GLORY-296 Primitivas de director técnico|GLORY-296]] y con la validación de [[GLORY-288 Separar comportamientos de jugador y de DT|GLORY-288]].

| Ticket | Épica | Quién | Qué hace |
|---|---|---|---|
| [[R6 - Comportamientos y DT#GLORY-259 — Guardar el `on_tick` en cada jugador\|GLORY-259]] | GLORY-198 Cambiar comportamiento durante el partido | Noah | Cada jugador guarda su función `on_tick` |
| [[R6 - Comportamientos y DT#GLORY-260 — Cambiar el comportamiento durante el partido\|GLORY-260]] | GLORY-198 | Noah | Endpoint para cambiar el comportamiento durante el partido |
| [[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga\|GLORY-299]] | GLORY-282 Comportamiento de DT | Noah | Elegir el DT de la liga |
| [[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor\|GLORY-297]] | GLORY-282 | Noah | Ejecutar el código del DT en el motor |

## GLORY-259 — Guardar el `on_tick` en cada jugador
- **Qué hace**: antes había una sola función compartida para ejecutar comportamientos; ahora cada `MatchPlayerState` tiene **su propio `on_tick`**.
- **Código clave**: `state.MatchPlayerState.on_tick`; `behavior_executor.assign_behavior` (**compila primero, asigna después**), `load_on_tick`, `execute_behavior`; `match_start_service.prepare_engine`; se eliminó `make_execute_behavior`.
- **Idea clave**: cada jugador tiene **su copia del código** (no comparten variables). Si el código no compila, **el jugador conserva el anterior**. Es lo que hace posible cambiarlo en el partido.

## GLORY-260 — Cambiar el comportamiento durante el partido
- **Qué hace**: `PUT /matches/{id}/players/{pid}/behavior`.
- **Código clave**: `routes/match_state.change_behavior_on_match_endpoint`; `match_player_service.validate_behavior_change` / `apply_behavior_change`.
- **Idea clave**: el cambio es **temporal** (solo en memoria, no toca `lineup_entries`); un jugador del **rival** devuelve **404 `PLAYER_NOT_FOUND`**; un `COACH` se rechaza ([[GLORY-288 Separar comportamientos de jugador y de DT|GLORY-288]]).

## GLORY-299 — Elegir el DT de la liga
- **Qué hace**: `PUT /leagues/{id}/teams/{tid}/coach` guarda qué comportamiento `COACH` usará el equipo.
- **Código clave**: `routes/leagues.choose_coach_endpoint`; `coach_service.choose_coach_behavior`; `state.CoachRegistry` / `coach_registry`.
- **Idea clave**: el DT se guarda **en memoria** (clave `(league_id, team_id)`), por lo tanto **se pierde si se reinicia el servidor**. Solo con la liga `IN_PROGRESS` y solo comportamientos `COACH`. El detalle de la liga devuelve `coach_behavior_id` **solo al equipo que consulta**.
- **Front**: `CoachSelector` → `chooseLeagueCoach`.
- **Test**: `test_choose_coach.py`.

## GLORY-297 — Ejecutar el DT en el motor
- **Qué hace**: corre el código del DT de cada equipo.
- **Código clave**: `match_start_service.load_coach_behaviors`; `state.coach_on_ticks`, `state.behavior_codes_for_coach`; `coach_api.execute_coach`; `match_runner._run_coaches` (**después de cada tick y al empezar cada pausa**); `run_match(execute_coach=...)`.
- **Idea clave**: **si el código del DT falla, el equipo queda como estaba** y el partido sigue.
- **Test**: `test_execute_coach.py`.
