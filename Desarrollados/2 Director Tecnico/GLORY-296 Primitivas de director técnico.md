---
tags: [glory, desarrollado, dt, back, motor]
ticket: GLORY-296
epica: Comportamiento de DT (GLORY-282)
tipo: Story
puntos: 3
reviewer: Noah Casini
estado: Done
---
# GLORY-296 — Implementar primitivas de director técnico
> Épica: [[DT (GLORY-282)]] · Bloqueado por [[GLORY-215 Función substitute_player en el motor|GLORY-215]] (necesita `substitute_player`) · Reportado por Alexander, implementado por mí

## En una frase
Crea la clase `CoachAPI`: la "interfaz" que ve el código del DT dentro de su `on_tick`. Tiene dos acciones (`change_behavior`, `substitute`) y cuatro lecturas.

## Qué pedía el ticket
- `CoachAPI` en `src/engine/_03_to_builder/coach_api.py`.
- `change_behavior(player_id, behavior_id)` y `substitute(player_out_id, player_in_id)`.
- Síncronas, trabajan directo sobre el estado del partido en memoria (sin endpoints). El DT solo actúa sobre **su propio equipo**.
- Lecturas: tick actual, tipo de pausa (o nada), titulares y suplentes en orden de alineación.
- Corregir el contrato (agregar `CoachAPI` a las primitivas).

## Dónde está en el código
| Qué | Archivo | Detalle |
|---|---|---|
| Clase | `src/engine/_03_to_builder/coach_api.py` | `CoachAPI` |
| Lecturas | idem | `get_current_tick`, `get_pause_type`, `get_starters`, `get_substitutes` |
| Acciones | idem | `change_behavior`, `substitute` |
| Ejecutar el DT de un equipo | idem | `execute_coach(state, team_id)` |
| Cambiar comportamiento | `behavior_executor.py` | `assign_behavior` |
| Sustituir | `src/engine/_02_to_match/state.py` | `substitute_player` ([[GLORY-215 Función substitute_player en el motor\|GLORY-215]]) |

## Cómo funciona
- `CoachAPI(state, team_id, behavior_codes)`: guarda el estado, el equipo del DT y `{behavior_id: código}` de los comportamientos de jugador que puede usar. **El motor no consulta la base**: los códigos ya vienen cargados ([[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor|GLORY-297]]).
- `change_behavior`: si el `behavior_id` no está entre los permitidos → `ValueError`; si el jugador no existe, es de otro equipo o no es titular (`position is None`) → `ValueError`; si todo está bien llama a `assign_behavior`.
- `substitute`: no valida nada propio; deja salir tal cual los errores de `substitute_player` (`NotInPauseError`, `InvalidSubstitutionError`, `SubstitutionLimitError`).
- `execute_coach`: busca el `on_tick` del DT (`state.coach_on_ticks[team_id]`), arma la `CoachAPI` y la llama.

## Decisiones de diseño
- **Síncronas y sin endpoints**: el DT actúa dentro del motor, entre ticks; nada de HTTP.
- **`get_pause_type` devuelve `str`** (`"HALF_TIME"`, `"COOLING_BREAK"`) y no el enum `PauseType`: el código del usuario no puede hacer `import`.
- **"Titular" = tener `position`**: el mismo criterio que `get_starters` y que usa `substitute_player`.
- **El DT solo ve y toca a su equipo**: se valida en `change_behavior` y `substitute_player` recibe el `team_id` del DT.
- **A diferencia de `PlayerAPI`, no registra intenciones**: no hay un "resultado" que aplicar después; cada primitiva modifica el estado en el momento.
- **Segunda capa de seguridad** de [[GLORY-288 Separar comportamientos de jugador y de DT|GLORY-288]]: un comportamiento `PLAYER` no puede llamar a las primitivas de `CoachAPI` porque esa clase no se le inyecta.

## Qué pasa si falla
- Comportamiento no permitido o jugador inválido → `ValueError`.
- El código del comportamiento nuevo no carga → el error de `assign_behavior` sale **sin capturarse** y el jugador **conserva** el que tenía.
- Sustitución imposible → error del motor y el estado no cambia.
- Esos errores no rompen el partido: `_run_coaches` (en `match_runner.py`) captura cualquier excepción del DT ([[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor|GLORY-297]]).

## Tests
`tests/unit/test_coach_api.py`

## Relacionado con
[[GLORY-215 Función substitute_player en el motor|GLORY-215]] · [[GLORY-288 Separar comportamientos de jugador y de DT|GLORY-288]] · [[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor|GLORY-297]] · [[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga|GLORY-299]] · front: `BehaviorHelp` (ayuda del DT) en [[F3 - Mis tickets vistos desde el front]].

## Preguntas que me pueden hacer
- ¿Por qué el DT puede sustituir solo en pausa si `substitute` no lo valida?
- ¿Qué pasa si el DT intenta usar un comportamiento que no es suyo?
- ¿Por qué `get_pause_type` devuelve un string?
- ¿En qué se diferencia `CoachAPI` de `PlayerAPI`?

## Con mis palabras (completar)
- Qué hice:
- Por qué así:
