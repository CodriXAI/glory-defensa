---
tags: [glory, desarrollado, tests, bugfix, sqlalchemy]
ticket: GLORY-224
epica: (sin épica, bugfix)
tipo: Bug
puntos: 3
reviewer: Alexander Huaman Rojas
estado: Done
---
# GLORY-224 — Bugfix: carrera de base de datos en `test_join_friendly_motor`
> Reportado por Noah, implementado por mí · **No toca `src/`**, solo el test

## En una frase
El test `test_join_friendly_motor.py` fallaba de forma **intermitente** porque tres sesiones de SQLAlchemy compartían **una sola conexión** de SQLite en memoria. Se arregló dándole al test su **propia fixture `db`**, con una conexión por sesión.

## El problema
- Los tests normales usan SQLite en memoria con `StaticPool`, que mantiene **una única conexión** abierta para todo.
- En este test intervienen **tres sesiones**: la del test, la del request (`get_db`) y la del callback `on_finished` que corre en el hilo del motor al terminar el partido.
- Las tres operaban sobre la misma conexión a la vez → una transacción podía pisar a otra → error intermitente (por eso "a veces pasa").

## Por qué en producción no pasa
En producción se usa **Postgres**, donde cada sesión tiene su propia conexión del pool y las transacciones están aisladas. Es un problema **solo del entorno de test**.

## La solución
En `tests/unit/test_join_friendly_motor.py` (alrededor de la línea 40) se define una fixture `db` propia que crea un engine/conexión **por sesión**, así no se comparten entre el test, el request y `on_finished`.

## Decisiones de diseño
- **Arreglar el test y no el código**: el código funciona, lo que estaba mal era el entorno de prueba.
- **No cambiar la fixture global**: hacerlo afectaría a todos los tests; solo este test usa tres sesiones concurrentes.

## Qué pasa si falla
Si la fixture no aislara las conexiones, el test volvería a ser intermitente.

## Relacionado con
[[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]] (hay un `on_finished` en hilo propio) · bugfix parecido en `ticket-bugfix-carrera-db-test-unirse-motor` (docs del proyecto) · [[04 - Glosario y patrones]] ("una sesión por request").

## Preguntas que me pueden hacer
- ¿Por qué fallaba "a veces"?
- ¿Por qué en producción no pasa?
- ¿Por qué no se arregló en el código de `src`?
- ¿Qué es `StaticPool`?

## Con mis palabras (completar)
- Qué hice:
