---
tags: [glory, desarrollado, front, react]
ticket: GLORY-311
epica: (sin épica)
tipo: Bug (FRONT)
puntos: 1
reviewer: Maximo Emanuel Nievas Greiff
estado: Done
---
# GLORY-311 — Front: botón "Volver al Inicio" en los lobbies
> **Mi único ticket de FRONT.** Para entender el contexto leer antes [[F1 - Como funciona el front]] y [[F2 - Como habla el front con el back]].

## En una frase
Se agrega un enlace **"Volver al Inicio"** (que lleva a `/`) en la **sala de espera de liga** y en el **lobby de amistoso**, para que el usuario no quede atrapado en esas pantallas.

## Qué se cambió (4 archivos + tests)
| Archivo | Cambio |
|---|---|
| `src/pages/LeagueLobbyPage/LeagueLobbyPage.jsx` | `<Link to="/" className="league-lobby__back">Volver al Inicio</Link>` al final de `<main>` |
| `src/pages/LeagueLobbyPage/LeagueLobbyPage.css` | reglas `.league-lobby .league-lobby__back` |
| `src/pages/FriendlyLobby/FriendlyLobby.jsx` | lo mismo con `friendly-lobby__back` |
| `src/pages/FriendlyLobby/FriendlyLobby.css` | reglas `.friendly-lobby__back` |
| `src/pages/LeagueLobbyPage/LeagueLobbyPage.test.jsx` / `FriendlyLobby.test.jsx` | 5 tests nuevos |

## Cómo funciona (en palabras de back)
- `Link` de `react-router-dom` es un `<a>` que **no recarga la página**: cambia la URL y React renderiza la otra pantalla. Es **solo navegación del cliente**: no llama al back.
- Está **fuera** del condicional `isWaitingRoom` (la pantalla tiene varios estados), así que aparece en **todos** los estados del lobby.

## Decisiones de diseño
- **`Link` y no `useNavigate`** (que es la forma "por código"): no hace falta ninguna lógica, es un enlace; es lo más simple y accesible.
- **Selector CSS `.league-lobby .league-lobby__back`**: existe una regla `.league-lobby a` que le daba otro estilo a los enlaces; el selector más específico hace que gane el estilo del botón.
- **No se llama a ningún endpoint** (no "sale" de la liga): volver al inicio **no te saca** de la liga; para eso existe otro botón.

## Qué pasa si falla
- No hay ruta de error: si `/` no existiera, `react-router` mostraría la pantalla vacía. `/` sí existe (la `Home`, ver `App.jsx`).
- Si el usuario no está logueado, `ProtectedRoute` lo manda a `/login` ([[F1 - Como funciona el front]]).

## Tests (vitest + Testing Library)
En `LeagueLobbyPage.test.jsx` y `FriendlyLobby.test.jsx`: que el enlace exista en cada estado y apunte a `/`.

## Cómo se relaciona con mis tickets de back
No se relaciona de manera directa. Sirve para entender cómo es un ticket de front: **JSX + CSS + test**, sin back.

## Preguntas que me pueden hacer
- ¿Qué es un `Link` y en qué se diferencia de un `<a>`?
- ¿Por qué el botón aparece en todos los estados?
- ¿Llama al back? ¿Qué pasa con la liga en la que estás?
- ¿Cómo lo probaron?

## Con mis palabras (completar)
- Qué hice:
