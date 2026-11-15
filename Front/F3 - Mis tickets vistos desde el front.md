---
tags: [glory, front, mapa]
---
# F3 — Mis tickets vistos desde el front
> Volver a [[00 - Inicio]] · Antes: [[F1 - Como funciona el front]] y [[F2 - Como habla el front con el back]]

## Para qué sirve esta nota
Si en la defensa me preguntan "**¿y esto cómo lo usa el front?**" sobre un ticket de back, acá está la respuesta corta con el archivo del front donde se ve.

## Mapa: ticket → qué hace el front con eso
| Ticket | Qué recibe / llama el front | Archivo del front |
|---|---|---|
| [[GLORY-279 Preparar la tanda - evento y sorteo\|GLORY-279]] penales: aviso de inicio/fin | Recibe `PENALTIES_STARTED/FINISHED` y marca el modo penales | `matchSockets.js`; `matchState.js` (`startPenalties`, `finishPenalties`); `isPenalties` |
| [[GLORY-310 Cuadro de preparación antes de cada penal\|GLORY-310]] cuadro de preparación | Recibe un cuadro con los jugadores ubicados y la pelota quieta, y la anima | `Pitch.jsx` (`getViewBox(isPenalties)` muestra solo la mitad izquierda) |
| [[R7 - Penales#GLORY-281 — Marcador de penales\|GLORY-281]] / [[R7 - Penales#GLORY-293 — Exponer el resultado y adaptar la tabla\|GLORY-293]] marcador de penales | Muestra `phome/paway` y quién ganó por penales | `MatchHeader`, `MatchResultPage`, `services/matchResult.js` (`hasPenalties`, `getOutcome`) |
| [[GLORY-216 Lógica de ticks en pausas\|GLORY-216]] pausas | Recibe `PAUSE_STARTED`, muestra una cuenta regresiva local | `matchState.js` (`startPause`, `countDownPause`, `finishPause`), `PauseMenu` |
| [[GLORY-217 Cambio de posición tras Half-Time\|GLORY-217]] Half-Time (cambio de lados) | **No hace nada especial**: dibuja las posiciones que manda el back | — |
| [[GLORY-214 Endpoint de sustitución - efectuar o planificar\|GLORY-214]] / [[GLORY-215 Función substitute_player en el motor\|GLORY-215]] sustitución | `POST .../substitutions`; con **201** actualiza la alineación, con **202** muestra "planificado" | `PauseMenu`, `SubstitutionSelector`, `services/matches.js` (`requestSubstitution` → `{ planned: status === 202 }`), `matchState.js` (`applySubstitution`, `applyPlannedSubstitution`) |
| [[GLORY-296 Primitivas de director técnico\|GLORY-296]] / [[GLORY-288 Separar comportamientos de jugador y de DT\|GLORY-288]] / [[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor\|GLORY-297]] / [[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga\|GLORY-299]] DT | Permite elegir el DT, filtra comportamientos por tipo y documenta las primitivas | `CoachSelector` (`filterCoachBehaviors`, `chooseLeagueCoach` → `PUT .../coach`), `BehaviorHelp` (`CoachHelp`), `services/behaviors.js`, `TacticalSetup` (solo `PLAYER`) |
| [[R1 - Ligas#GLORY-309 — `league_id` en el resultado final del partido\|GLORY-309]] `league_id` en el resultado | Si el partido es de liga, vuelve a la liga | `MatchResultPage` (`isLeagueMatch`) |
| [[R1 - Ligas#GLORY-312 — Listado de ligas con varios estados\|GLORY-312]] / [[R1 - Ligas#GLORY-270 — `is_member` en el listado de ligas\|GLORY-270]] listado de ligas | Pide `status=PENDING,IN_PROGRESS` y usa `isMember` para ir al lobby | `services/leagues.js` (`fetchLeagues`), `LeagueListPage` |
| [[R5 - Pausas y estado en vivo#GLORY-298 — `/state` con pelota y posiciones\|GLORY-298]] `/state` con pelota y posiciones | Reconstruye la cancha al conectarse | `useMatchLive.resync`, `applyFullState` |
| [[GLORY-311 Botón Volver al Inicio en los lobbies\|GLORY-311]] **mi ticket de front** | Botón "Volver al Inicio" | `LeagueLobbyPage.jsx`, `FriendlyLobby.jsx` |
| [[GLORY-181 La pelota rebota en los bordes de la cancha\|GLORY-181]], [[GLORY-180 Saque después de un gol\|GLORY-180]], [[GLORY-224 Fix test_join_friendly_motor\|GLORY-224]], [[GLORY-183 Unir constants y timing en metrics\|GLORY-183]], [[GLORY-95 Unir game_types, geometry_types e intentions en data_types\|GLORY-95]] | **Nada**: son del motor y de tests | — |

## Detalle de los que más probablemente pregunten

### Sustitución (GLORY-214/215)
1. En la pausa, `PauseMenu` abre `SubstitutionSelector`.
2. El usuario elige quién sale y quién entra → `requestSubstitution(matchId, teamId, in, out)`.
3. El back responde **201** (aplicado) o **202** (planificado). El front lo traduce a `{ planned: true/false }`.
4. Según eso llama `applySubstitution` (cambia la alineación en pantalla) o `applyPlannedSubstitution` (muestra "cambio planificado").
5. Si recarga la página, el plan vuelve en `GET /state` (`planned_substitution`).

### Penales (GLORY-279/280/292/310)
1. `PENALTIES_STARTED` → `startPenalties` activa `isPenalties`.
2. `Pitch` cambia el `viewBox` (SVG) para mostrar solo la **mitad izquierda** (donde se patea).
3. Cada penal llega como cuadros `TICK_UPDATE` normales; el primero ([[GLORY-310 Cuadro de preparación antes de cada penal|GLORY-310]]) tiene todo quieto.
4. `PENALTIES_FINISHED` → `finishPenalties`; después llega `MATCH_FINISHED`.
5. Resultado: `MatchResultPage` muestra el global **y** el marcador de penales.

### DT (GLORY-296/288/299/297)
- El usuario crea un comportamiento de tipo **COACH** en `/behaviors` (el tipo se elige al crear y **no se puede cambiar**: el formulario solo lo muestra al editar).
- En el lobby de la liga en curso, `CoachSelector` lista solo los `COACH` y hace `PUT /leagues/{id}/teams/{tid}/coach`.
- Durante el partido el DT corre **solo en el back**; el front solo ve los efectos (jugadores con otro comportamiento, cambios).

## Cómo es "un ticket de front" (GLORY-311 como ejemplo)
1. Se toca un `.jsx` (estructura), un `.css` (estilo) y un `.test.jsx` (prueba).
2. No hay base de datos ni endpoints nuevos.
3. Se valida con `npm test` y mirando la pantalla con `npm run dev`.
