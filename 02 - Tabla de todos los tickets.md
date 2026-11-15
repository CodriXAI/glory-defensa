---
tags: [glory, indice]
---
# 02 — Tabla de todos los tickets
> Volver a [[00 - Inicio]]. Generada a partir de los CSV de Jira (44 tickets: 17 desarrollados + 27 revisados).

## 🟢 Desarrollados por mí (17)

| Ticket | Tipo | Pts | Épica | Reviewer | Resumen |
|---|---|---|---|---|---|
| [[GLORY-311 Botón Volver al Inicio en los lobbies\|GLORY-311]] | Bug | 1 | — | Maximo Emanuel Nievas Greiff | [FRONT] Bugfix: los lobbies de liga y de amistoso no tienen forma de volver al inicio |
| [[GLORY-310 Cuadro de preparación antes de cada penal\|GLORY-310]] | Bug | 2 | GLORY-277 Penales | Orlando Vigetti | Bugfix: Cuadro de preparación antes de cada penal de la tanda |
| [[GLORY-301 Correcciones pendientes de ubicar jugadores\|GLORY-301]] | Bug | 1 | GLORY-277 Penales | Noah Casini | [BACK] bugfix correcciones pendientes de ubicar jugadores e inyeccion de comportamientos |
| [[GLORY-296 Primitivas de director técnico\|GLORY-296]] | Story | 3 | GLORY-282 Comportamiento de DT | Noah Casini | [BACK] Implementar primitivas de director técnico  |
| [[GLORY-292 Ejecutar la tanda de penales en el motor\|GLORY-292]] | Story | 5 | GLORY-277 Penales | Orlando Vigetti | [BACK] Ejecutar la tanda de penales en el motor |
| [[GLORY-288 Separar comportamientos de jugador y de DT\|GLORY-288]] | Story | 2 | GLORY-282 Comportamiento de DT | Noah Casini | [BACK] Separar los comportamientos de jugador y de DT |
| [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales\|GLORY-280]] | Story | 2 | GLORY-277 Penales | Noah Casini | [BACK] Implementar funciones de Ubicar jugadores e inyectar comportamientos de penales |
| [[GLORY-279 Preparar la tanda - evento y sorteo\|GLORY-279]] | Story | 3 | GLORY-277 Penales | Noah Casini | [BACK] Preparar la tanda de penales: evento y sorteo |
| [[GLORY-224 Fix test_join_friendly_motor\|GLORY-224]] | Bug | 3 | — | Alexander Huaman Rojas | bugfix: el test de unirse con el motor falla de forma intermitente (carrera en la base de test) |
| [[GLORY-217 Cambio de posición tras Half-Time\|GLORY-217]] | Story | 3 | GLORY-196 Pausas (Entretiempo y Cooling Break) | Noah Casini | [BACK] Implementar cambio de posición de jugadores tras Half-Time |
| [[GLORY-216 Lógica de ticks en pausas\|GLORY-216]] | Story | 3 | GLORY-196 Pausas (Entretiempo y Cooling Break) | Alexander Huaman Rojas | [BACK] Implementar lógica de ticks en Pausas Cooling Break y Half-Time |
| [[GLORY-215 Función substitute_player en el motor\|GLORY-215]] | Story | 2 | GLORY-197 Ejecutar Cambio en una Pausa / Planear Pausa | Orlando Vigetti | [BACK]  Implementar función substitute_player en el motor |
| [[GLORY-214 Endpoint de sustitución - efectuar o planificar\|GLORY-214]] | Story | 3 | GLORY-197 Ejecutar Cambio en una Pausa / Planear Pausa | Orlando Vigetti | [BACK] Endpoint de sustitución: efectuar o planificar el cambio y reflejarlo en GET /state |
| [[GLORY-183 Unir constants y timing en metrics\|GLORY-183]] | Story | 2 | GLORY-182 Refactorizacion de Motor | Noah Casini | Refactorizacion de archivos y unificacion en metrics.py |
| [[GLORY-181 La pelota rebota en los bordes de la cancha\|GLORY-181]] | Bug | 3 | — | Noah Casini | bugfix rebote de pelota en cancha |
| [[GLORY-180 Saque después de un gol\|GLORY-180]] | Bug | 2 | — | Noah Casini | bugfix que saque la pelota el equipo al que le metieron gol |
| [[GLORY-95 Unir game_types, geometry_types e intentions en data_types\|GLORY-95]] | Story | 2 | GLORY-182 Refactorizacion de Motor | Noah Casini | Refactorizacion de archivos y unificacion en data_types.py |

