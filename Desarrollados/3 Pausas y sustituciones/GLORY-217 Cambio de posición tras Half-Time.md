---
tags: [glory, desarrollado, pausas, back, motor]
ticket: GLORY-217
epica: Pausas (GLORY-196)
tipo: Story
puntos: 3
reviewer: Noah Casini
estado: Done
---
# GLORY-217 — Cambio de posición de jugadores tras Half-Time
> Épica: [[Pausas y sustituciones (GLORY-196 y 197)]] · Bloqueado por [[GLORY-216 Lógica de ticks en pausas|GLORY-216]]

## En una frase
En el Half-Time los equipos **intercambian de lado**: al terminar la pausa cada equipo defiende el arco contrario, los jugadores quedan en sus posiciones iniciales **espejadas**, la pelota vuelve al centro y los goles siguen sumando al equipo correcto.

## Qué pedía el ticket
- "Espejar" = **cambio de lado + reinicio en las posiciones iniciales espejadas**. Solo espejar posiciones sin invertir `side` haría que los jugadores defiendan el arco equivocado.
- Invertir `side` en **todos** los jugadores, también los suplentes (para que los que entren después ya tengan el lado correcto).
- `position` e `initial_position` pasan a `(105 - x, 60 - y)`, la misma fórmula que usa `starter_position` para AWAY.
- Pelota al centro, velocidad 0.
- **Riesgo**: `run_tick` sumaba a `score.home` cuando `detect_goal` devolvía `HOME`. Con los lados invertidos el gol iría al equipo equivocado → el marcador tiene que seguir al **equipo**, no al lado.
- Contrato: aclarar que el anfitrión defiende x = 0 **en el primer tiempo** (el comentario del ticket incluye el texto de reemplazo).

## Dónde está en el código
| Qué | Archivo | Función |
|---|---|---|
| Intercambio de lados | `src/engine/_02_to_match/state.py` | `swap_sides` |
| Quién la llama | `src/engine/match_runner.py` | `_run_pause` (solo `HALF_TIME`, al final de la pausa) |
| Lado actual del anfitrión | `state.py` | `MatchState.host_side` |
| Gol al equipo correcto | `src/engine/match_runner.py` | `run_tick` (`scoring_side is state.host_side`) |
| Espejo de posiciones | `src/engine/_01_to_world/pitch.py` | `mirror_position`, `opposite_side`, `BALL_INITIAL_POSITION` |

## Cómo funciona `swap_sides`
1. `state.host_side = opposite_side(state.host_side)`.
2. Para cada jugador (también suplentes): `player.side = opposite_side(player.side)`.
3. Si el jugador está en cancha: `initial_position = mirror_position(initial_position)` y `position = initial_position`.
4. Pelota al centro con velocidad cero.

## Decisiones de diseño
- **Se hace al final de la pausa**, no al principio: así los suplentes que entraron durante la pausa también quedan del lado nuevo.
- **`host_side` + comparar `scoring_side is state.host_side`** en vez de comparar con `HOME`: el marcador sigue al equipo. Es la solución al "Riesgo" del ticket.
- **El Cooling Break no cambia lados ni posiciones** (solo el Half-Time).
- **`snapshot` usa `goal_center(side)`**: por eso `team_goal` y `opponent_goal` reflejan el lado nuevo.

## Qué pasa si falla
No hay un camino de error propio: es una operación en memoria, sin `await`, que recorre jugadores. Si el partido tiene un número de ticks sin pausas, nunca se llama.

## Cambio de contrato (se registró en el ticket)
Antes: "el anfitrión defiende x = 0 y ataca hacia x = 105". Ahora: **en el primer tiempo** el anfitrión defiende x = 0; al terminar el Half-Time los equipos intercambian de lado y las posiciones iniciales se espejan; `score` no cambia de criterio (`home` sigue siendo el anfitrión).

## Tests
`tests/unit/test_match_pauses.py`

## Relacionado con
[[GLORY-216 Lógica de ticks en pausas|GLORY-216]] · [[GLORY-180 Saque después de un gol|GLORY-180]] (la pelota después de un gol) · front: el front dibuja con las posiciones que manda el back, no calcula lados ([[F3 - Mis tickets vistos desde el front]]).

## Preguntas que me pueden hacer
- ¿Por qué no alcanza con espejar las posiciones?
- ¿Qué pasaría con el marcador si comparáramos con `TeamSide.HOME`?
- ¿Por qué se cambian también los suplentes?
- ¿Por qué se hace al final de la pausa?

## Con mis palabras (completar)
- Qué hice:
- Por qué así:
