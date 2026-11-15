---
tags: [glory, desarrollado, motor, fisica, bugfix]
ticket: GLORY-180
epica: (sin épica, bugfix)
reviewer: Noah Casini
tipo: Bug
puntos: 2
estado: Done
---
# GLORY-180 — Bugfix: saque tras un gol (versión simplificada)
> Épica: [[Motor - refactor, fisica y tests]] · Complementa a [[GLORY-181 La pelota rebota en los bordes de la cancha|GLORY-181]]

## En una frase
Después de un gol los jugadores vuelven a su posición inicial y la pelota se pone **cerca del centro, corrida hacia el arco del equipo que recibió el gol** (así le toca sacar a ese equipo). Es una versión simplificada: no hay reglas de saque.

## Dónde está
| Qué | Archivo | Función / constante |
|---|---|---|
| Reiniciar tras gol | `src/engine/_04_to_physics/simulation.py` | `_reset_after_goal` |
| Pelota corrida del centro | `src/engine/_01_to_world/pitch.py` | `kickoff_ball_position(conceding_side)`, `KICKOFF_OFFSET = 14.0` |
| Centro | `pitch.py` | `BALL_INITIAL_POSITION` |

## Cómo funciona
1. `detect_goal` dice qué lado marcó.
2. `_reset_after_goal` pone a cada jugador en `initial_position` (acción `IDLE`).
3. La pelota va a `kickoff_ball_position(conceding_side)`: el centro desplazado `KICKOFF_OFFSET` metros hacia el arco del equipo que **recibió** el gol, velocidad 0.

## Decisiones de diseño
- **Sin reglas de saque** (sin "todos en su mitad", sin "no se puede patear hasta el pitazo"): se corre la pelota hacia el equipo que recibió el gol y el motor sigue; el equipo con la pelota cerca es el que va a jugarla.
- **`conceding_side` y no "el que anotó"**: es más directo leer "defiende ese arco → saca".
- **El tick 0 no cambia**: el comienzo del partido usa `BALL_INITIAL_POSITION` (centro exacto), no esta función.
- **Con Half-Time**: los lados cambian ([[GLORY-217 Cambio de posición tras Half-Time|GLORY-217]]), pero `conceding_side` se calcula con lados actuales, así que sigue siendo correcto.

## Qué pasa si falla
No tiene camino de error propio (cálculo puro).

## Tests
`tests/unit/test_pitch.py` (verifica la posición de la pelota para cada lado).

## Relacionado con
[[GLORY-181 La pelota rebota en los bordes de la cancha|GLORY-181]] · [[GLORY-217 Cambio de posición tras Half-Time|GLORY-217]] · [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]] (en penales no hay saque)

## Preguntas que me pueden hacer
- ¿Quién saca después de un gol y cómo se logra?
- ¿Por qué no se implementaron reglas de saque?
- ¿Qué cambia si el gol ocurre en el segundo tiempo?

## Con mis palabras (completar)
- Qué hice:
