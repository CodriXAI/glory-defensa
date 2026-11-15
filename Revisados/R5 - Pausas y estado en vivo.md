---
tags: [glory, revisado, pausas, estado-en-vivo]
---
# R5 — Pausas y estado en vivo
> Volver a [[00 - Inicio]] · Mis tickets de pausas: [[Pausas y sustituciones (GLORY-196 y 197)]]

## La historia
Cuando el motor entra en una pausa el front tiene que enterarse (**evento WS**) y, si recarga la página, tiene que poder **reconstruir todo** con `GET /matches/{id}/state`. Estos dos tickets cubren esas dos mitades: avisos en vivo y foto del estado.

| Ticket | Quién | Qué hace |
|---|---|---|
| [[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas\|GLORY-218]] | Alexander | Eventos WS `PAUSE_STARTED` / `PAUSE_FINISHED` + pausa en `/state` |
| [[R5 - Pausas y estado en vivo#GLORY-298 — `/state` con pelota y posiciones\|GLORY-298]] | Maximo | `/state` devuelve la pelota y la posición de los jugadores |

## GLORY-218 — Eventos WS de las pausas
- **Qué hace**: el motor avisa cuando empieza y cuando termina cada pausa; `/state` informa la fase y la pausa en curso.
- **Código clave**: `realtime/match_broadcast.py` → `build_pause_started_payload` (`{"event":"PAUSE_STARTED","match_id","type","duration_seconds","current_tick"}`), `build_pause_finished_payload`, `notify_pause`; `engine/match_runner._notify_pause_safely`; `services/match_state_service.build_match_state_response` (devuelve `phase` y `pause{type, remaining_seconds}`, con `remaining_seconds = max(0, ends_at − monotonic)`).
- **Idea clave**: si el aviso falla, el partido **sigue** (`_notify_pause_safely`). `remaining_seconds` nunca es negativo.
- **Front**: `startPause`/`finishPause`/`countDownPause` y `PauseMenu` ([[F3 - Mis tickets vistos desde el front]]).
- **Tests**: `test_match_broadcast.py`, `test_match_pauses.py`.

## GLORY-298 — `/state` con pelota y posiciones
- **Qué hace**: antes `/state` no devolvía dónde estaba la pelota ni dónde estaban los jugadores, así que un cliente que se conectaba tarde veía una cancha vacía.
- **Código clave**: `schemas/match_state.py` (`PointOut`), `services/match_state_service.build_player_out` y `build_match_state_response`.
- **Idea clave**: se devuelve `ball` y `position` / `current_action` por jugador; **solo se oculta el `behavior_id` del rival**; los suplentes tienen `position = null`.
- **Front**: `applyFullState` y `useMatchLive.resync` reconstruyen la cancha desde esa foto.
- **Test**: `test_match_state_endpoint.py`.
