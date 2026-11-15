---
tags: [glory, desarrollado, epica, motor]
epica: GLORY-182 Refactorización de Motor (+ bugfixes sin épica)
---
# Motor: refactor, física y tests

> Volver a [[00 - Inicio]] · Historia: [[01 - La historia de una liga]]

## La idea
Tres grupos de trabajo sobre el motor de simulación:
1. **Reorganización de carpetas** (épica GLORY-182, "ticket n/7"): el motor quedó en capas numeradas `_01_to_world`, `_02_to_match`, `_03_to_builder`, `_04_to_physics`, `_05_to_docs`.
2. **Bugfixes de física**: rebote de la pelota y saque después de un gol.
3. **Un bugfix de test**: carrera entre sesiones de SQLite en memoria.

## Mis tickets
| Ticket | Qué hace | Tipo |
|---|---|---|
| [[GLORY-95 Unir game_types, geometry_types e intentions en data_types\|GLORY-95]] | Une `game_types`, `geometry_types` e `intentions` en `data_types.py` (1/7) | Refactor |
| [[GLORY-183 Unir constants y timing en metrics\|GLORY-183]] | Une `constants` y `timing` en `metrics.py` (2/7) | Refactor |
| [[GLORY-181 La pelota rebota en los bordes de la cancha\|GLORY-181]] | La pelota **rebota** en los bordes de la cancha | Bugfix |
| [[GLORY-180 Saque después de un gol\|GLORY-180]] | Después de un gol, saca el equipo que lo recibió | Bugfix |
| [[GLORY-224 Fix test_join_friendly_motor\|GLORY-224]] | Fix del test `test_join_friendly_motor` | Bugfix de test |

## Los que revisé de esta zona
[[R4 - Motor refactor#GLORY-184 — `pitch.py`|GLORY-184]] (`pitch.py`, 3/7) y [[R4 - Motor refactor#GLORY-185 — `state.py`|GLORY-185]] (`state.py`, 4/7). Ver [[R4 - Motor refactor]].

## Las capas del motor (para ubicarse)
| Carpeta | Qué hay |
|---|---|
| `_01_to_world` | Reglas y datos del mundo: `data_types.py`, `metrics.py`, `pitch.py` |
| `_02_to_match` | El estado de un partido: `state.py` |
| `_03_to_builder` | Lo que el usuario "ve" desde su código: `PlayerAPI`, `CoachAPI`, `behavior_executor.py` |
| `_04_to_physics` | La física: `simulation.py` |
| `_05_to_docs` | Documentación (`ABOUT_DESIGN.md`) |
| (raíz de `engine/`) | `match_runner.py` (el bucle del partido), `penalties.py` |

## Con mis palabras (completar)
- ¿Por qué se reorganizó el motor?
