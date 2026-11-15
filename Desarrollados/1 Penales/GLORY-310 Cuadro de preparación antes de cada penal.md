---
tags: [glory, desarrollado, penales, back, motor, bugfix]
ticket: GLORY-310
epica: Penales (GLORY-277)
tipo: Bug
puntos: 2
reviewer: Orlando Vigetti
estado: Done
---
# GLORY-310 — Bugfix: cuadro de preparación antes de cada penal de la tanda
> Épica: [[Penales (GLORY-277)]] · Toca lo hecho en [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]] y [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]]

## En una frase
Antes de cada patada el cliente recibe **un cuadro con los jugadores ubicados y la pelota quieta en el punto penal**, se espera 1,5 s y recién después se juegan los ticks. Así el front puede animar el recorrido de la pelota.

## El problema
En cada penal "arrancaba de golpe": el primer aviso al cliente salía **después** del primer tick. Con ticks de 0,5 s y un tiro de ~12 m por tick, un penal de 11 m se resolvía en ese primer tick, así que el cliente veía la pelota ya en el arco (o ya atajada), nunca quieta en el punto penal. Además la pausa entre penales ocurría **antes** de ubicar a los jugadores: durante esa pausa se seguía viendo el penal anterior.

## La solución
- Constante nueva `PENALTY_SETUP_SECONDS = 1.5` en `src/engine/_01_to_world/metrics.py`.
- En `_play_penalty` (`src/engine/penalties.py`): **justo después de `prepare_turn`** → `await _notify_safely(...)` con el estado ya preparado → `await asyncio.sleep(setup_seconds)` → recién ahí el bucle de ticks.
- `play_shootout` recibe `penalty_setup_seconds` (default `PENALTY_SETUP_SECONDS`), igual que `penalty_pause_seconds`, y se lo pasa a `_play_penalty`.
- El `try/except` que protege el `notify` se extrae a `_notify_safely`, porque ahora se usa en dos lugares (el aviso de preparación y el de cada tick).

## Qué pasa si falla
Si el aviso de preparación falla, se registra con `logger.exception` y **la tanda continúa** (criterio de aceptación del ticket).

## Cómo lo ve el front
La cancha en penales muestra la mitad izquierda (`Pitch.jsx`, `getViewBox(isPenalties)`), y con el primer cuadro la pelota ya está quieta en el punto penal, así que se ve el recorrido. Ver [[F3 - Mis tickets vistos desde el front]].

## Tests
`tests/unit/test_penalties_shootout.py` (los de la tanda llaman a `notify` antes del primer tick y esperan `setup_seconds`).

## Relacionado con
[[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]] · [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]] · [[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]]

## Preguntas que me pueden hacer
- ¿Por qué el cliente "veía aparecer" la pelota en el arco?
- ¿Por qué 1,5 s? ¿Por qué se saca `_notify_safely`?
- ¿Cambia algo en el contrato o en el front? (No cambia el contrato; el front recibe un cuadro más por penal.)

## Con mis palabras (completar)
- Qué hice:
