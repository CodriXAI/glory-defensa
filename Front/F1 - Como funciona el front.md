---
tags: [glory, front, react]
---
# F1 — Cómo funciona el front (para alguien de back)
> Volver a [[00 - Inicio]] · Siguiente: [[F2 - Como habla el front con el back]] → [[F3 - Mis tickets vistos desde el front]]

## Lo mínimo que hay que saber
El front es una **aplicación de una sola página (SPA)** hecha con **React** y empaquetada con **Vite**. El navegador descarga **un solo HTML + JavaScript** una vez; después, al "cambiar de pantalla", **no pide otra página al servidor**: React dibuja otra pantalla en el mismo documento. Los datos los pide al back con `fetch` (REST) y recibe eventos en vivo por WebSocket.

| Tecnología | Para qué | Equivalente mental en el back |
|---|---|---|
| **React 19** | Dibujar la interfaz con componentes | — |
| **Vite** | Servidor de desarrollo y `build` | `uvicorn` + empaquetado |
| **react-router-dom 7** | Rutas del navegador (`/leagues/3`) | `routes/` de FastAPI |
| **vitest + Testing Library** | Tests | `pytest` |
| **oxlint** | Linter | `ruff` / `flake8` |
| **CodeMirror** | Editor de código en la pantalla de comportamientos | — |

## Conceptos de React en 2 minutos
- **Componente**: una función que devuelve JSX (HTML dentro de JS). Ej.: `function Pitch() { return <svg>...</svg> }`.
- **Props**: los **parámetros** de un componente (`<Pitch isPenalties={true} />`).
- **Estado (`useState`)**: una variable que, **cuando cambia, vuelve a dibujar** el componente. Es lo que hace "viva" a la pantalla.
- **Efecto (`useEffect`)**: código que corre **después de dibujar** (pedir datos, abrir un WebSocket) y puede **limpiarse** al salir.
- **Hook propio (`useXxx`)**: función reutilizable que junta estado + efectos. Ej.: `useMatchLive`, `useGlobalEvent`.
- **Contexto (`Context`)**: una variable "global" para todo un árbol de componentes (la sesión del usuario, el WebSocket global).
- **JSX**: `<Link to="/">Volver</Link>` es JS con forma de HTML; se compila a llamadas a funciones.

## Estructura de `src/`
| Carpeta | Qué hay | Analogía |
|---|---|---|
| `pages/` | Una por pantalla (`LeagueLobbyPage`, `MatchPage`, `RankingPage`...) | un endpoint "completo" |
| `components/` | Piezas reutilizables (`Pitch`, `PauseMenu`, `CoachSelector`...) | funciones auxiliares |
| `services/` | Llamadas al back (`leagues.js`, `matches.js`, `behaviors.js`...) y WebSockets | `repositories` hacia afuera |
| `hooks/` | Lógica reutilizable con estado | `services` de negocio del front |
| `context/` | Sesión y WebSocket global | estado compartido |
| `constants/` | Configuración (medidas de la cancha, `FIELD_CONFIG`) | `metrics.py` |

## Cómo arranca (`main.jsx`)
```
BrowserRouter                  ← habilita las rutas
 └─ SessionProvider            ← quién está logueado (token + usuario)
     └─ GlobalEventsProvider   ← el WebSocket global, un solo para toda la app
         └─ App                ← define las rutas
```

## Rutas (`App.jsx`)
- **Públicas (solo si NO estás logueado)** — `PublicOnlyRoute`: `/login`, `/register`.
- **Protegidas (hay que estar logueado)** — `ProtectedRoute`: `/`, `/friendlies`, `/friendlies/:matchId`, `/friendlies/:matchId/join`, `/friendlies/new`, `/leagues`, `/leagues/new`, `/leagues/:leagueId`, `/leagues/:leagueId/verify-password`, `/join`, `/lineup`, `/matches/:matchId`, `/matches/:matchId/result`, `/players`, `/players/createplayer`, `/behaviors`, `/profile`, `/ranking`.
- `:leagueId` / `:matchId` son **parámetros de ruta** (como `{league_id}` en FastAPI).

## La sesión
- Al loguearse, el back devuelve un **token**. El front lo guarda en **`localStorage`** del navegador: `futbot_token`, `futbot_token_type`, `futbot_user`.
- `ProtectedRoute` mira si hay sesión; si no, redirige a `/login`.
- Si cualquier request devuelve **401** (`UNAUTHORIZED` / `UNAUTHORIZED_ACCESS`), el cliente HTTP llama a `expireSession()` → se borra la sesión y se vuelve a `/login`.

## Variables de entorno (`.env`)
```
VITE_API_BASE_URL=http://localhost:8000/api/v1
VITE_WS_BASE_URL=ws://localhost:8000/ws/v1
```
(Las variables de Vite deben empezar con `VITE_` para llegar al navegador.)

## Cómo se prueba
`npm run dev` (servidor local) · `npm test` (vitest) · `npm run build` (empaquetado) · `npm run lint` (oxlint).

## Qué diría si me preguntan "¿por qué React?"
Es el requisito del laboratorio (React + Vite en el front, FastAPI en el back). La separación en servicios, hooks y componentes **espeja** la separación del back (`routes/services/repositories`): cada capa con una sola responsabilidad.
