---
tags: [glory, desarrollado, epica, pausas, sustituciones]
epica: GLORY-196 y GLORY-197
---
# Épicas Pausas (GLORY-196) y Ejecutar cambio / Planear pausa (GLORY-197)

> Volver a [[00 - Inicio]] · Historia: [[01 - La historia de una liga]]

## La idea
El partido tiene **3 pausas** (Cooling Break al 25 %, Half-Time al 50 %, Cooling Break al 75 %). Durante una pausa el reloj se detiene y cada equipo puede hacer **un cambio**. Si el usuario pide el cambio **fuera** de una pausa, queda **planificado** y se aplica al empezar la próxima.

## Mis tickets
| Ticket | Qué hace |
|---|---|
| [[GLORY-216 Lógica de ticks en pausas\|GLORY-216]] | Lógica de pausas en `run_match`: calendario, fase `PAUSED`, reloj detenido |
| [[GLORY-217 Cambio de posición tras Half-Time\|GLORY-217]] | Half-Time: los equipos cambian de lado |
| [[GLORY-215 Función substitute_player en el motor\|GLORY-215]] | `substitute_player`: el cambio dentro del motor |
| [[GLORY-214 Endpoint de sustitución - efectuar o planificar\|GLORY-214]] | Endpoint `POST .../substitutions`: aplicar (201) o planificar (202) |

## Tickets de esta zona que revisé
[[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas|GLORY-218]] (avisos WS `PAUSE_STARTED/FINISHED` y `GET /state` con la pausa) · [[R5 - Pausas y estado en vivo#GLORY-298 — `/state` con pelota y posiciones|GLORY-298]] (`GET /state` con pelota y posiciones). Ver [[R5 - Pausas y estado en vivo]].

## Orden lógico
[[GLORY-216 Lógica de ticks en pausas|GLORY-216]] → [[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas|GLORY-218]] (avisos) → [[GLORY-217 Cambio de posición tras Half-Time|GLORY-217]] (lados) → [[GLORY-215 Función substitute_player en el motor|GLORY-215]] (sustituir) → [[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]] (endpoint y plan) → [[GLORY-296 Primitivas de director técnico|GLORY-296]] / [[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor|GLORY-297]] (el DT sustituye).

## Con mis palabras (completar)
- ¿Cómo explico las pausas y las sustituciones en 1 minuto?
