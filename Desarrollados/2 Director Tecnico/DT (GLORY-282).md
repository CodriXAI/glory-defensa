---
tags: [glory, desarrollado, epica, dt]
epica: GLORY-282
---
# Épica Comportamiento de DT (GLORY-282)

> Volver a [[00 - Inicio]] · Historia: [[01 - La historia de una liga]]

## La idea
Además del comportamiento de cada jugador, un equipo de **liga** puede tener un **Director Técnico (DT)**: un comportamiento especial (tipo `COACH`) cuyo código corre **después de cada tick** y **al empezar cada pausa**. Con sus primitivas (`CoachAPI`) puede **cambiar el comportamiento de un titular** y **hacer sustituciones**. Los amistosos no tienen DT.

## Las piezas de la épica
| Pieza | Ticket | Quién |
|---|---|---|
| Tipo de comportamiento `PLAYER`/`COACH` (columna + migración) | GLORY-300 | (no es mío) |
| Primitivas del DT (`CoachAPI`) | [[GLORY-296 Primitivas de director técnico\|GLORY-296]] | **Yo** |
| Separar comportamientos de jugador y de DT | [[GLORY-288 Separar comportamientos de jugador y de DT\|GLORY-288]] | **Yo** |
| Elegir el DT de una liga (`PUT .../coach`) | [[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga\|GLORY-299]] | Revisé |
| Ejecutar el código del DT en el motor | [[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor\|GLORY-297]] | Revisé |

## Cómo encajan
1. El usuario crea un comportamiento `COACH` (solo puede usar primitivas de `CoachAPI`: [[GLORY-288 Separar comportamientos de jugador y de DT|GLORY-288]]).
2. Elige ese DT para su equipo en una liga en curso ([[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga|GLORY-299]]); se guarda en `coach_registry` (memoria).
3. Al arrancar cada partido de la liga, `load_coach_behaviors` carga el DT del equipo y la lista de comportamientos de jugador que puede asignar ([[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor|GLORY-297]]).
4. Durante el partido, `execute_coach` corre el DT con una `CoachAPI` ([[GLORY-296 Primitivas de director técnico|GLORY-296]]).

## Con mis palabras (completar)
- Cómo le explico el DT a alguien que no conoce el proyecto:
