---
tags: [glory, desarrollado, penales, back, bugfix, proceso]
ticket: GLORY-301
epica: Penales (GLORY-277)
tipo: Bug
puntos: 1
reviewer: Noah Casini
estado: Done
---
# GLORY-301 — Bugfix: correcciones pendientes de ubicar jugadores e inyección de comportamientos
> Épica: [[Penales (GLORY-277)]] · Corrige lo pedido en el review de [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]]

## En una frase
Es un ticket de **proceso**: la rama de [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]] se mergeó a `develop` por error **antes** de aplicar los cambios que pidió la reviewer; este ticket los aplica.

## Qué pasó
- La reviewer había pedido cambios en la PR de la rama `feature_GLORY-280_ubicar_jugadores_inyectar_comportamientos`.
- Por confusión y mal orden de mergeo de ramas, esa rama se fusionó con `develop` sin completar los cambios.
- Solución: abrir un ticket nuevo con las correcciones pendientes.

## Qué se corrigió
1. Aclarar los **tipos en la firma** de `prepare_turn`.
2. Comentario más claro en `prepare_turn` (el de la línea del `position = None`).
3. En los tests, aclarar que al armar el estado con tanda (`_estado_con_tanda()`) **se pisan los valores** que `prepare_shootout()` le pone a los jugadores, para poder testearlos de forma más determinística.
4. **Test nuevo**: con distancia a la pelota de 0,3 m **sí** se puede patear con CONTROL 20.

## Dónde está
- `src/engine/penalties.py` → `prepare_turn`.
- `tests/unit/test_penalties_turn.py`.

## Qué pasa si falla
No aplica a runtime: es refactor de claridad y un test nuevo. El riesgo era de **proceso** (código sin review mergeado).

## Qué aprendí / qué diría en la defensa
- El orden de mergeo importa: una rama dependiente no se mergea hasta tener sus cambios de review.
- Un cambio de review no se pierde: se registra en un ticket (esto) y se trazan con GLORY-301 → GLORY-280.

## Relacionado con
[[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]] · [[Penales (GLORY-277)]]

## Preguntas que me pueden hacer
- ¿Por qué hay un ticket de bugfix que no arregla un bug de ejecución?
- ¿Cómo evitarían que vuelva a pasar? (PR con review aprobado antes del merge, y respetar el orden de dependencias).

## Con mis palabras (completar)
- Qué hice:
