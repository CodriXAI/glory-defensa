---
tags: [glory, desarrollado, motor, fisica, bugfix]
ticket: GLORY-181
epica: (sin épica, bugfix)
tipo: Bug
puntos: 3
reviewer: Noah Casini
estado: Done
---
# GLORY-181 — Bugfix: rebote de la pelota cuando sale de la cancha
> Épica: [[Motor - refactor, fisica y tests]] · Reportado por Noah, implementado por mí

## En una frase
Antes, si la pelota salía de la cancha se "pegaba" al borde o desaparecía; ahora **rebota como un espejo** y la rapidez se conserva.

## El problema
La pelota podía terminar fuera de los límites (0–105 en x, 0–60 en y) y el partido quedaba en un estado raro.

## La solución (en `simulation._move_ball`)
Función auxiliar `_bounce(coordinate, velocity, limit)`:
- Si `coordinate < 0` → `-coordinate`, y la velocidad cambia de signo.
- Si `coordinate > limit` → `2*limit - coordinate`, y la velocidad cambia de signo.
- Si está dentro, no cambia nada.
Se aplica a x (con límite = largo) y a y (límite = ancho).

## Dónde está
| Qué | Archivo | Función |
|---|---|---|
| Movimiento de la pelota | `src/engine/_04_to_physics/simulation.py` | `_move_ball` |
| Rebote | idem | `_bounce` |
| Largo y ancho | `src/engine/_01_to_world/pitch.py` | `PITCH_LENGTH`, `PITCH_WIDTH` |

## Decisiones de diseño
- **Una sola reflexión alcanza**: la pelota se mueve como mucho ~15 m por tick y la cancha mide 60 m en su lado más corto, así que no puede salirse dos veces en un tick.
- **Rebote "de espejo"**: la rapidez total se conserva (solo cambia el signo de la componente que rebotó). El test lo verifica.
- **Un gol o una atajada frena la pelota antes de rebotar**: si cruza la línea de gol **entre los palos** es gol; un tiro **fuera de los palos** sí rebota y no es gol.
- **No se agregó una regla de "fuera de juego / saque lateral"**: se mantiene simple (alcance académico).

## Qué pasa si falla
No hay camino de error: es matemática pura sin I/O. Si el límite fuera 0 o negativo el rebote no tendría sentido, pero eso lo fija `pitch.py`.

## Tests
`tests/unit/test_simulation.py`:
- `test_deberia_pelota_que_sale_de_la_cancha_rebotar_en_la_direccion_opuesta`
- `test_deberia_pelota_que_rebota_conservar_la_rapidez`

## Relacionado con
[[GLORY-180 Saque después de un gol|GLORY-180]] (qué pasa después de un gol) · [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]] (en penales la pelota se queda quieta en la línea de gol).

## Preguntas que me pueden hacer
- ¿Por qué alcanza una sola reflexión?
- ¿Qué diferencia hay entre un tiro que sale por la línea de fondo y un gol?
- ¿Cómo se prueba que la rapidez se conserva?

## Con mis palabras (completar)
- Qué hice:
