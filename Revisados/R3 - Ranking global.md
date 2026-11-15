---
tags: [glory, revisado, ranking]
---
# R3 — Ranking global de clubes (épica GLORY-202 "Ver ranking de clubes")
> Volver a [[00 - Inicio]] · Antes: [[R2 - Iniciar y correr la liga]]

## La historia
Cada partido de una **liga pública** que termina suma puntos a las **estadísticas globales** del club (guardadas en la tabla `teams`). El ranking global lee esas columnas **ya acumuladas**, ordenadas y paginadas.

| Ticket | Quién | Qué hace |
|---|---|---|
| [[R3 - Ranking global#GLORY-205 — Acumular estadísticas globales\|GLORY-205]] | Orlando | **Acumula** las estadísticas globales al terminar un partido |
| [[R3 - Ranking global#GLORY-204 — Ver el ranking global\|GLORY-204]] | Noah | `GET` del ranking global |

## GLORY-205 — Acumular estadísticas globales
- **Qué hace**: al terminar un partido suma partidos, victorias/empates/derrotas, goles y puntos a los dos equipos.
- **Código clave**: `global_stats_service.add_match_to_global_stats`; llamado desde `match_start_service.save_match_result` **dentro de la misma transacción**; migración Alembic `8f23a963366d_teams_global_stats.py` (agrega las columnas a `teams`).
- **Idea clave**:
  - **Solo ligas públicas** (`league.password_hash is None`); amistosos y ligas privadas no cuentan.
  - **Solo si el partido no estaba ya terminado**: evita sumar dos veces.
  - Reutiliza `get_match_result` y `POINTS` de la tabla de posiciones (una sola regla de puntos).
  - El evento `RANKING_UPDATED` se publica **después del commit**.
- **Test**: `test_global_stats.py`.

## GLORY-204 — Ver el ranking global
- **Qué hace**: `GET /ranking` paginado.
- **Código clave**: `routes/rankings.get_global_ranking_endpoint`, `ranking_service.get_global_ranking` (por defecto página 1, límite 50, **máximo 100**), `team_repository.get_global_ranking_page` (`order_by(points desc, goal_diff desc, team_id)`, con `offset/limit` **dentro de la consulta**).
- **Idea clave**: el orden y la paginación los hace la **base**, no Python. Último desempate: **menor `team_id`**.
- **Front**: `useGlobalRanking` vuelve a pedir el ranking cuando llega `RANKING_UPDATED` ([[F2 - Como habla el front con el back]]).
- **Test**: `test_global_ranking.py`.
