---
tags: [glory, desarrollado, sustituciones, back, endpoint]
ticket: GLORY-214
epica: Ejecutar cambio en una pausa (GLORY-197)
tipo: Story
puntos: 3
reviewer: Orlando Vigetti
estado: Done
---
# GLORY-214 — Endpoint de sustitución: efectuar o planificar el cambio y reflejarlo en GET /state
> Épica: [[Pausas y sustituciones (GLORY-196 y 197)]] · Bloqueado por [[GLORY-215 Función substitute_player en el motor|GLORY-215]] · Lo consume el front: `PauseMenu` / `SubstitutionSelector`

## En una frase
`POST /api/v1/matches/{matchId}/teams/{teamId}/substitutions`: si el partido está **en pausa**, el cambio se aplica al momento (**201**); si no, queda **planificado** para la próxima pausa (**202**). El plan se ve en `GET /matches/{id}/state`, así el front lo recupera si el usuario recarga.

## Qué pedía el ticket
- Seguir el patrón de `routes/match_state.py`: ruta `async`, base consultada con `run_in_threadpool`.
- **Ningún `await`** entre el chequeo y el cambio sobre `MatchState`.
- Token → `team_id`: 401 si el usuario no existe; 403 si el `teamId` de la URL no es el suyo; 403 si el equipo no juega ese partido.
- 404 `RESOURCE_NOT_FOUND` si el partido no existe o no está en ejecución, o si algún jugador no está en el partido.
- Con pausa: `substitute_player` → 201. Sin pausa: validar los jugadores (sin cambiar nada), guardar en `state.planned_substitutions[team_id]` y responder 202; si ya había un plan, **se reemplaza**.
- Al empezar la pausa el motor aplica cada plan con `substitute_player`; si lanza un `SubstitutionError`, el plan se **descarta** y el partido sigue; el plan siempre se borra.
- `GET /state` suma `planned_substitution` ({`player_in_id`, `player_out_id`} o `null`), **solo del equipo que consulta**.
- Traducir: `InvalidSubstitutionError` → 400 `INVALID_PLAYER_SELECTION`; `SubstitutionLimitError` → 409 `SUBSTITUTION_LIMIT_EXCEEDED`.
- Contrato: se elimina `substitution_id` del 201, se agrega el 202, se elimina el 400 `MATCH_NOT_IN_BREAK`.
- Fuera de este ticket: **cancelar** un plan (solo se pisan).

## Dónde está en el código
| Qué | Archivo | Función |
|---|---|---|
| Ruta | `src/routes/substitution.py` | `substitution_endpoint` |
| Paso 1 (base) | `src/services/substitution_service.py` | `validate_substitution` |
| Paso 2 (memoria) | idem | `apply_substitution`, `SubstitutionResult` (`APPLIED`/`PLANNED`) |
| Plan | `src/engine/_02_to_match/state.py` | `MatchState.planned_substitutions`, `apply_planned_substitutions` |
| Aplicar el plan al empezar la pausa | `src/engine/match_runner.py` | `_run_pause` |
| Chequeo de jugadores compartido | `state.py` | `validate_substitution_players` |
| Plan en `/state` | `src/services/match_state_service.py` | `build_match_state_response` |
| Schemas | `src/schemas/substitution.py`, `match_state.py` | `SubstitutionRequest/Response`, `PlannedSubstitutionOut` |

## Cómo funciona
1. `await run_in_threadpool(validate_substitution, ...)`: consulta la base (lento) en un hilo aparte para no frenar el event loop.
2. **Desde acá, sin `await`**: `apply_substitution(match_id, team_id, player_in_id, player_out_id)`.
3. Busca el partido en `match_registry` (si no está → 404), verifica que el equipo tenga jugadores en él (403) y que ambos jugadores estén (404).
4. Si `state.pause` existe → `substitute_player` → `APPLIED` → 201. Si no → `validate_substitution_players` + guardar plan → `PLANNED` → la ruta cambia el status a 202.
5. Al empezar la próxima pausa, `_run_pause` llama a `apply_planned_substitutions` **antes** de avisar `PAUSE_STARTED`: cuando el front consulta `/state` ya ve la alineación nueva.

## Decisiones de diseño
- **Dos pasos (base / memoria)**: la base es lenta y va en un hilo; el cambio sobre `MatchState` es rápido y **no puede intercalarse con el motor**. Mientras no hay `await`, el event loop no corre el motor.
- **201 vs 202 con el mismo cuerpo**: el cliente sabe si ya se aplicó (201) o quedó para la próxima pausa (202).
- **El plan se valida al pedirlo** (con la alineación actual) **pero se vuelve a validar al aplicarlo**: entre medio puede haber cambiado (por ejemplo, un cambio del DT). Si ya no es válido, se descarta sin error.
- **Un solo plan por equipo y se pisa**: más simple que una cola.
- **Solo se informa el plan propio en `/state`**: no se le muestra al rival.
- **`NotInPauseError` no se traduce**: no puede ocurrir porque solo se llama a `substitute_player` cuando hay pausa.
- **El front recupera el plan al recargar** gracias a `planned_substitution`.

## Qué pasa si falla
| Situación | Resultado |
|---|---|
| Token de un usuario borrado | 401 |
| `teamId` de la URL distinto al del usuario | 403 `FORBIDDEN` |
| Partido inexistente o no en ejecución | 404 `RESOURCE_NOT_FOUND` |
| Equipo que no juega ese partido | 403 |
| Jugador que no está en el partido | 404 |
| Jugadores inválidos (entrante en cancha, etc.) | 400 `INVALID_PLAYER_SELECTION` |
| El equipo ya cambió en esta pausa | 409 `SUBSTITUTION_LIMIT_EXCEEDED` |
| Plan que ya no es válido al empezar la pausa | se descarta, el partido continúa |

## Tests
`tests/unit/test_substitution_endpoint.py`

## Relacionado con
[[GLORY-215 Función substitute_player en el motor|GLORY-215]] · [[GLORY-216 Lógica de ticks en pausas|GLORY-216]] · [[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas|GLORY-218]] ([[R5 - Pausas y estado en vivo]]) · front: `requestSubstitution` devuelve `{ planned }`; `applySubstitution` / `applyPlannedSubstitution` ([[F3 - Mis tickets vistos desde el front]]).

## Preguntas que me pueden hacer
- ¿Por qué 201 y 202?
- ¿Qué pasa si pido el cambio dos veces antes de la pausa? ¿Y si cambió el DT mientras tanto?
- ¿Por qué hay un paso con `run_in_threadpool` y otro sin `await`?
- ¿Cómo se entera el front del plan si recarga la página?
- ¿Por qué solo se informa el plan del equipo que consulta?

## Con mis palabras (completar)
- Qué hice:
- Por qué así:
