---
tags: [glory, revisado, ligas, iniciar-liga]
---
# R2 — Iniciar y correr la liga (épica GLORY-195 "Iniciar Liga")
> Volver a [[00 - Inicio]] · Historia: [[01 - La historia de una liga]] · Antes: [[R1 - Ligas]] · Después: [[R3 - Ranking global]]

## La historia
Una liga **pendiente** se inicia (se arma el fixture de todos contra todos) → se **arranca una fecha** → dentro de cada fecha se **arrancan sus partidos en paralelo** → cuando termina cada partido se **suman los puntos**, y cuando terminó la fecha se **avanza a la siguiente** → al terminar el último partido se **cierra la liga**. Es la columna vertebral del proyecto.

```
PUT /leagues/{id} ── start_league ──► fixture + LEAGUE_INITIATED
        │
        ▼
start_league_round ──► por cada partido PENDING: start_league_match
        │                         │
        │                         ▼
        │                 begin_league_match (claim) ──► launch_match (corre el motor)
        │                                                     │
        │                                                     ▼
        │                                           on_finished → save_match_result
        ▼                                                     │
advance_league ◄──────────────────────────────────────────────┘
   ├─ quedan partidos en la fecha → espera
   ├─ terminó la fecha → siguiente fecha
   └─ no hay más → finish_league ──► LEAGUE_FINISHED
```

## Mapa rápido
| Ticket | Quién | Qué hace |
|---|---|---|
| [[R2 - Iniciar y correr la liga#GLORY-239 — Iniciar la liga\|GLORY-239]] | Alexander | `PUT /leagues/{id}`: iniciar la liga + evento `LEAGUE_INITIATED` |
| [[R2 - Iniciar y correr la liga#GLORY-237 — Tabla de posiciones\|GLORY-237]] | Alexander | Tabla de posiciones |
| [[R2 - Iniciar y correr la liga#GLORY-241 — Arrancar un partido de liga\|GLORY-241]] | Noah | Arrancar **un** partido de liga |
| [[R2 - Iniciar y correr la liga#GLORY-242 — Arrancar una fecha\|GLORY-242]] | Orlando | Arrancar **una fecha** (varios partidos en paralelo) |
| [[R2 - Iniciar y correr la liga#GLORY-243 — Fin de un partido y avance de fecha\|GLORY-243]] | Alexander | Fin de un partido y avance de fecha |
| [[R2 - Iniciar y correr la liga#GLORY-244 — Cerrar la liga\|GLORY-244]] | Alexander | Cerrar la liga al terminar su último partido |

---
## GLORY-239 — Iniciar la liga
- **Qué hace**: valida y pasa la liga de `PENDING` a `IN_PROGRESS`, genera el fixture y publica `LEAGUE_INITIATED`.
- **Código clave**: `routes/leagues.start_league_endpoint`; `league_service.start_league` / `_start_league` (**la misma función la usa el inicio automático** cuando se llena el cupo en `join_league`); `MIN_TEAMS_TO_START = 4`; errores `InsufficientTeamsError`, `OddTeamsCountError`; `fixture_service.generate_fixture` / `build_round_robin`; `_publish_league_initiated`.
- **Idea clave**: **cambio de estado + fixture en una sola transacción**. Si falla el armado del fixture, la liga sigue `PENDING` sin partidos (no queda a medias). El evento se publica **después del commit**.
- **Tests**: `test_start_league_service.py`, `test_start_league_endpoint.py`.

## GLORY-237 — Tabla de posiciones
- **Qué hace**: calcula la tabla desde los partidos terminados.
- **Código clave**: `standings_service.build_standings`, `standings_sort_key`, `get_match_result`, `POINTS`; `match_repository.get_finished_league_matches`.
- **Idea clave**: **se calcula en cada consulta** (no se guarda); incluye también equipos con 0 partidos. Orden (`standings_sort_key`): puntos, diferencia de gol, goles a favor, menos goles en contra y, como último desempate, **menor `team_id`**.
- **Después se amplió con penales**: [[R7 - Penales#GLORY-293 — Exponer el resultado y adaptar la tabla|GLORY-293]].
- **Test**: `test_standings_service.py`.

## GLORY-241 — Arrancar un partido de liga
- **Qué hace**: toma **un** partido `PENDING` de la liga, arma el motor y lo lanza.
- **Código clave**: `league_run_service.begin_league_match` / `start_league_match`; `match_repository.claim_pending_league_match` (**`UPDATE` condicional** que "reclama" el partido: solo uno lo logra); `lineup_service.copy_league_lineup_to_match`; `match_start_service.prepare_engine` / `launch_match`.
- **Idea clave**: el `claim` evita que dos hilos arranquen el mismo partido. Publica `MATCH_STARTED` con `league_id`. **Si el partido no se puede preparar, se cierra 0-0 `FINISHED`** (1 punto cada uno; cuenta también para el ranking global si la liga es pública).
- **Ojo (inconsistencia conocida)**: los criterios de aceptación del ticket se contradicen ("queda pending" vs "se cierra 0-0"). Ver [[05 - Inconsistencias conocidas]].
- **Test**: `test_launch_league_match.py`.

## GLORY-242 — Arrancar una fecha
- **Qué hace**: arranca **todos los partidos `PENDING` de una fecha** en paralelo.
- **Código clave**: `league_run_service.start_league_round`; `match_repository.get_pending_round_matches`.
- **Idea clave**: **un fallo no frena a los demás** (cada partido en su propia tarea); solo se toman los `PENDING` (así no se duplican).
- **Test**: `test_start_league_round.py`.

## GLORY-243 — Fin de un partido y avance de fecha
- **Qué hace**: al terminar cada partido guarda el resultado y decide si hay que arrancar la próxima fecha.
- **Código clave**: `match_start_service.make_on_finished` / `save_match_result`; `league_run_service.advance_league`, `next_round_to_start`, `_run_in_background`, `_start_round`; `match_repository.count_unfinished_round_matches`, `get_last_round_number`.
- **Idea clave**: publica `MATCH_FINISHED` (con `league_id`) y `LEAGUE_MATCH_FINISHED`. El **import circular** entre `match_start_service` y `league_run_service` se resuelve con un **import local** de `advance_league` dentro de la función.
- **Test**: `test_league_match_finished.py`.

## GLORY-244 — Cerrar la liga
- **Qué hace**: cuando no queda ningún partido, pasa la liga a `FINISHED` y avisa `LEAGUE_FINISHED`.
- **Código clave**: `league_run_service.finish_league`.
- **Idea clave**: **bloquea la liga (`for_update`)**, solo actúa si está `IN_PROGRESS` y publica el evento **una sola vez, después del commit**.
- **Test**: `test_finish_league.py`.
