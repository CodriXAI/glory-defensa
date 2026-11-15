---
tags: [glory, desarrollado, motor, refactor]
ticket: GLORY-183
epica: Reorganización del motor (GLORY-182) — ticket 2 de 7
tipo: Story
reviewer: Noah Casini
puntos: 2
estado: Done
---
# GLORY-183 — Unir `constants` y `timing` en `metrics.py`
> Épica: [[Motor - refactor, fisica y tests]] · Anterior: [[GLORY-95 Unir game_types, geometry_types e intentions en data_types|GLORY-95]] · Siguiente: [[R4 - Motor refactor#GLORY-184 — `pitch.py`|GLORY-184]]

## En una frase
Refactor: las constantes físicas del juego y las de tiempo (ticks, duraciones, pausas) pasan a vivir juntas en `engine/_01_to_world/metrics.py`.

## Qué contiene `metrics.py` hoy
- Física: `MAX_RUN_SPEED_METERS_PER_SECOND`, `MAX_KICK_SPEED_METERS_PER_SECOND`, `reach_distance(control)`, `kick_speed(...)`, `GOALKEEPER_SAVE_RADIUS`.
- Tiempos: `TICK_DURATION_SECONDS = 0.5`, `VALID_MATCH_DURATIONS_MINUTES = (2, 5)`, `total_ticks_for(minutos)` y el **calendario de pausas** (`PauseType`, `PAUSE_DURATION_SECONDS`, `PAUSE_PLAN`, `pause_schedule_for`; [[GLORY-216 Lógica de ticks en pausas|GLORY-216]]).
- Penales: `PENALTY_MAX_TICKS`, `PENALTY_MAX_ROUNDS`, `PENALTY_PAUSE_SECONDS`, `PENALTY_SETUP_SECONDS` ([[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]], [[GLORY-310 Cuadro de preparación antes de cada penal|GLORY-310]]).

## Decisiones de diseño
- **Un solo lugar para los números "mágicos"**: si hay que cambiar una duración o un alcance, se hace acá y no hay que buscarlo en el código.
- **Refactor sin cambio de comportamiento**, igual que [[GLORY-95 Unir game_types, geometry_types e intentions en data_types|GLORY-95]].
- Cuando se agregaron pausas y penales, las constantes nuevas **ya tenían su lugar** (por eso aparecen en este mismo archivo).

## Qué pasa si falla
Un import roto aparece al importar; los tests lo detectan.

## Relacionado con
[[GLORY-95 Unir game_types, geometry_types e intentions en data_types|GLORY-95]] · [[GLORY-216 Lógica de ticks en pausas|GLORY-216]] · [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]]

## Preguntas que me pueden hacer
- ¿Qué hay en `metrics.py` y por qué ahí?
- ¿Dónde cambiaría la duración de una pausa?

## Con mis palabras (completar)
- Qué hice:
