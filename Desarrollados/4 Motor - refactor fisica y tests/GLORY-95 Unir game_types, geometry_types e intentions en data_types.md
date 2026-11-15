---
tags: [glory, desarrollado, motor, refactor]
ticket: GLORY-95
epica: Reorganización del motor (GLORY-182) — ticket 1 de 7
tipo: Story
reviewer: Noah Casini
puntos: 2
estado: Done
---
# GLORY-95 — Unir `game_types`, `geometry_types` e `intentions` en `data_types.py`
> Épica: [[Motor - refactor, fisica y tests]] · Siguiente: [[GLORY-183 Unir constants y timing en metrics|GLORY-183]] → [[R4 - Motor refactor#GLORY-184 — `pitch.py`|GLORY-184]] → [[R4 - Motor refactor#GLORY-185 — `state.py`|GLORY-185]]

## En una frase
Refactor puro: tres archivos chicos que definen **tipos de datos** del motor pasan a ser **uno solo**, `engine/_01_to_world/data_types.py`. No cambia el comportamiento.

## Qué contiene `data_types.py`
Los tipos básicos de datos del motor: geometría (`Vector2D`, `Point2D`), `AttributeType` (los atributos de un jugador), `GameState`, `ActionType` y `Intention` (lo que un jugador pide hacer en un tick). `TeamSide` vive en `pitch.py` y `MatchPhase` en `state.py`.

## Decisiones de diseño
- **Por qué unir**: tres archivos tan chicos y tan acoplados se importaban siempre juntos; uno solo reduce imports y el "dónde está X".
- **Por qué sin cambiar comportamiento**: refactor seguro; los tests existentes tienen que seguir pasando **sin modificarse**, salvo los imports.
- **Parte de una serie de 7 tickets** (GLORY-95 es el 1/7): se hizo en tickets chicos para que cada PR sea revisable.

## Qué pasa si falla
Si se me olvidara actualizar algún import, el error aparece al **importar** (ImportError) y los tests lo detectan enseguida: es la ventaja de un refactor mecánico.

## Relacionado con
[[GLORY-183 Unir constants y timing en metrics|GLORY-183]] · [[R4 - Motor refactor#GLORY-184 — `pitch.py`|GLORY-184]] · [[R4 - Motor refactor#GLORY-185 — `state.py`|GLORY-185]] · `src/engine/_05_to_docs/ABOUT_DESIGN.md` (documenta la nueva estructura).

## Preguntas que me pueden hacer
- ¿Por qué reorganizar el motor en capas numeradas?
- ¿Cómo se asegura que un refactor no rompe nada?

## Con mis palabras (completar)
- Qué hice:
