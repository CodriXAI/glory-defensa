---
tags: [glory, desarrollado, penales, back, motor]
ticket: GLORY-280
epica: Penales (GLORY-277)
tipo: Story
puntos: 2
reviewer: Noah Casini
estado: Done
---
# GLORY-280 — Ubicar jugadores e inyectar comportamientos de penales
> Épica: [[Penales (GLORY-277)]] · Depende de [[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]] · Correcciones en [[GLORY-301 Correcciones pendientes de ubicar jugadores|GLORY-301]]

## En una frase
Para cada turno de la tanda, deja en cancha **solo al pateador y al arquero**: el arquero en el arco izquierdo, el pateador a 11 m, la pelota quieta casi pegada al pateador, y cada uno con su comportamiento (Pateador / Arquero).

## Qué pedía el ticket
- Usar el orden sorteado en [[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]].
- Pelota a 0,3 m del pateador (el alcance mínimo con CONTROL 20 es 0,5 m; con más separación un jugador de control bajo nunca patearía).
- Los titulares alternan roles: al preparar un turno hay que **retirar el comportamiento del turno anterior** antes de inyectar el nuevo.
- Fuera de alcance: cómo se juega el turno y cómo se pasa al siguiente (eso es [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]]).

## Dónde está en el código
| Qué | Archivo | Función / constante |
|---|---|---|
| Preparar un turno | `src/engine/penalties.py` | `prepare_turn(state, kicker_team_id, keeper_team_id, round_index)` |
| Comportamientos de penales | `src/penalty_behaviors.py` | `PENALTY_KICKER`, `PENALTY_GOALKEEPER` (texto del código) |
| Inyectarlos | `src/penalty_behaviors.py` | `assign_penalty_kicker`, `assign_penalty_goalkeeper` |
| Asignar la función al jugador | `src/engine/_03_to_builder/behavior_executor.py` | `assign_behavior` |
| Distancias | `src/engine/_01_to_world/pitch.py` | `PENALTY_DISTANCE = 11.0`, `PENALTY_BALL_GAP = 0.3` |
| Arco de los penales | `src/engine/penalties.py` | `PENALTY_GOAL_SIDE = TeamSide.HOME` (arco izquierdo, x=0) |
| Atajada | `src/engine/_04_to_physics/simulation.py` | `_move_ball` (usa `is_goalkeeper`) |

## Cómo funciona `prepare_turn`
1. Toma al pateador y al arquero de los órdenes sorteados, en la posición `round_index`.
2. **Primero inyecta los comportamientos** (`assign_penalty_kicker` / `assign_penalty_goalkeeper`).
3. Saca de la cancha a **todos** los jugadores (`position = None`).
4. Ubica al arquero en el centro del arco y al pateador en `x = 11` (el arco está en x=0).
5. Pone la pelota quieta a `PENALTY_BALL_GAP` del pateador, del lado del arco.

## Decisiones de diseño
- **Inyectar primero, ubicar después**: si el comportamiento falla, el estado no queda a medias (comentario en el código).
- **Los comportamientos de penales no están en la base**: `behavior_id = None`, no aparecen en `GET /behaviors`, se asignan en memoria.
- **`assign_*` pisa `on_tick`, `behavior_id` e `is_goalkeeper`**: así un jugador que fue pateador y ahora ataja (o al revés) queda solo con el comportamiento nuevo. Ese es el criterio de aceptación del ticket.
- **El arquero corre de poste a poste** con `run_to`; **la atajada la hace el motor**, no el texto del comportamiento (`_move_ball` frena la pelota si pasa a menos de `GOALKEEPER_SAVE_RADIUS = 1.0` m de un jugador con `is_goalkeeper`).
- **El pateador patea a potencia 1.0 hacia un lugar del arco elegido al azar**: rincón izquierdo, centro o rincón derecho (`CORNER_OFFSET = 3.0` m desde el centro; los postes están a 3,66 m). Usa `random`, que `load_on_tick` ya le deja cargado.
- **Constantes y arco fijo a la izquierda**: simplifica; los dos equipos patean contra el mismo arco.

## Qué pasa si falla
- Si `assign_behavior` falla (el código no compila / no define `on_tick`) → lanza el error y **el jugador queda como estaba** (primero compila, después asigna).
- El pateador con CONTROL bajo igual puede patear porque la pelota está a 0,3 m (< 0,5 m de alcance mínimo): lo verifica el test agregado en [[GLORY-301 Correcciones pendientes de ubicar jugadores|GLORY-301]].

## Tests
`tests/unit/test_penalties_turn.py` · `tests/unit/test_penalty_behaviors.py` · `tests/unit/test_penalty_goalkeeper.py`

## Relacionado con
[[GLORY-301 Correcciones pendientes de ubicar jugadores|GLORY-301]] (correcciones del review) · [[GLORY-310 Cuadro de preparación antes de cada penal|GLORY-310]] (aviso del cuadro ya ubicado) · [[R6 - Comportamientos y DT#GLORY-259 — Guardar el `on_tick` en cada jugador|GLORY-259]] (`assign_behavior`, cada jugador guarda su `on_tick`).

## Preguntas que me pueden hacer
- ¿Por qué 0,3 m y no 1 m?
- ¿Qué pasa con un jugador que patea en una ronda y ataja en la siguiente?
- ¿Por qué los comportamientos de penales no están en la base?
- ¿Quién ataja realmente, el comportamiento o el motor?

## Con mis palabras (completar)
- Qué hice:
- Por qué así:
