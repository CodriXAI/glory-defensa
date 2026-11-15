---
tags: [glory, desarrollado, epica, penales]
epica: GLORY-277
---
# Épica Penales (GLORY-277)

> Volver a [[00 - Inicio]] · Historia completa: [[01 - La historia de una liga]]

## La idea en 3 líneas
Si un **partido de liga con penales habilitados** termina empatado en el tiempo reglamentario, el motor juega una **tanda de penales** antes de pasar a `FINISHED`. Cada ronda es un penal de cada equipo, hay un máximo de 3 rondas y, si siguen empatados, se **sortea** al ganador. El resultado se guarda una sola vez, junto con el marcador del partido.

## Mis tickets de esta épica
| Ticket | Qué hace | Nota |
|---|---|---|
| [[GLORY-279 Preparar la tanda - evento y sorteo\|GLORY-279]] | Prepara la tanda: sortea quién patea y quién ataja, y define los avisos `PENALTIES_STARTED/FINISHED` | Se ejecuta una sola vez |
| [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales\|GLORY-280]] | Ubica al arquero, al pateador y a la pelota, e inyecta sus comportamientos | `prepare_turn` |
| [[GLORY-292 Ejecutar la tanda de penales en el motor\|GLORY-292]] | Juega la tanda completa ronda a ronda dentro de `run_match` | El más grande (5 pts) |
| [[GLORY-301 Correcciones pendientes de ubicar jugadores\|GLORY-301]] | Correcciones pendientes del review de 280 | Ticket de proceso |
| [[GLORY-310 Cuadro de preparación antes de cada penal\|GLORY-310]] | Cuadro de preparación antes de cada penal | Bugfix visual |

## Tickets de la épica que revisé
[[R7 - Penales#GLORY-291 — Opción de penales en la liga|GLORY-291]] (opción `penalties_enabled` en la liga) · [[R7 - Penales#GLORY-281 — Marcador de penales|GLORY-281]] (marcador `phome`/`paway`) · [[R7 - Penales#GLORY-293 — Exponer el resultado y adaptar la tabla|GLORY-293]] (exponer el resultado y adaptar la tabla). Ver [[R7 - Penales]].

## Orden en que se encadenan (de cómo llega un partido a penales)
1. [[R7 - Penales#GLORY-291 — Opción de penales en la liga|GLORY-291]]: la liga se crea con `penalties_enabled`.
2. `prepare_match` le pasa ese valor al estado del partido (`MatchState.penalties_enabled`).
3. [[R7 - Penales#GLORY-281 — Marcador de penales|GLORY-281]]: `Score` tiene `phome`/`paway` (en `None` si no hay penales).
4. Termina el tiempo con empate, `run_match` avisa [[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]] `PENALTIES_STARTED`.
5. `play_shootout` ([[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]]) → por cada penal `prepare_turn` ([[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]]) → aviso de preparación ([[GLORY-310 Cuadro de preparación antes de cada penal|GLORY-310]]) → ticks.
6. `PENALTIES_FINISHED`, `FINISHED`, `save_match_result` guarda goles y penales ([[R7 - Penales#GLORY-293 — Exponer el resultado y adaptar la tabla|GLORY-293]]).

## Con mis palabras (completar)
- Cómo le explicaría la tanda a un profesor en 1 minuto:
- Qué parte me cuesta más recordar:
