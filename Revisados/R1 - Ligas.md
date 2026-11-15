---
tags: [glory, revisado, ligas]
---
# R1 — Ligas: crear, unirse, listar, ver detalle y alineación
> Volver a [[00 - Inicio]] · Historia: [[01 - La historia de una liga]] · Lo que sigue: [[R2 - Iniciar y correr la liga]]

## La historia
Antes de jugar una liga hay que **crearla** (con capacidad, privacidad y alineación), **unirse** (con contraseña si es privada), **verla en un listado** y, mientras está pendiente o en curso, **ver y cambiar la alineación**. Estos tickets son esa base.

## Mapa rápido
| Ticket | Épica | Quién lo hizo | Qué hace en una línea |
|---|---|---|---|
| [[R1 - Ligas#GLORY-118 — Modelo y persistencia de liga\|GLORY-118]] | GLORY-78 Crear Liga | Noah | Modelos y tablas `League` y `LineupEntry` |
| [[R1 - Ligas#GLORY-127 — Verificación de contraseña de la liga\|GLORY-127]] | GLORY-80 Unirse a Liga | Noah | Verificar la contraseña de una liga privada → token de unión |
| [[R1 - Ligas#GLORY-178 — Ver el detalle de una liga\|GLORY-178]] | (sin épica) | Fernando | Ver el detalle de una liga |
| [[R1 - Ligas#GLORY-270 — `is_member` en el listado de ligas\|GLORY-270]] | GLORY-199 Lobby de Liga en curso | Maximo | `is_member` en el listado de ligas |
| [[R1 - Ligas#GLORY-312 — Listado de ligas con varios estados\|GLORY-312]] | GLORY-199 | Orlando | Filtrar el listado por varios estados |
| [[R1 - Ligas#GLORY-309 — `league_id` en el resultado final del partido\|GLORY-309]] | GLORY-199 | Orlando | `league_id` en el resultado de un partido |
| [[R1 - Ligas#GLORY-229 — Ver la alineación vigente de una liga\|GLORY-229]] | GLORY-228 Cambiar alineación en Ligas | Fernando | Ver la alineación vigente en una liga |
| [[R1 - Ligas#GLORY-230 — Cambiar la alineación vigente\|GLORY-230]] | GLORY-228 | Fernando | Cambiar la alineación vigente |

---
## GLORY-118 — Modelo y persistencia de liga
- **Qué hace**: define las tablas de la liga y de las alineaciones (`lineup_entries`) y sus **restricciones en la base**.
- **Código clave**: `src/models/league.py` (`CheckConstraint` `capacity_valid`: `max_teams IN (4,6,8,10,12)`; `duration_positive`), `src/models/lineup_entry.py` (XOR `either_match_id_or_league_id_for_lineup`; únicos `uq_lineup_entries_league_player` y `uq_lineup_entries_league_team_role`; `only_starter_has_role`).
- **Idea clave**: **no existe una tabla `league_teams`**: un equipo está inscripto si tiene `lineup_entries` con ese `league_id`. Una alineación pertenece a **un partido o a una liga, nunca a ambos** (XOR).
- **Por qué capacidades pares (4–12)**: se registró en la aclaración del 08/10; ver la defensa técnica en [[03 - Preguntas tipo de defensa]].
- **Tests**: `test_league_persistence.py`, `test_lineup_entry_persistence.py`.

## GLORY-127 — Verificación de contraseña de la liga
- **Qué hace**: `POST` que recibe la contraseña de una liga **privada** y devuelve un **token de unión**.
- **Código clave**: `league_service.check_league_password`; `core/security.create_join_token` (vale **300 s** y está atado a esa liga y a ese usuario).
- **Errores**: `LEAGUE_NOT_PRIVATE` 400 · `PASSWORD_INCORRECT` 401 · `LEAGUE_NOT_FOUND` 404.
- **Idea clave**: la contraseña se compara contra el **hash** (`password_hash`); el token evita volver a mandar la contraseña al unirse.
- **Test**: `test_league_join_token.py`.

## GLORY-178 — Ver el detalle de una liga
- **Qué hace**: `GET /leagues/{id}`: datos de la liga y los equipos inscriptos.
- **Código clave**: `league_service.get_league_detail`, `league_repository.get_league_teams`.
- **Idea clave**: una liga **privada** devuelve **403** a quien no es miembro. Orden de validación: **401 → 404 → 403** (primero el token, después que exista, después que pueda verla).
- **Después se amplió**: `coach_behavior_id` se devuelve solo al equipo que consulta ([[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga|GLORY-299]]).

## GLORY-270 — `is_member` en el listado de ligas
- **Qué hace**: cada liga del listado trae `is_member` (si mi equipo ya está inscripto). El front usa esto para redirigir directo al lobby.
- **Código clave**: `league_repository.get_member_league_ids` (**una sola consulta por página**, con `.distinct()`); `list_leagues` agrega el campo.
- **Idea clave**: evita el problema "N+1" (una consulta por liga).
- **Front**: `LeagueListPage` usa `isMember` ([[F3 - Mis tickets vistos desde el front]]).
- **Test**: `test_list_leagues_endpoint.py`.

## GLORY-312 — Listado de ligas con varios estados
- **Qué hace**: `GET /leagues?status=PENDING,IN_PROGRESS`.
- **Código clave**: `list_leagues` parte `status` por coma y lo convierte en `LeagueStatus`; `get_leagues` filtra con `in_`. Un valor inválido → **400 `INVALID_QUERY_PARAMS`**.
- **Front**: `services/leagues.js → fetchLeagues`.

## GLORY-309 — `league_id` en el resultado final del partido
- **Qué hace**: `GET /matches/{id}/result` devuelve `league_id` (**`null` si es amistoso**).
- **Código clave**: `routes/match_result.get_match_result_endpoint`, `schemas/match_result.MatchResultResponse`.
- **Front**: `MatchResultPage` usa `isLeagueMatch` para mandar al usuario de vuelta a la liga.
- **Test**: `test_match_result_endpoint.py`.

## GLORY-229 — Ver la alineación vigente de una liga
- **Qué hace**: `GET` de la alineación del equipo en esa liga.
- **Código clave**: `league_service.get_league_lineup`, `lineup_service.build_lineup_config` (**la formación se deduce de los roles de los titulares**, no se guarda).
- **Test**: `test_get_league_lineup_endpoint.py`.

## GLORY-230 — Cambiar la alineación vigente
- **Qué hace**: `PUT` que reemplaza la alineación del equipo en la liga.
- **Código clave**: `league_service.update_league_lineup`, `lineup_service.replace_league_lineup` (**borra y vuelve a guardar en una sola transacción**).
- **Idea clave**: un `for_update` bloquea la liga mientras se cambia; si está `FINISHED` → **409 `LEAGUE_FINISHED`**; no se emite evento WS. Un partido en curso **no se ve afectado** porque la alineación se **copia al partido** al arrancar (`copy_league_lineup_to_match`, ver [[R2 - Iniciar y correr la liga#GLORY-241 — Arrancar un partido de liga|GLORY-241]]).
- **Test**: `test_update_league_lineup_endpoint.py`.
