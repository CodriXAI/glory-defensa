---
tags: [glory, desarrollado, dt, back, validaciones]
ticket: GLORY-288
epica: Comportamiento de DT (GLORY-282)
tipo: Story
puntos: 2
reviewer: Noah Casini
estado: Done
---
# GLORY-288 — Separar los comportamientos de jugador y de DT
> Épica: [[DT (GLORY-282)]] · Bloqueado por GLORY-300 (tipo de comportamiento) y [[GLORY-296 Primitivas de director técnico|GLORY-296]]

## En una frase
Un comportamiento de jugador y uno de DT usan **primitivas distintas** y no deben mezclarse: se rechaza el código que usa primitivas del otro tipo y los comportamientos de DT en lugares donde solo van los de jugador.

## Qué pedía el ticket
- Al **crear o editar** un comportamiento, además de las validaciones que ya existían, revisar las primitivas: un `COACH` solo puede usar las de `CoachAPI`; un `PLAYER` no puede usar ninguna de `CoachAPI`. Al editar se usa el tipo que el comportamiento ya tiene.
- Si usa una primitiva ajena → `INVALID_DATA`, como el resto de errores de código.
- Es la **primera capa**; la segunda: en ejecución cada clase solo tiene sus propias primitivas.
- Las **alineaciones** (crear/unirse a liga o amistoso) solo aceptan `PLAYER`; si hay un `COACH` → mismo error de comportamiento no válido (`INVALID_CONFIGURATION`).
- El **cambio de comportamiento durante el partido** solo acepta `PLAYER`; si es `COACH` se rechaza y el jugador conserva el suyo.
- Fuera de alcance: que el DT elegido para una liga sea `COACH` ([[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga|GLORY-299]]), la prueba de comportamiento y el front.

## Dónde está en el código
| Qué | Archivo | Función |
|---|---|---|
| Validación del código (AST) | `src/services/behavior_service.py` | `validate_behavior_code(code, behavior_type)` |
| Primitivas de cada API | idem | `_primitives_of`, `PLAYER_PRIMITIVES`, `COACH_PRIMITIVES` |
| Crear / editar | idem | `create_behavior`, `update_behavior` (usa `behavior.behavior_type`) |
| Alineaciones | `src/services/lineup_service.py` | `_check_behaviors` (solo `PLAYER` y propio/predeterminado) |
| Cambio en partido | `src/services/match_player_service.py` | `validate_behavior_change` (rechaza `COACH` con `InvalidData`) |
| Tipo | `src/models/behavior.py` | `BehaviorType` (`PLAYER`, `COACH`) |

## Cómo funciona la validación
1. `ast.parse(code)`: si hay `SyntaxError` → `InvalidData`.
2. Si el texto contiene `"__"` → `InvalidData` (no se permiten `__import__`, `__builtins__`, etc.).
3. Se recorre el árbol (`ast.walk`) y se rechaza `for`, `while`, `import`, `from ... import` y comprehensions.
4. Si el nodo es un `ast.Attribute` cuyo nombre está entre las primitivas **prohibidas** para ese tipo → `InvalidData`. Esto atrapa tanto `api.run_to(...)` como `f = api.run_to`.
5. Por último, tiene que existir `def on_tick` en el primer nivel del código.

## Decisiones de diseño
- **Las primitivas se sacan de las clases** (`vars(PlayerAPI)` / `vars(CoachAPI)`), no de una lista escrita a mano: si se agrega una primitiva, la validación la toma sola.
- **Validar con el tipo guardado al editar**: el tipo no se puede cambiar, así que el pedido no puede "engañar" a la validación.
- **Dos capas**: validación al guardar (rápida, con AST, sin ejecutar nada) y límite en ejecución (cada API solo tiene sus métodos).
- **Mismo código de error que ya existía** (`INVALID_DATA` / `INVALID_CONFIGURATION`) para no inventar errores nuevos.
- **El front también filtra** (`filterPlayerBehaviors`, `filterCoachBehaviors`), pero el back **no confía** en el front.

## Qué pasa si falla
- COACH con una primitiva de jugador → `INVALID_DATA`, no se guarda ni se modifica.
- PLAYER con una primitiva de DT → `INVALID_DATA`.
- Alineación con un COACH → `INVALID_CONFIGURATION`.
- Cambio durante el partido a un COACH → `InvalidData("No se pudo cambiar el comportamiento")` y el jugador conserva el suyo.

## Tests
`tests/unit/test_validate_behaviors.py` · `tests/unit/test_behavior_type.py` · `tests/unit/test_lineup_service.py`

## Relacionado con
[[GLORY-296 Primitivas de director técnico|GLORY-296]] · [[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga|GLORY-299]] · [[R6 - Comportamientos y DT#GLORY-260 — Cambiar el comportamiento durante el partido|GLORY-260]] (cambio durante el partido) · [[R6 - Comportamientos y DT#GLORY-259 — Guardar el `on_tick` en cada jugador|GLORY-259]]

## Preguntas que me pueden hacer
- ¿Por qué validar con AST y no ejecutar el código para probarlo?
- ¿Qué pasa si alguien guarda `f = api.substitute` dentro de un comportamiento de jugador?
- ¿Por qué el back vuelve a validar si el front ya filtra?
- ¿Por qué no se pueden usar `for` ni `while`?

## Con mis palabras (completar)
- Qué hice:
- Por qué así:
