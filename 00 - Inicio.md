---
tags: [glory, indice]
---
# GLORY — Defensa final · Índice
> Defensa: **jueves 19/11/2026**. Proyecto: FutBot (React + Vite · FastAPI · SQLAlchemy 2.0 · WebSockets).
> Yo hice **BACK** (y un ticket de FRONT, [[GLORY-311 Botón Volver al Inicio en los lobbies|GLORY-311]]).

## Cómo usar este vault
1. Leer **una vez** [[01 - La historia de una liga]]: ordena todo en una sola película.
2. Estudiar por épica: primero los que **desarrollé** (profundo), después los que **revisé** (a grandes rasgos).
3. Cuando se me ocurra una duda, buscar el ticket con `Ctrl+O` y escribir el número (`GLORY-292`).
4. Practicar con [[03 - Preguntas tipo de defensa]] (taparlas y responder en voz alta).
5. En cada nota hay un bloque **"Con mis palabras (completar)"**: escribir ahí mi explicación; eso es lo que más ayuda a recordar.

## Dónde mirar cuando me preguntan algo
| Me preguntan... | Voy a... |
|---|---|
| "¿Qué hiciste en este ticket?" | La nota del ticket en `Desarrollados/` |
| "¿Por qué lo hiciste así?" | Sección **Decisiones de diseño** de la nota |
| "¿Y si falla?" | Sección **Qué pasa si falla** |
| "¿Dónde está hecho X?" | Tabla **Dónde está en el código** |
| "¿Cómo lo ve el front?" | [[F3 - Mis tickets vistos desde el front]] |
| "¿Qué es X (for_update, claim, etc.)?" | [[04 - Glosario y patrones]] |
| "Revisaste el ticket Y, ¿qué hacía?" | `Revisados/` → la épica correspondiente |
| "¿Cómo lo probaron?" | [[07 - Tests]] (mapa ticket → archivo de tests) |
| "¿Por qué un `set` / una tupla / un dict ahí?" | [[08 - Estructuras de datos poco convencionales]] |
| Pregunta de equipo (cupos pares, duración, prueba de comportamiento, aislamiento) o no me acuerdo | [[06 - Comodines para la defensa]] |

## Desarrollados (17 tickets)
- **[[Penales (GLORY-277)]]**: [[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]] · [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]] · [[GLORY-301 Correcciones pendientes de ubicar jugadores|GLORY-301]] · [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]] · [[GLORY-310 Cuadro de preparación antes de cada penal|GLORY-310]]
- **[[DT (GLORY-282)]]**: [[GLORY-296 Primitivas de director técnico|GLORY-296]] · [[GLORY-288 Separar comportamientos de jugador y de DT|GLORY-288]]
- **[[Pausas y sustituciones (GLORY-196 y 197)]]**: [[GLORY-216 Lógica de ticks en pausas|GLORY-216]] · [[GLORY-217 Cambio de posición tras Half-Time|GLORY-217]] · [[GLORY-215 Función substitute_player en el motor|GLORY-215]] · [[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]]
- **[[Motor - refactor, fisica y tests]]**: [[GLORY-95 Unir game_types, geometry_types e intentions en data_types|GLORY-95]] · [[GLORY-183 Unir constants y timing en metrics|GLORY-183]] · [[GLORY-181 La pelota rebota en los bordes de la cancha|GLORY-181]] · [[GLORY-180 Saque después de un gol|GLORY-180]] · [[GLORY-224 Fix test_join_friendly_motor|GLORY-224]]
- **Front**: [[GLORY-311 Botón Volver al Inicio en los lobbies|GLORY-311]]

## Revisados (27 tickets)
- [[R1 - Ligas]] — 118, 127, 178, 270, 312, 309, 229, 230
- [[R2 - Iniciar y correr la liga]] — 239, 237, 241, 242, 243, 244
- [[R3 - Ranking global]] — 205, 204
- [[R4 - Motor refactor]] — 184, 185
- [[R5 - Pausas y estado en vivo]] — 218, 298
- [[R6 - Comportamientos y DT]] — 259, 260, 299, 297
- [[R7 - Penales]] — 291, 281, 293

## Notas generales
- [[01 - La historia de una liga]] · [[02 - Tabla de todos los tickets]] · [[03 - Preguntas tipo de defensa]] · [[04 - Glosario y patrones]] · [[05 - Inconsistencias conocidas]]
- **Tests y estructuras**: [[07 - Tests]] (cómo se testeó cada ticket, técnicas y preguntas) · [[08 - Estructuras de datos poco convencionales]] (`set`, `dict`, tuplas, `default_factory`, closures...)
- **Defensa final**: [[06 - Comodines para la defensa]] (cupos pares · prueba de comportamiento · duración 2 y 5 min · "no me acuerdo" · aislamiento del código)
- Front: [[F1 - Como funciona el front]] · [[F2 - Como habla el front con el back]] · [[F3 - Mis tickets vistos desde el front]]

## Mi resumen en una frase (completar)
> Yo me encargué de…
