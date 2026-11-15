---
tags: [glory, revisado, motor, refactor]
---
# R4 — Refactorización del motor (épica GLORY-182)
> Volver a [[00 - Inicio]] · Mis tickets de la misma épica: [[GLORY-95 Unir game_types, geometry_types e intentions en data_types|GLORY-95]] y [[GLORY-183 Unir constants y timing en metrics|GLORY-183]] (ver [[Motor - refactor, fisica y tests]])

## La historia
El motor se reorganizó en capas numeradas. La épica se partió en **7 tickets** para que cada PR fuera chica y revisable: yo hice los 1/7 y 2/7; revisé el 3/7 y el 4/7.

| Ticket | Quién | Paso | Qué mueve |
|---|---|---|---|
| [[GLORY-95 Unir game_types, geometry_types e intentions en data_types\|GLORY-95]] | Yo | 1/7 | `game_types`, `geometry_types`, `intentions` → `_01_to_world/data_types.py` |
| [[GLORY-183 Unir constants y timing en metrics\|GLORY-183]] | Yo | 2/7 | `constants`, `timing` → `_01_to_world/metrics.py` |
| [[R4 - Motor refactor#GLORY-184 — `pitch.py`\|GLORY-184]] | Alexander | 3/7 | `pitch.py` → `_01_to_world/pitch.py` |
| [[R4 - Motor refactor#GLORY-185 — `state.py`\|GLORY-185]] | Noah | 4/7 | `state.py` → `_02_to_match/state.py` |

## GLORY-184 — `pitch.py`
- **Qué hace**: mueve y ordena el módulo de la cancha (dimensiones, arcos, posiciones iniciales, `mirror_position`, `kickoff_ball_position`).
- **Código clave**: `src/engine/_01_to_world/pitch.py`.
- **Idea clave**: cuando se hizo el refactor, el resto del motor solo cambió imports; el comportamiento no cambió.

## GLORY-185 — `state.py`
- **Qué hace**: mueve y ordena el **estado del partido** (`MatchState`, `MatchPlayerState`, `Score`, `PauseState`, registros en memoria).
- **Código clave**: `src/engine/_02_to_match/state.py`.
- **Idea clave**: es el archivo central del motor: todo lo que el partido "sabe" en un instante vive ahí. Ver [[04 - Glosario y patrones]] ("estado en memoria vs. base").

## Documentación
`src/engine/_05_to_docs/ABOUT_DESIGN.md` describe la nueva estructura. (Tiene una referencia rota a `ABOUT_FORMULAS`: ver [[05 - Inconsistencias conocidas]].)
