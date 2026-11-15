---
tags: [glory, front, api, websocket]
---
# F2 — Cómo habla el front con el back
> Volver a [[00 - Inicio]] · Antes: [[F1 - Como funciona el front]] · Después: [[F3 - Mis tickets vistos desde el front]]

## Tres canales
| Canal | Para qué | En el front | En el back |
|---|---|---|---|
| **REST** (`fetch`) | Pedir o modificar datos | `services/*.js` + `createhttpService` | `routes/` |
| **WebSocket global** | Avisos de "algo cambió" (liga creada, ranking...) | `globalSocket.js`, `GlobalEventsProvider`, `useGlobalEvent` | `realtime/` (canal global) |
| **WebSocket de un partido** | Cuadros del partido en vivo | `matchSockets.js`, `useMatchLive` | `realtime/match_broadcast.py` |

## 1) REST: `createhttpService`
- Un cliente HTTP propio que **agrega el header** `Authorization: Bearer <token>` a cada request.
- Si el back responde un error, lanza una **`ApiError(status, error, message)`**: `error` es el código de nuestro contrato (`LEAGUE_FINISHED`, `INVALID_PLAYER_SELECTION`...). Por eso los errores del back **tienen que ser estables**: el front los usa.
- **`401`** con `UNAUTHORIZED`/`UNAUTHORIZED_ACCESS` → `expireSession()`.
- **`postWithStatus`**: devuelve también el **status** (201 o 202). Lo usa la sustitución ([[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]]).

## 2) WebSocket global (uno por usuario)
1. El front pide un token de corta vida: `POST /live-token/global`.
2. Abre `ws://.../ws/v1/...` con ese token (**un solo socket para toda la app**).
3. Los componentes se **suscriben** a un evento con `useGlobalEvent(NOMBRE, funcion)`.
4. Si se corta, **reintenta** esperando 1 s, 2 s, 4 s... hasta 30 s.

### El patrón que más se usa: **evento → volver a pedir**
El back **no manda los datos**, manda un aviso chico (`{"event": "LEAGUE_FINISHED", ...}`) y el front **vuelve a llamar al endpoint REST** para traer la versión actual. Ejemplos:
| Pantalla / hook | Eventos que escucha |
|---|---|
| Ligas en curso (`useLeagueInProgressEvents`) | `MATCH_STARTED`, `LEAGUE_MATCH_FINISHED`, `LEAGUE_FINISHED` |
| Lobby de liga (`LeagueLobbyPage`) | `TEAM_JOINED_LEAGUE`, `LEAVE_LOBBY_LEAGUE`, `LEAGUE_INITIATED`, `LEAGUE_CANCELLED` |
| Amistosos (`useFriendlyMatches`) | `FRIENDLY_CREATED`, `FRIENDLY_CANCELLED`, `GUEST_JOINED_FRIENDLY`, `MATCH_STARTED` |
| Ranking (`useGlobalRanking`) | `RANKING_UPDATED` |

**Por qué así**: es simple y **no se pierde nada** (si el front se desconecta, al volver pide el estado actual). Del lado del back, por eso los eventos se publican **después del commit**: cuando el front vuelve a pedir el dato, ya está en la base.

## 3) WebSocket de un partido
1. `POST /matches/{id}/live-token` devuelve una `websocket_url` con token (`requestMatchWebsocketUrl`).
2. `matchSockets.js` abre el socket y llama a callbacks: `onTick`, `onPauseStarted`, `onPauseFinished`, `onPenaltiesStarted`, `onPenaltiesFinished`, `onConnected`, `onDisconnected`, `onFirstConnectionFailed`, `onRejected`.
3. Los mensajes: `TICK_UPDATE` (cada 0,5 s), `PAUSE_STARTED/FINISHED` ([[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas|GLORY-218]]), `PENALTIES_STARTED/FINISHED` ([[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]]), `MATCH_FINISHED`.

### `useMatchLive` (el hook del partido en vivo)
1. Pide en paralelo `GET /matches/{id}/state` y los equipos.
2. Abre el WebSocket.
3. **Al conectarse hace `resync`**: vuelve a pedir `/state` por si se perdió algún cuadro ([[R5 - Pausas y estado en vivo#GLORY-298 — `/state` con pelota y posiciones|GLORY-298]] lo hace posible: la foto incluye pelota y posiciones).
4. En `PAUSE_STARTED` llama a `refreshLineup` (la alineación pudo cambiar por un plan o por el DT).
5. Navega a `/matches/:id/result` al llegar `MATCH_FINISHED` o el último tick — **salvo que esté empatado**, porque pueden venir penales.

### `matchState.js`: funciones puras
El estado del partido en el front se actualiza con **funciones puras** (reciben el estado y el mensaje, devuelven el nuevo): `applyTick`, `applyFullState`, `applySubstitution`, `applyPlannedSubstitution`, `applyLineup`, `applyPlayerDetails`, `startPause` / `countDownPause` / `finishPause`, `startPenalties` / `finishPenalties`.
- Un tick **viejo se descarta** (se compara el número de tick).
- **En penales no se compara el tick** porque el back puede repetirlo (cada turno de la tanda reutiliza ticks).

## Esquema completo de un partido
```
Usuario abre /matches/12
   │ GET /matches/12/state            ← foto inicial (REST)
   │ POST /matches/12/live-token      ← token del socket
   ▼
WebSocket de partido  ◄── TICK_UPDATE cada 0,5 s
   │                 ◄── PAUSE_STARTED / PAUSE_FINISHED
   │                 ◄── PENALTIES_STARTED / PENALTIES_FINISHED
   │                 ◄── MATCH_FINISHED  ──► navega a /matches/12/result
   ▼
Acciones del usuario (REST):
   POST .../substitutions                  ([[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]])
   PUT  .../players/{pid}/behavior         ([[R6 - Comportamientos y DT#GLORY-260 — Cambiar el comportamiento durante el partido|GLORY-260]])
```

## Para la defensa
- Si me preguntan "¿cómo sabe el front que cambió el ranking?": evento global `RANKING_UPDATED` → `useGlobalRanking` vuelve a hacer `GET /ranking`.
- "¿Qué pasa si se corta el WebSocket?": reintenta con espera creciente; al reconectar, `resync` trae el estado actual.
- "¿Por qué dos tipos de WebSocket?": uno para avisos generales (siempre abierto) y otro por partido (solo mientras se mira).