## 🔵 Revisados por mí (27)

| Ticket | Tipo | Pts | Épica | Lo hizo | Resumen |
|---|---|---|---|---|---|
| [[R1 - Ligas#GLORY-312 — Listado de ligas con varios estados\|GLORY-312]] | Bug | 2 | GLORY-199 Lobby de Liga en curso | Orlando Vigetti | [BACK] Bugfix: el listado de ligas no permite pedir las ligas pendientes y en curso a la vez |
| [[R1 - Ligas#GLORY-309 — `league_id` en el resultado final del partido\|GLORY-309]] | Bug | 2 | GLORY-199 Lobby de Liga en curso | Orlando Vigetti | [BACK] Bugfix: Devolver league_id en el resultado final del partido |
| [[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga\|GLORY-299]] | Story | 2 | GLORY-282 Comportamiento de DT | Noah Casini | [BACK] Elegir el DT de la liga |
| [[R5 - Pausas y estado en vivo#GLORY-298 — `/state` con pelota y posiciones\|GLORY-298]] | Bug | 1 | — | Maximo Emanuel Nievas Greiff | bugfix: GET /state no devuelve la pelota ni la posición de los jugadores |
| [[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor\|GLORY-297]] | Story | 5 | GLORY-282 Comportamiento de DT | Noah Casini | [BACK] Ejecutar el código de comportamiento del director técnico |
| [[R7 - Penales#GLORY-293 — Exponer el resultado y adaptar la tabla\|GLORY-293]] | Story | 2 | GLORY-277 Penales | Noah Casini | [BACK] Exponer el resultado de penales y adaptar la tabla de posiciones |
| [[R7 - Penales#GLORY-291 — Opción de penales en la liga\|GLORY-291]] | Story | 1 | GLORY-277 Penales | Noah Casini | [BACK] Agregar la opción de penales a la liga |
| [[R7 - Penales#GLORY-281 — Marcador de penales\|GLORY-281]] | Story | 1 | GLORY-277 Penales | Noah Casini | [BACK] Marcador de penales |
| [[R1 - Ligas#GLORY-270 — `is_member` en el listado de ligas\|GLORY-270]] | Bug | 2 | GLORY-199 Lobby de Liga en curso | Maximo Emanuel Nievas Greiff | [BACK] bugfix: Redireccion del listado de ligas al lobby  |
| [[R6 - Comportamientos y DT#GLORY-260 — Cambiar el comportamiento durante el partido\|GLORY-260]] | Story | 3 | GLORY-198 Cambiar comportamiento durante el partido | Noah Casini | [BACK] Implementar endpoint de cambiar comportamiento del jugador durante el partido |
| [[R6 - Comportamientos y DT#GLORY-259 — Guardar el `on_tick` en cada jugador\|GLORY-259]] | Story | 3 | GLORY-198 Cambiar comportamiento durante el partido | Noah Casini | [BACK] Guardar la función de comportamiento en cada jugador del partido |
| [[R2 - Iniciar y correr la liga#GLORY-244 — Cerrar la liga\|GLORY-244]] | Story | 1 | GLORY-195 Iniciar Liga | Alexander Huaman Rojas | [BACK] Cerrar la liga al terminar su último partido  |
| [[R2 - Iniciar y correr la liga#GLORY-243 — Fin de un partido y avance de fecha\|GLORY-243]] | Story | 3 | GLORY-195 Iniciar Liga | Alexander Huaman Rojas | [BACK] Resolver el fin de un partido de liga y avance de fecha |
| [[R2 - Iniciar y correr la liga#GLORY-242 — Arrancar una fecha\|GLORY-242]] | Story | 3 | GLORY-195 Iniciar Liga | Orlando Vigetti | [BACK] Arrancar una fecha de la liga   |
| [[R2 - Iniciar y correr la liga#GLORY-241 — Arrancar un partido de liga\|GLORY-241]] | Story | 3 | GLORY-195 Iniciar Liga | Noah Casini | [BACK] Arrancar un partido de liga  |
| [[R2 - Iniciar y correr la liga#GLORY-239 — Iniciar la liga\|GLORY-239]] | Story | 2 | GLORY-195 Iniciar Liga | Alexander Huaman Rojas | [BACK] Implementar endpoint de iniciar liga y evento LEAGUE_INITIATED |
| [[R2 - Iniciar y correr la liga#GLORY-237 — Tabla de posiciones\|GLORY-237]] | Story | 3 | GLORY-195 Iniciar Liga | Alexander Huaman Rojas | [BACK] Generar tabla de posiciones de la liga |
| [[R1 - Ligas#GLORY-230 — Cambiar la alineación vigente\|GLORY-230]] | Story | 2 | GLORY-228 Cambiar alineación en Ligas | Fernando Lara | [BACK] Endpoint para cambiar la alineación vigente en una liga |
| [[R1 - Ligas#GLORY-229 — Ver la alineación vigente de una liga\|GLORY-229]] | Story | 1 | GLORY-228 Cambiar alineación en Ligas | Fernando Lara | [BACK] Endpoint para ver la alineación vigente en una liga |
| [[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas\|GLORY-218]] | Story | 2 | GLORY-196 Pausas (Entretiempo y Cooling Break) | Alexander Huaman Rojas | [BACK] Eventos WS de las Pausas |
| [[R3 - Ranking global#GLORY-205 — Acumular estadísticas globales\|GLORY-205]] | Story | 2 | GLORY-202 Ver ranking de clubes | Orlando Vigetti | [BACK] Acumular las estadísticas globales del club al terminar un partido |
| [[R3 - Ranking global#GLORY-204 — Ver el ranking global\|GLORY-204]] | Story | 2 | GLORY-202 Ver ranking de clubes | Noah Casini | [BACK] Implementar endpoint de ver ranking |
| [[R4 - Motor refactor#GLORY-185 — `state.py`\|GLORY-185]] | Story | 3 | GLORY-182 Refactorizacion de Motor | Noah Casini | Refactorizacion de state |
| [[R4 - Motor refactor#GLORY-184 — `pitch.py`\|GLORY-184]] | Story | 3 | GLORY-182 Refactorizacion de Motor | Alexander Huaman Rojas | Refactorizacion de pitch |
| [[R1 - Ligas#GLORY-178 — Ver el detalle de una liga\|GLORY-178]] | Story | 2 | — | Fernando Lara | [BACK] Implementar endpoint de ver detalle de liga |
| [[R1 - Ligas#GLORY-127 — Verificación de contraseña de la liga\|GLORY-127]] | Story | 1 | GLORY-80 Unirse a Liga | Noah Casini | [BACK] Implementar endpoint de verificación de contraseña de liga privada |
| [[R1 - Ligas#GLORY-118 — Modelo y persistencia de liga\|GLORY-118]] | Story | 3 | GLORY-78 Crear Liga | Noah Casini | [BACK] Implementar modelo y persistencia de liga en base de datos |

> Total de puntos desarrollados: 1+2+1+3+5+2+2+3+3+3+3+2+3+2+3+2+2 = **42** (verificar con Jira).
