---
tags: [glory, desarrollado, sustituciones, back, motor]
ticket: GLORY-215
epica: Ejecutar cambio en una pausa (GLORY-197)
tipo: Story
puntos: 2
reviewer: Orlando Vigetti
estado: Done
---
# GLORY-215 — Implementar función `substitute_player` en el motor
> Épica: [[Pausas y sustituciones (GLORY-196 y 197)]] · Bloqueado por [[GLORY-216 Lógica de ticks en pausas|GLORY-216]] · Reportado por Orlando · Lo usan [[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]] y [[GLORY-296 Primitivas de director técnico|GLORY-296]]

## En una frase
Cambia un titular por un suplente **del mismo equipo** durante una pausa: el suplente ocupa el lugar del titular (mismo rol y posición), el titular va al banco, y desde el tick siguiente el suplente juega con **sus** atributos y **su** comportamiento.

## Qué pedía el ticket
- `substitute_player(state, team_id, player_out_id, player_in_id)` sin valor de retorno.
- Validar **en este orden**: (1) hay pausa → si no, `NotInPauseError`; (2) ambos jugadores son del equipo, el saliente en cancha y el entrante en el banco → si no, `InvalidSubstitutionError`; (3) el equipo no cambió en esta pausa → si no, `SubstitutionLimitError`.
- Si todo está bien: el entrante recibe `position`, `role` e `initial_position` del saliente y queda con `is_starter=True`; el saliente queda con esos tres en `None`, `is_starter=False` y `current_action=IDLE`; se agrega el `team_id` a `state.pause.substituted_team_ids`.
- Los tres errores heredan de una base `SubstitutionError` propia del motor (**no es `AppError`**).
- La función es **síncrona y sin `await`**: el chequeo del límite y el cambio son una sola operación.
- No toca la base ni el contrato: `lineup_entries` queda con la alineación inicial; el partido en memoria es el que cambia.
- Comentario del 04/11: "Se bajó la complejidad de 3 → 2"; implicó más gestión de jugadores y 3 validaciones.

## Dónde está en el código
| Qué | Archivo | Función / clase |
|---|---|---|
| La sustitución | `src/engine/_02_to_match/state.py` | `substitute_player` |
| Validar jugadores sin modificar | idem | `validate_substitution_players` |
| Errores | idem | `SubstitutionError`, `NotInPauseError`, `InvalidSubstitutionError`, `SubstitutionLimitError` |
| Aplicar los planes al empezar una pausa | idem | `apply_planned_substitutions` (de [[GLORY-214 Endpoint de sustitución - efectuar o planificar\|GLORY-214]]) |
| Quién está en cancha | idem | `MatchState.on_pitch_player_ids` (`position is not None`) |

## Decisiones de diseño
- **"En cancha" = tener `position`**: un solo criterio para `on_pitch_player_ids`, `get_starters` y la validación.
- **El orden de las validaciones fija qué error sale** cuando hay más de un problema a la vez (está documentado en el docstring).
- **Sin `await`**: dos pedidos casi simultáneos no pueden pasar los dos el chequeo del límite, porque no hay forma de que el motor intercale otra tarea entre el chequeo y el cambio.
- **El límite vive en `state.pause`** (`substituted_team_ids`): como `pause` es un objeto nuevo en cada pausa ([[GLORY-216 Lógica de ticks en pausas|GLORY-216]]), el límite se reinicia solo.
- **Errores del motor ≠ `AppError`**: el motor no sabe de HTTP; el endpoint ([[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]]) los traduce.
- **No se modifica `lineup_entries`**: el cambio vale solo para ese partido.
- **Validar primero, modificar después**: si algo falla arriba, no se toca nada.

## Qué pasa si falla
| Situación | Error |
|---|---|
| Sin pausa | `NotInPauseError` |
| Entrante ya en cancha / saliente en el banco / jugador de otro equipo o inexistente | `InvalidSubstitutionError` |
| Segundo cambio del mismo equipo en la misma pausa | `SubstitutionLimitError` |
| Cambio de un equipo | el otro equipo **sí** puede cambiar |
| Pausa siguiente | el equipo puede volver a cambiar |

## Tests
`tests/unit/test_substitute_player.py`

## Relacionado con
[[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]] · [[GLORY-296 Primitivas de director técnico|GLORY-296]] · [[GLORY-216 Lógica de ticks en pausas|GLORY-216]]

## Preguntas que me pueden hacer
- ¿Por qué la función no tiene `await`?
- ¿Qué errores puede lanzar y en qué orden se evalúan?
- ¿Se guarda la sustitución en la base?
- ¿Por qué no hay que limpiar el límite al terminar la pausa?

## Con mis palabras (completar)
- Qué hice:
- Por qué así:
