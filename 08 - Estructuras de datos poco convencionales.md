---
tags: [glory, defensa, python, estructuras]
---
# 08 — Estructuras de datos "raras": por qué las usamos y qué significan
> Volver a [[00 - Inicio]] · Tests: [[07 - Tests]] · Patrones: [[04 - Glosario y patrones]]

**Idea para la defensa**: cada estructura se elige por una **propiedad**. Si sé cuál es, puedo justificar cualquier uso.

| Estructura | Propiedad clave | Pregunta que responde |
|---|---|---|
| `list` | **Ordenada**, se puede modificar, admite repetidos | "¿en qué orden?" |
| `tuple` | **Ordenada, inmutable, hashable** | "un valor compuesto de tamaño fijo" / "clave compuesta" |
| `set` | **Sin repetidos, sin orden, pertenencia rápida** | "¿está o no está?" |
| `dict` | **Clave → valor**, búsqueda directa | "dado este id, dame su dato" |

---
## 1. `set` (conjunto)
**Qué es**: una colección **sin repetidos y sin orden**. Preguntar `x in conjunto` es muy rápido y agregar dos veces lo mismo no hace nada. Se crea con `set()` (**`{}` crea un diccionario vacío**, trampa clásica).

| Dónde | Para qué | Por qué un set y no una lista |
|---|---|---|
| `PauseState.substituted_team_ids: set[int]` ([[GLORY-215 Función substitute_player en el motor\|GLORY-215]]) | Equipos que **ya cambiaron** en esta pausa. `if team_id in ...substituted_team_ids` → `SubstitutionLimitError`; `.add(team_id)` al cambiar | Solo importa **si está o no**; no puede haber un equipo dos veces; `add` es idempotente |
| `_primitives_of(api_class) -> set[str]` ([[GLORY-288 Separar comportamientos de jugador y de DT\|GLORY-288]]) | Nombres de las primitivas de `PlayerAPI`/`CoachAPI`, armado con una **set comprehension** `{name for name, member in vars(api_class).items() if callable(member) and not name.startswith("_")}` | Después solo se pregunta "¿este nombre es una primitiva ajena?"; el orden no importa |
| `valid_totals = {total_ticks_for(m) for m in VALID_MATCH_DURATIONS_MINUTES}` ([[GLORY-216 Lógica de ticks en pausas\|GLORY-216]]) | Los totales de ticks válidos (`{240, 600}`) | Para preguntar `total_ticks in valid_totals` |
| `get_member_league_ids(...) -> set[int]` ([[R1 - Ligas#GLORY-270 — `is_member` en el listado de ligas\|GLORY-270]]) | Ids de ligas donde mi equipo está inscripto; `is_member = league_id in member_ids` | Búsqueda directa por id; sin repetidos (se usa con `.distinct()` en la consulta) |
| `enrolled_team_ids = set(session.scalars(...))` (`join_league`) | Equipos ya inscriptos | Comprobar pertenencia; y `len(...)+1 == max_teams` para saber si se completó el cupo |
| `_formation_from_roles(roles: set[str\|None])` | La formación **se deduce de los roles de los titulares**: `set(positions) == roles` | Dos conjuntos son iguales **sin importar el orden**: `{"DEF_LEFT","DEF_RIGHT","FORWARD"}` |
| `_background_tasks: set[asyncio.Task]` ([[R2 - Iniciar y correr la liga#GLORY-243 — Fin de un partido y avance de fecha\|GLORY-243]]) | Guardar las tareas en segundo plano mientras corren | `asyncio` guarda **referencias débiles**; si nadie guarda la tarea, el recolector de basura puede cancelarla a la mitad. Al terminar se la saca con un callback |
| `_running_matches: set[asyncio.Task]` (`match_start_service`) | Igual: los motores de partido en curso | Mismo motivo |
| `event_bus._subscribers = set()` | Las colas de los suscriptores | `unsubscribe` usa `.discard` (no falla si no estaba) |
| `for queue in set(self._subscribers):` en `publish` | Recorrer una **copia** | Si alguien se suscribe o se da de baja **mientras** se recorre, un set original cambiaría de tamaño y daría error |
| `defaultdict(set)` en `match_connections` | `partido → conexiones WebSocket` | `d[match_id].add(ws)` crea el set solo si no existía |

**Para recordar**: set = "**¿está?**". `add` (agregar), `discard` (sacar sin error), `remove` (sacar y falla si no está), `in`, `==` (igualdad sin orden).

---
## 2. `tuple` (tupla)
**Qué es**: como una lista pero **inmutable**: no se puede cambiar después de crearla. Por eso es **hashable** y puede usarse como **clave de un diccionario**. Se usa para "un valor compuesto de tamaño fijo" donde cada posición **significa algo**.

| Dónde | Qué es | Detalle |
|---|---|---|
| `planned_substitutions: dict[int, tuple[int, int]]` ([[GLORY-214 Endpoint de sustitución - efectuar o planificar\|GLORY-214]]) | `team_id → (player_out_id, player_in_id)` | ⚠️ **El orden es (saliente, entrante)**, aunque el endpoint recibe `player_in_id` primero. Se desempaqueta en `for team_id, (out_id, in_id) in ....items()` |
| `CoachRegistry._coaches: dict[tuple[int,int], int]` ([[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga\|GLORY-299]]) | clave `(league_id, team_id)` → `behavior_id` | Una tupla como clave de un diccionario: es la **clave compuesta**. Una lista **no** podría (no es hashable). `.get((league_id, team_id))` |
| `_bounce(...) -> tuple[float, float]` ([[GLORY-181 La pelota rebota en los bordes de la cancha\|GLORY-181]]) | devuelve `(coordenada, velocidad)` | Una función que devuelve **dos cosas** devuelve una tupla; se recibe con `x, vx = _bounce(...)` |
| `validate_substitution_players(...) -> tuple[MatchPlayerState, MatchPlayerState]` | `(player_out, player_in)` | Se recibe con `player_out, player_in = ...` |
| `build_round_robin -> list[list[tuple[int,int]]]` ([[R2 - Iniciar y correr la liga\|GLORY-239]]) | cada fecha es una lista de pares `(local, visitante)` | Un partido = un par **ordenado**; para alternar localía se invierte `(a, b) → (b, a)` |
| `VALID_MATCH_DURATIONS_MINUTES = (2, 5)` ([[GLORY-183 Unir constants y timing en metrics\|GLORY-183]]) | Constante: duraciones permitidas | Tupla y no lista: **nadie puede modificarla** por accidente; se usa con `in` |
| `PAUSE_PLAN = ((25, COOLING_BREAK), (50, HALF_TIME), (75, COOLING_BREAK))` ([[GLORY-216 Lógica de ticks en pausas\|GLORY-216]]) | Calendario: `(porcentaje, tipo)` | Se recorre con `for number, (percentage, pause_type) in enumerate(PAUSE_PLAN, start=1)`: **desempaquetado anidado** y `enumerate` numerando desde 1 |
| Retornos de listados `-> tuple[list[Team], int, int, int]` ([[R3 - Ranking global\|GLORY-204]]) | `(items, total, page, limit)` | Varios valores devueltos juntos |
| `@pytest.mark.parametrize("a, b", [(240, [60,...]), ...])` ([[07 - Tests]]) | Cada fila = una tupla de datos de un caso de prueba | |
| `assert (state.score.phome, state.score.paway) == (1, 0)` | Comparar **dos valores a la vez** | Más corto que dos `assert` |

**Trampas**: una tupla de **un** elemento se escribe `(2,)` (con coma). Las tuplas se pueden **desempaquetar**: `a, b = (1, 2)`. El contenido **dentro** de una tupla puede ser mutable (una lista dentro de una tupla sí se puede modificar).

---
## 3. `dict` (diccionario)
**Qué es**: pares **clave → valor** con búsqueda directa por clave. Es la estructura más usada del motor.

| Dónde | Forma | Para qué |
|---|---|---|
| `match_registry` (`MatchRegistry._matches`) | `match_id → MatchState` | Encontrar el estado de un partido **en curso** en memoria ([[GLORY-214 Endpoint de sustitución - efectuar o planificar\|GLORY-214]]) |
| `MatchState.players` | `player_id → MatchPlayerState` | Acceso directo a cualquier jugador; `on_pitch_player_ids()` filtra los que tienen `position` |
| `intentions: dict[int, Intention]` (`play_tick`) | `player_id → Intention` | Primero **se guardan todas**, después **se aplican juntas** (ver [[GLORY-292 Ejecutar la tanda de penales en el motor\|GLORY-292]]) |
| `PenaltyShootout.kicker_order / keeper_order` | `team_id → [player_ids]` | Orden de pateo y atajada **por equipo** ([[GLORY-279 Preparar la tanda - evento y sorteo\|GLORY-279]]) |
| `attributes: dict[AttributeType, int]` | enum → valor | **Claves que son un Enum**: evita escribir strings sueltos |
| `PAUSE_DURATION_SECONDS: dict[PauseType, int]` | `{COOLING_BREAK: 10, HALF_TIME: 20}` | Config por tipo; se pasa por parámetro (`pause_durations`) y los tests pasan 0 |
| `FORMATION_POSITIONS: dict[str, dict[str, Point2D]]` | formación → rol → posición | **Diccionario anidado**: `FORMATION_POSITIONS["2-1"]["FORWARD"]` |
| `team_stats` en `build_standings` ([[R2 - Iniciar y correr la liga\|GLORY-237]]) | `team_id → {matches, wins, goals_for, ...}` | **Dict de dicts**, inicializado en 0 para **todos** los equipos (los que no jugaron también aparecen) |
| `coach_on_ticks`, `behavior_codes_for_coach: dict[int, dict[int,str]]` ([[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor\|GLORY-297]]) | `team_id → función` / `team_id → {behavior_id: código}` | Lo que necesita el DT de cada equipo |
| `planned_substitutions` | `team_id → (out, in)` | **Un solo plan por equipo**: asignar de nuevo **reemplaza** el anterior (con una lista habría que buscar y borrar) |
| `entries_by_team = {host_id: [], away_id: []}` | `team_id → [entradas]` | Agrupar |
| `llamadas = {"apply": 0, "goal": 0}` (tests) | contadores | Una función interna puede **modificar** el dict de afuera sin `nonlocal` |

**Detalles de lenguaje que aparecen**
- **`field(default_factory=dict)` y no `= {}`**: en una `@dataclass`, un `{}` como valor por defecto sería **el mismo diccionario compartido por todos los objetos** (y Python lo prohíbe). `default_factory` crea uno **nuevo** por objeto. Está comentado en `state.py` ("así cada partido tiene su propio dict de jugadores"). Vale igual para `set` y `list`.
- **`.get(clave)`** devuelve `None` si no existe (`coach_registry.get(...)`); `dict[clave]` lanza `KeyError`.
- **`.pop(clave, None)`**: saca sin error aunque no exista (`CoachRegistry.remove`).
- **`.clear()`**: vaciar (`planned_substitutions.clear()` tras aplicar los planes: se usan **una sola vez**).
- **`.items()`** para recorrer pares; **`dict comprehension`** `{k: v for ...}`.
- **Orden de inserción**: desde Python 3.7 un dict conserva el orden en que se insertaron las claves.

---
## 4. `list` y sus trucos
| Dónde | Truco | Explicación |
|---|---|---|
| `build_round_robin` | `mixed_teams = [mixed_teams[0]] + [mixed_teams[-1]] + mixed_teams[1:-1]` | **Rotación** del algoritmo del círculo: el primero queda **fijo**, el último pasa al lugar 1, el resto se corre. Se repite N−1 veces → N−1 fechas |
| `build_round_robin` | `pairs = [(mixed_teams[i], mixed_teams[n-1-i]) for i in range(n//2)]` | **List comprehension**: enfrenta el primero con el último, el segundo con el anteúltimo... |
| `random.shuffle(mixed_teams)` sobre una **copia** (`team_ids.copy()`) | El **sorteo** del fixture | Se copia para **no modificar** la lista que recibió la función |
| `kicker_order[team_id]` | lista **ordenada** de jugadores | El orden **es el dato** (quién patea primero); por eso es lista y no set |
| `round[0] = (round[0][1], round[0][0])` | Intercambiar local y visitante | Se alterna la localía en las fechas impares |
| `[pausa.tick for pausa in calendario] == [60, 120, 180]` (tests) | Comparar listas armadas con comprehension | |

---
## 5. Otras estructuras y trucos que pueden llamar la atención
- **`@dataclass`**: genera el `__init__` y la comparación a partir de los campos. **`@dataclass(frozen=True)`** (`Point2D`, `Vector2D`, `GameState`, `Intention`, `PauseSlot`) hace al objeto **inmutable**: es la "segunda capa" de seguridad con la que el código del usuario no puede modificar lo que ve (ver [[GLORY-288 Separar comportamientos de jugador y de DT|GLORY-288]]). Ojo: `frozen` **no** protege las listas que contenga (por eso el builder entrega **copias** vía snapshots).
- **`Enum`** (`TeamSide`, `MatchPhase`, `PauseType`, `ActionType`): un conjunto cerrado de valores con nombre. Se compara con **`is`** (`scoring_side is state.host_side`) porque cada miembro es único.
- **`Optional[X]` / `X | None`**: "puede no haber valor". En el motor `position is None` **significa "está en el banco"**: un solo criterio para titular/suplente.
- **`Callable` y alias de tipo**: `ApplyIntentions = Callable[[MatchState, dict[int, Intention]], None]` describe la **firma** de una función que se pasa por parámetro.
- **Closures (funciones dentro de funciones)**: `make_on_finished()` **devuelve** la función `on_finished` que "recuerda" `session_factory`; en los tests, `sleep_espia` recuerda las listas de afuera.
- **`functools.partial`**: `task.add_done_callback(partial(_on_task_done, state=state, on_finished=on_finished, registry=registry))`. `asyncio` llama al callback pasándole **solo la tarea**; `partial` "pre-carga" los demás argumentos para que `_on_task_done` los reciba igual.
- **`global` dentro de un comportamiento** (`PENALTY_GOALKEEPER`): `target_post` vive **fuera** de `on_tick` para conservar su valor entre ticks; `global target_post` permite reasignarla. Cada jugador carga su **propia copia** del código, así que no se comparte entre arqueros.
- **`-team_id` con `reverse=True`** (`standings_sort_key`): para ordenar "mayor es mejor" y desempatar por **menor id**, se **niega** el número: al ordenar de mayor a menor, `-1` queda antes que `-2`. Mismo truco con `-goals_against`.
- **Tuplas como "llaves" de orden**: `standings_sort_key` devuelve una **tupla** `(points, goal_diff, goals_for, -goals_against, -team_id)`; Python compara tuplas **elemento por elemento**, así que desempata en cascada.
- **`yield` en fixtures** y **generadores** (`get_db` hace `yield session` y cierra después): "dame esto y, cuando termine, ejecutá lo que sigue".
- **`from __future__ import annotations`**: permite usar tipos antes de definirlos (referencias adelantadas).
- **`str | None`** (Python 3.10+) equivale a `Optional[str]`.

---
## 6. Preguntas típicas (con respuesta corta)
- **¿Por qué `substituted_team_ids` es un set y no una lista?** → Solo importa si el equipo **ya está**; sin repetidos; `in` y `add` son simples. Y se crea nuevo en cada pausa, así que se reinicia solo ([[GLORY-215 Función substitute_player en el motor|GLORY-215]]).
- **¿Por qué `planned_substitutions` es un dict de tuplas?** → Un plan **por equipo** (la clave `team_id` lo garantiza y reemplazar es asignar) y cada plan es un par **fijo y ordenado** `(saliente, entrante)` ([[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]]).
- **¿Por qué la clave de `coach_registry` es una tupla?** → Porque el DT se elige **por liga y por equipo**: la clave compuesta `(league_id, team_id)` identifica un único DT ([[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga|GLORY-299]]).
- **¿Por qué `(2, 5)` y no `[2, 5]`?** → Es una constante: inmutable.
- **¿Para qué un set de tareas (`_background_tasks`)?** → Para que Python no las "recoja" mientras corren; se sacan al terminar.
- **¿Por qué `field(default_factory=dict)`?** → Para que cada objeto tenga **su propio** diccionario y no uno compartido.
- **¿Por qué se copia el set antes de recorrerlo en `publish`?** → Para poder suscribirse/desuscribirse durante el recorrido sin error.
- **¿Por qué un dict de dicts en la tabla de posiciones?** → Para acumular estadísticas por equipo con acceso directo `team_stats[team_id]["points"] += ...` y que **todos** los equipos aparezcan.
- **¿Para qué sirve `-team_id` ahí?** → Desempatar por menor id al ordenar de mayor a menor.
- **¿Qué significa `position is None`?** → Que el jugador está en el banco; no hay otro flag.
