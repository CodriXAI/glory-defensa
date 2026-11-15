---
tags: [glory, glosario, patrones]
---
# 04 — Glosario y patrones que se repiten
> Volver a [[00 - Inicio]]. Si entiendo estos patrones, entiendo el 80 % de mis tickets.

## Patrones del back

### 1. `for_update` (bloqueo de fila)
`session.get(League, id, with_for_update=True)` → la base **bloquea esa fila** hasta el `commit`. Otro request que quiera la misma fila **espera**. Se usa cuando hay "leer → decidir → escribir" y no puede haber dos a la vez (`finish_league`, `update_league_lineup`).

### 2. UPDATE condicional como "reclamo" (*claim*)
```
UPDATE matches SET status='IN_PROGRESS' WHERE match_id=… AND status='PENDING'
```
Si afectó **1 fila**, ese hilo "ganó" el partido. Si afectó 0, otro ya lo tomó. `claim_pending_match`, `claim_pending_league_match`. No necesita bloqueos largos.

### 3. Eventos **después** del commit
Primero se guarda en la base, **después** se avisa por WebSocket. Si fuera al revés, el front volvería a pedir el dato y encontraría el estado viejo (o uno que luego se revierte).

### 4. `notify` que nunca frena el partido
`_notify_safely`, `_notify_pause_safely`, `_notify_penalties_safely`: envuelven el aviso en `try/except` y solo registran el error. Un WebSocket caído **no puede** romper una simulación.

### 5. Parámetros opcionales con default `None`
`run_match(notify_pause=None, notify_penalties=None, execute_coach=None, play_shootout=None…)`. Los tests y usos viejos siguen funcionando sin cambios.

### 6. Validar antes de mutar
`validate_substitution_players` antes de tocar el estado; `assign_behavior` compila **antes** de asignar. Si algo falla, no queda nada a medias.

### 7. Secciones síncronas sin `await` (atomicidad)
En `asyncio` solo se puede intercalar otra tarea en un `await`. Una función **sin** `await` es atómica para el motor (`substitute_player`, `apply_substitution`, `play_tick`).
- Ejemplo: `play_tick` (jugar un tick) es síncrona a propósito: lee y modifica posiciones, pelota e intenciones de todos, y nadie debe ver ese estado a medias. Lo que sí espera (avisos WS, `sleep`) queda **afuera**, en `run_match` o en `_play_penalty`. Detalle en [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]].

### 8. Estado en memoria vs. base
| En memoria | En la base |
|---|---|
| `MatchState` del partido en curso (`match_registry`) | Partidos, resultados, ligas, alineaciones |
| Cambio de comportamiento durante el partido | Comportamientos guardados |
| DT elegido (`coach_registry`) | — |
| Sustitución planificada | — |
Lo que está en memoria **se pierde si se reinicia el servidor**.

### 9. Servicios en dos pasos
Primero lo lento (consulta a la base) con `run_in_threadpool`; después lo rápido y atómico en memoria, **sin `await`** ([[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]]).

### 10. Transacción única
Cambiar estado + generar fixture ([[R2 - Iniciar y correr la liga|GLORY-239]]); borrar + guardar alineación ([[R1 - Ligas|GLORY-230]]); guardar resultado + sumar ranking ([[R3 - Ranking global|GLORY-205]]).

## SQLAlchemy 2.0 (lo mínimo)
| Concepto | Qué es | Ejemplo en el proyecto |
|---|---|---|
| `Base` | Clase madre de los modelos (con convención de nombres de constraints, `NAMING_CONVENTION`) | `src/database/base.py` |
| `Mapped[tipo]` + `mapped_column` | Declarar una columna con tipo de Python | `max_teams: Mapped[int] = mapped_column(...)` |
| `CheckConstraint` / `UniqueConstraint` | Reglas **en la base** | `capacity_valid`, `uq_lineup_entries_league_player` |
| `select(Model)` | Armar una consulta (estilo 2.0) | repositories |
| `session.scalars(stmt)` | Ejecutar y obtener objetos | `.all()`, `.first()` |
| `selectinload` | Cargar relaciones sin N+1 | listados |
| `.in_([...])` · `.distinct()` · `.order_by()` · `.offset().limit()` | Filtros y paginación | `get_leagues`, `get_global_ranking_page` |
| `session.get(..., with_for_update=True)` | Bloqueo de fila | `finish_league` |
| `update(Model).where(...)` | UPDATE condicional | `claim_pending_*` |
| `get_db` | Una sesión por request (FastAPI `Depends`) | `src/database/base.py` |
| `session.flush()` | Manda los cambios pendientes a la base **sin cerrar la transacción** (ver más abajo) | `create_league`, `generate_fixture`, `begin_league_match` |
| **Alembic** | Migraciones versionadas de esquema | `alembic/versions/` (cabeza `8f23a963366d`) |

### `add` vs `flush` vs `commit` (SQLAlchemy)
> En mis palabras: el `flush` **aplica parcialmente** los cambios a la base **sin cerrar ni guardar definitivamente** la transacción. Si algo falla → `rollback()`; si todo anda bien → `commit()`.

| Paso | Qué pasa | ¿Se puede deshacer? |
|---|---|---|
| `session.add(obj)` | El objeto queda solo **en memoria** (en la sesión); no hay SQL todavía | Sí |
| `session.flush()` | Se ejecutan los `INSERT/UPDATE/DELETE` pendientes. La base los procesa **dentro de mi transacción**; otras sesiones **no** los ven | Sí, con `rollback()` |
| `session.commit()` | Hace un `flush` y **cierra** la transacción: queda guardado de verdad | No |

**Para qué lo usamos**
1. **Obtener el ID que genera la base**: después de `add()` el objeto todavía no tiene `league_id`/`team_id`; con `flush()` aparece y se puede usar en lo que sigue (`create_league`: `add` → `flush` → `save_lineup(..., league.league_id)` → `commit`).
2. **Mantener una sola transacción**: `generate_fixture` hace `add_all(matches)` + `flush()` para tener los IDs sin commitear; si algo falla, se revierte todo junto con el cambio de estado de la liga ([[R2 - Iniciar y correr la liga|GLORY-239]]).
3. **Que otra parte del código lea los cambios**: en `begin_league_match` ([[R2 - Iniciar y correr la liga|GLORY-241]]) se hace `flush()` para que `prepare_engine` lea la alineación copiada, y `session.refresh(match)` para recargar el objeto que en memoria seguía en `PENDING`.
4. **Efecto secundario útil**: las restricciones de la base (`CheckConstraint`, `UniqueConstraint`) saltan **en el flush**, no en el `add`. Por eso un error de constraint aparece ahí y el `except` hace `rollback()`.

**Preguntas típicas**
- *¿Por qué no `commit()` en vez de `flush()`?* → Un commit anticipado rompe la atomicidad: si después falla algo, lo ya guardado no se puede deshacer.
- *¿Otra sesión ve lo flusheado?* → No, hasta el commit (aislamiento de Postgres).
- *¿Es la única forma de validar?* → No: validamos en los services antes de escribir; el flush es una **red de seguridad** de la base, no el mecanismo principal.

## Términos del dominio
| Término | Significado |
|---|---|
| **Tick** | Un paso de simulación (0,5 s de reloj). 240 ticks = 2 min; 600 = 5 min |
| **Fixture** | Calendario de la liga (todos contra todos) |
| **Fecha / round** | Conjunto de partidos que se juegan en paralelo |
| **Comportamiento** | Código Python del usuario (`PLAYER` o `COACH`) con `def on_tick(api)` |
| **Primitivas** | Los métodos de `PlayerAPI` / `CoachAPI` que el código del usuario puede llamar |
| **Titular / suplente** | Titular = tiene `position`; suplente = `position is None` |
| **`host_side`** | Lado en el que juega hoy el anfitrión (cambia en el Half-Time) |
| **Claim** | "Reclamar" un registro con un UPDATE condicional |
| **Registro** | Diccionario en memoria (`match_registry`, `coach_registry`) |
| **Amistoso** | Partido 1 vs 1 sin liga; sin DT ni penales |
| **Contrato** | Documento de la API (`contrato api mas actualizado.pdf`) |

## Eventos WebSocket (todos)
| Evento | Canal | Cuándo |
|---|---|---|
| `TICK_UPDATE` | partido | cada tick |
| `PAUSE_STARTED` / `PAUSE_FINISHED` | partido | pausas ([[GLORY-216 Lógica de ticks en pausas\|GLORY-216]]) |
| `PENALTIES_STARTED` / `PENALTIES_FINISHED` | partido | penales ([[GLORY-279 Preparar la tanda - evento y sorteo\|GLORY-279]]) |
| `MATCH_STARTED` / `MATCH_FINISHED` | global | partido empieza / termina |
| `LEAGUE_INITIATED` / `LEAGUE_CANCELLED` / `LEAGUE_FINISHED` / `LEAGUE_MATCH_FINISHED` | global | vida de la liga |
| `TEAM_JOINED_LEAGUE` / `LEAVE_LOBBY_LEAGUE` | global | lobby |
| `FRIENDLY_CREATED` / `FRIENDLY_CANCELLED` / `GUEST_JOINED_FRIENDLY` | global | amistosos |
| `RANKING_UPDATED` | global | ranking global |
