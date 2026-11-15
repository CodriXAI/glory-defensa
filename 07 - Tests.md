---
tags: [glory, defensa, tests]
---
# 07 — Tests: cómo se testeó todo lo que desarrollé y revisé
> Volver a [[00 - Inicio]] · Estructuras de datos raras: [[08 - Estructuras de datos poco convencionales]] · Respuestas rápidas: [[06 - Comodines para la defensa]]

**Regla de la cátedra** (Sprint 3): *"Ningún requerimiento está completo sin sus unit tests. No puede haber tests fallando en develop."* Por eso **cada ticket tiene su archivo de tests** y es lo primero que pueden preguntar: "¿cómo lo probaron?".

## 1. Lo básico: herramientas y convenciones
| Qué | Detalle |
|---|---|
| Herramienta | **pytest 8.3.3** + **pytest-asyncio 0.24** (para tests con `async`/`await`) |
| Dónde | `tests/unit/test_*.py`; fixtures compartidas en `tests/conftest.py` |
| Nombres | `test_deberia_<lo que debe pasar>` (y `test_no_deberia_...` para lo que **no** debe pasar). Se lee como una frase y funciona como documentación |
| Estructura | **Arrange – Act – Assert**: armo el estado, ejecuto la función, verifico con `assert` |
| Config | `pytest.ini` solo define `pythonpath = .`; los tests async se marcan a mano con `@pytest.mark.asyncio` |
| Front | `vitest` + Testing Library (`npm test`); ver [[GLORY-311 Botón Volver al Inicio en los lobbies\|GLORY-311]] |

### La base de datos de los tests (`conftest.py`)
- Fixture **`session`**: crea una base **SQLite en memoria** (`sqlite://`) con **`StaticPool`** (una única conexión compartida, necesaria porque una base en memoria desaparece si se cierra la conexión), activa **`PRAGMA foreign_keys=ON`** (SQLite no hace cumplir las claves foráneas por defecto), crea las tablas (`Base.metadata.create_all`) y al terminar las borra (`drop_all`).
- Con la variable **`TEST_DATABASE_URL`** los mismos tests corren contra **Postgres real**: sirve para detectar lo que SQLite no controla (longitud de `VARCHAR`, mayúsculas/minúsculas).
- **`autoflush=False`** en la sesión de tests: SQLAlchemy no manda cambios solo antes de cada consulta (ver `flush` en [[04 - Glosario y patrones]]).
- Fixture **`usuario_con_equipo`**: un `User` y un `Team` ya guardados (casi todo test de comportamientos los necesita).
- **`test_alembic_migrations.py`** compara el esquema que arma `alembic upgrade head` contra el de `create_all`: si alguien cambia un modelo y olvida la migración, el test falla.
- Por qué importa [[GLORY-224 Fix test_join_friendly_motor|GLORY-224]]: con `StaticPool` el test, el request y el `on_finished` **comparten una conexión**; en ese test se usa una base **en archivo** (`tmp_path`) para tener una conexión por sesión.

## 2. Los 4 niveles de test que usamos
| Nivel | Qué prueba | Qué usa | Ejemplo |
|---|---|---|---|
| **Unidad pura (motor)** | Una función del motor, **sin base ni HTTP** | `_build_state()`, fakes, `Mock` | `test_substitute_player.py`, `test_simulation.py`, `test_penalties_turn.py` |
| **Servicio con base** | Reglas de negocio + consultas | fixture `session` (SQLite en memoria) | `test_finish_league.py`, `test_global_stats.py`, `test_standings_service.py` |
| **Endpoint** | Ruta + servicio + repositorio + base juntos | `TestClient(app)` + `dependency_overrides[get_db]` (le pasan **la sesión del test** a la app) | `test_list_leagues_endpoint.py`, `test_get_league_endpoint.py` |
| **Punta a punta** | Un flujo completo con el **motor real** | `httpx` dentro del event loop del test | `test_join_friendly_motor.py` ([[GLORY-224 Fix test_join_friendly_motor\|GLORY-224]]) |

## 3. Las técnicas, con ejemplos reales
### 3.1 Estado de prueba compartido: `_build_state()`
`tests/unit/test_match_runner.py` arma un partido chico: **equipo 1 (HOME)** con titulares 100, 101, 102 y suplentes 103–105; **equipo 2 (AWAY)** con titulares 200–202 y suplentes 203–205. Casi todos los tests del motor lo reutilizan (`from tests.unit.test_match_runner import _build_state`). Por eso en las notas de tickets aparecen esos números.

### 3.2 Inyección de dependencias → fakes
`run_tick` y `run_match` reciben `execute_behavior`, `apply_intentions`, `detect_goal` y `notify` **por parámetro**. Eso permite reemplazarlas por **fakes** simples en los tests: `_idle_behavior` (todos quietos), `_noop_apply` (no mueve nada), `_no_goal` (nunca hay gol). *Esta es la razón de diseño de pasar funciones por parámetro.*

### 3.3 `Mock`, `AsyncMock` y `side_effect`
```python
detect_goal = Mock(side_effect=[GOL, SIN_GOL])   # 1ª llamada → GOL, 2ª → SIN_GOL
...
assert detect_goal.call_count == 2               # cuántas veces lo llamaron
```
- `Mock(return_value=x)`: siempre devuelve `x`. `Mock(side_effect=[a, b])`: devuelve **un valor por llamada** (una **lista** como script de lo que va a pasar).
- `AsyncMock()` es lo mismo para funciones `async` (`notify`).
- `call_count` / `assert_called...`: verificar que algo **se llamó** (o no).

### 3.4 `monkeypatch`: cambiar algo durante un test
```python
monkeypatch.setattr(penalties, "PENALTY_MAX_TICKS", 1)           # cada penal dura 1 tick
monkeypatch.setattr(match_runner.asyncio, "sleep", sleep_espia)  # no esperar de verdad
```
Se restaura solo al terminar el test. Es la forma de **no esperar 10 o 20 segundos** de pausa en un test.

### 3.5 El "espía": funciones que anotan lo que ven
En `test_match_pauses.py`, `sleep_espia(segundos)` **no espera**: si le piden dormir más de 0 s sabe que es una pausa y anota la fase, el tick y cuántos comportamientos se ejecutaron. Los `@dataclass PausaVista` y `PartidoJugado` guardan lo observado para hacer los `assert` al final. Con `tick_interval_seconds=0` toda espera mayor a 0 es una pausa.

### 3.6 Tiempo en 0
Los tests pasan `tick_interval_seconds=0`, `pause_durations`/`penalty_pause_seconds=0` y `penalty_setup_seconds=0`: **el motor completo corre en milisegundos**. Por eso esos valores son parámetros con default y no constantes fijas ([[GLORY-216 Lógica de ticks en pausas|GLORY-216]], [[GLORY-310 Cuadro de preparación antes de cada penal|GLORY-310]]).

### 3.7 `@pytest.mark.asyncio`
Las funciones `async def test_...` necesitan la marca para que pytest-asyncio las ejecute dentro de un event loop.

### 3.8 `@pytest.mark.parametrize`: la misma prueba con varios datos
```python
@pytest.mark.parametrize("total_ticks, ticks_esperados",
    [(240, [60, 120, 180]), (600, [150, 300, 450])])
```
Genera **un test por fila**. Cada fila es una **tupla** de valores (ver [[08 - Estructuras de datos poco convencionales]]). Se usa en el calendario de pausas, el rebote de la pelota (`test_simulation.py`), el saque tras gol, el orden de la tabla, etc.

### 3.9 `pytest.raises`: verificar que **falla bien**
```python
with pytest.raises(InvalidData):
    validate_behavior_code("def on_tick(api:\n    return", BehaviorType.PLAYER)
```
El test pasa **si** se lanza esa excepción. Es la forma de probar los "qué pasa si falla".

### 3.10 Fixtures con `yield` (limpiar después)
```python
@pytest.fixture()
def partido():
    state = _build_state()
    match_registry.add(state)
    yield state                       # el test corre acá
    match_registry.remove(MATCH_ID)   # y esto corre al terminar
```
Necesario porque `match_registry` es **global**: si un test lo deja sucio, rompe a los siguientes.

### 3.11 Truquitos de diseño de tests (muy buenas respuestas en la defensa)
- **Valores distintos a propósito**: en `test_substitute_player.py` el saliente y el entrante tienen atributos y `behavior_id` distintos; con valores iguales, un intercambio por error pasaría desapercibido.
- **Dato imposible como centinela**: en `test_penalties_preparation.py` se reemplaza el orden de pateadores por `[999]` (un jugador que no existe). Si la función **sorteara de nuevo**, lo pisaría y el `assert` falla: así se prueba que *no vuelve a sortear*.
- **Probar que NO pasó nada** (`test_no_deberia_...`): "no se suma dos veces", "no se publica el evento dos veces", "no se modifica un amistoso".
- **Contadores en un dict**: `llamadas = {"apply": 0, "goal": 0}` y funciones internas que lo incrementan (la función interna puede modificar el dict de afuera sin `nonlocal`; con un `int` no).
- **Releer lo guardado**: `session.expire_all()` antes de leer, para ver lo que quedó **en la base** y no lo que la sesión recuerda en memoria (`test_global_stats.py`).
- **Test de borde**: 0,3 m con CONTROL 20 ([[GLORY-301 Correcciones pendientes de ubicar jugadores|GLORY-301]]), pelota a 58,0 m y 2,0 m del borde ([[GLORY-181 La pelota rebota en los bordes de la cancha|GLORY-181]]), `total_ticks` 1, 3, 5, 239, 1000 ([[GLORY-216 Lógica de ticks en pausas|GLORY-216]]).

## 4. Mapa: ticket → archivo de tests → qué se verificó

### 🟢 Los que desarrollé
| Ticket | Archivo de tests | Qué se verifica / técnica |
|---|---|---|
| [[GLORY-279 Preparar la tanda - evento y sorteo\|GLORY-279]] | `test_penalties_preparation.py`, `test_match_broadcast.py` | Sorteo solo una vez (centinela `[999]`); payloads de `PENALTIES_STARTED/FINISHED`; `notify_penalties` con un `_FakeWebSocket` |
| [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales\|GLORY-280]] | `test_penalties_turn.py`, `test_penalty_behaviors.py`, `test_penalty_goalkeeper.py` | Arquero en el arco izquierdo, pateador a 11 m y pelota cerca; quien pateó y luego ataja queda **solo** con el comportamiento de arquero; pateador patea a potencia 1.0, elige distintos lugares, se queda quieto si la pelota está lejos o en movimiento; el arquero va y vuelve de poste a poste; la pelota que pasa por el arquero **queda quieta y no cuenta gol** |
| [[GLORY-301 Correcciones pendientes de ubicar jugadores\|GLORY-301]] | `test_penalties_turn.py` | `test_..._el_pateador_con_control_minimo_poder_patear_la_pelota_a_0_3_metros` |
| [[GLORY-292 Ejecutar la tanda de penales en el motor\|GLORY-292]] | `test_penalties_shootout.py`, `test_penalties_result.py` | Tanda completa con `Mock(side_effect=[GOL, SIN_GOL])` y `PENALTY_MAX_TICKS=1`; gana el host a la 1ª ronda; empate tras 3 rondas → **sorteo + 1 gol**; guardado de `phome/paway` |
| [[GLORY-310 Cuadro de preparación antes de cada penal\|GLORY-310]] | `test_penalties_shootout.py` | El aviso de preparación sale **antes** del primer tick y se espera `setup_seconds` |
| [[GLORY-296 Primitivas de director técnico\|GLORY-296]] | `test_coach_api.py` | `change_behavior` deja el comportamiento indicado y **lo conserva si el código no carga**; `substitute` lanza `NotInPause`/`SubstitutionLimit`/`InvalidSubstitution` **sin cambiar el estado**; las lecturas devuelven tick, pausa, titulares y suplentes en orden |
| [[GLORY-288 Separar comportamientos de jugador y de DT\|GLORY-288]] | `test_validate_behaviors.py`, `test_behavior_type.py`, `test_lineup_service.py` | Código válido; `InvalidData` con sintaxis rota, `for`, `while`, sin `on_tick`, primitivas del otro tipo; `load_on_tick` **falla al ejecutar** (`NameError`) si usa una función no permitida; los comportamientos por defecto validan y cargan; el tipo se conserva al editar |
| [[GLORY-216 Lógica de ticks en pausas\|GLORY-216]] | `test_match_pauses.py` | `pause_schedule_for` (`parametrize` 240/600 y "cualquier otro total → `[]`"); el espía de `sleep`: fase `PAUSED`, reloj detenido, sin comportamientos ni `TICK_UPDATE`, 10/20/10 s, cada pausa una sola vez, 3 pausas y `FINISHED` |
| [[GLORY-217 Cambio de posición tras Half-Time\|GLORY-217]] | `test_match_pauses.py`, `test_penalties_turn.py` | `swap_sides`: invierte `side` de **todos**, posiciones espejadas, pelota al centro; el gol del 2º tiempo suma al equipo correcto |
| [[GLORY-215 Función substitute_player en el motor\|GLORY-215]] | `test_substitute_player.py` | El entrante hereda posición/rol; el saliente queda en el banco; los **3 errores** en el orden fijado; el otro equipo sí puede cambiar; la pausa siguiente reinicia el límite |
| [[GLORY-214 Endpoint de sustitución - efectuar o planificar\|GLORY-214]] | `test_substitution_endpoint.py` | `apply_substitution`: con pausa → `APPLIED`; sin pausa → `PLANNED` y `planned_substitutions == {1: (100, 103)}`; nuevo plan **reemplaza** al anterior; `_run_pause` aplica el plan al empezar |
| [[GLORY-95 Unir game_types, geometry_types e intentions en data_types\|GLORY-95]], [[GLORY-183 Unir constants y timing en metrics\|GLORY-183]] | `test_geometry_types.py`, `test_timing.py` (y todos los que importan esos módulos) | Refactor: **no hubo tests nuevos de comportamiento**; los commits de estos tickets solo **corrigen las rutas de import** de los tests existentes ("GLORY-95 refactor corrección de rutas de imports en tests para data_types"). La prueba del refactor es que siguen pasando. Ej.: `test_timing`: tick de 0,5 s, minutos→ticks, máximo 600 |
| [[GLORY-181 La pelota rebota en los bordes de la cancha\|GLORY-181]] | `test_simulation.py` | `parametrize` del rebote en bordes (y=58 → 59, velocidad invertida); la **rapidez se conserva**; el tiro desviado rebota y **no cuenta gol** (`y_del_tiro` 20.0 vs 25.0) |
| [[GLORY-180 Saque después de un gol\|GLORY-180]] | `test_pitch.py`, `test_simulation.py` | `kickoff_ball_position` para HOME y AWAY; el saque tras gol (`parametrize` con `SAQUE_DE_HOME`) |
| [[GLORY-224 Fix test_join_friendly_motor\|GLORY-224]] | `test_join_friendly_motor.py` | El test mismo **es** el arreglo: fixture `db` propia, base en archivo temporal (`tmp_path`) |
| [[GLORY-311 Botón Volver al Inicio en los lobbies\|GLORY-311]] | `LeagueLobbyPage.test.jsx`, `FriendlyLobby.test.jsx` | Front (vitest): el enlace "Volver al Inicio" existe en cada estado y apunta a `/` |

### 🔵 Los que revisé
| Ticket(s) | Archivo de tests | Qué se verifica |
|---|---|---|
| [[R1 - Ligas#GLORY-118 — Modelo y persistencia de liga\|GLORY-118]] | `test_league_persistence.py`, `test_lineup_entry_persistence.py` | Cupo par 4–12 aceptado; menor a 4 y duración 0 **rechazados por la base**; alineación sin partido ni liga, o con ambos, rechazada (XOR); jugador repetido y rol repetido rechazados |
| [[R1 - Ligas#GLORY-127 — Verificación de contraseña de la liga\|GLORY-127]] | `test_league_join_token.py` | Token válido; 401 `PASSWORD_INCORRECT`; 400 `LEAGUE_NOT_PRIVATE`; 404 |
| [[R1 - Ligas#GLORY-178 — Ver el detalle de una liga\|GLORY-178]] | `test_get_league_endpoint.py` | Detalle completo; equipos **sin duplicados y anfitrión primero**; 403 privada sin inscribirse; 404 |
| [[R1 - Ligas#GLORY-270 — `is_member` en el listado de ligas\|GLORY-270]], [[R1 - Ligas#GLORY-312 — Listado de ligas con varios estados\|GLORY-312]] | `test_list_leagues_endpoint.py` | No expone la contraseña; cuenta inscriptos; filtra por **varios `status` separados por coma**; pagina de la más nueva a la más vieja; lista vacía fuera de rango |
| [[R1 - Ligas#GLORY-309 — `league_id` en el resultado final del partido\|GLORY-309]] | `test_match_result_endpoint.py` | `league_id` si es de liga; goles al equipo correcto; 0-0; 400 `MATCH_NOT_FINISHED` |
| [[R1 - Ligas#GLORY-229 — Ver la alineación vigente de una liga\|GLORY-229]], [[R1 - Ligas#GLORY-230 — Cambiar la alineación vigente\|GLORY-230]] | `test_get_league_lineup_endpoint.py`, `test_update_league_lineup_endpoint.py` | Devuelve la vigente y la nueva tras cambiar; 400 y **mantiene la anterior** si es inválida; 409 si la liga terminó; 401/403/404 |
| [[R2 - Iniciar y correr la liga#GLORY-239 — Iniciar la liga\|GLORY-239]] | `test_start_league_service.py`, `test_start_league_endpoint.py` | Inicia y crea fixture; 404 / 403 si no es el anfitrión; **inicio automático al completar el cupo**; `LEAGUE_INITIATED` **una sola vez**; orden `TEAM_JOINED` → `LEAGUE_INITIATED` |
| [[R2 - Iniciar y correr la liga#GLORY-237 — Tabla de posiciones\|GLORY-237]] | `test_standings_service.py` | Todos en cero sin partidos; estadísticas de cada equipo; **orden** de la tabla con `parametrize` |
| [[R2 - Iniciar y correr la liga#GLORY-241 — Arrancar un partido de liga\|GLORY-241]] | `test_launch_league_match.py` | Pasa a `IN_PROGRESS` y copia la alineación; arranca con la **vigente aunque cambie después**; rechaza uno ya arrancado; si no se puede preparar: `begin_league_match` lo deja `PENDING` y sin alineación, y `start_league_match` lo **cierra 0-0 sin `MATCH_STARTED`** |
| [[R2 - Iniciar y correr la liga#GLORY-242 — Arrancar una fecha\|GLORY-242]] | `test_start_league_round.py` | Arranca los de la fecha y deja pendientes los demás; **sigue aunque uno falle** y rompa la sesión; no relanza si se arranca dos veces la misma fecha |
| [[R2 - Iniciar y correr la liga#GLORY-243 — Fin de un partido y avance de fecha\|GLORY-243]] | `test_league_match_finished.py` | Eventos de liga; la fecha siguiente arranca **una sola vez**; avanza aunque el último partido se cierre 0-0; **no** arranca nada tras la última fecha; un amistoso **no** publica eventos de liga |
| [[R2 - Iniciar y correr la liga#GLORY-244 — Cerrar la liga\|GLORY-244]] | `test_finish_league.py` | Pasa a `FINISHED` y publica el evento **una sola vez**; no cierra si quedan fechas |
| [[R3 - Ranking global#GLORY-205 — Acumular estadísticas globales\|GLORY-205]] | `test_global_stats.py` | Suma a ambos equipos en liga **pública**; **no** en liga privada ni amistoso; **no suma dos veces** si el cierre corre dos veces; `RANKING_UPDATED` **después** de guardar |
| [[R3 - Ranking global#GLORY-204 — Ver el ranking global\|GLORY-204]] | `test_global_ranking.py` | Orden por puntos y, ante empate, diferencia de goles; incluye clubes sin partidos; posición calculada sobre el ranking **completo** en la 2ª página |
| [[R4 - Motor refactor#GLORY-184 — `pitch.py`\|GLORY-184]], [[R4 - Motor refactor#GLORY-185 — `state.py`\|GLORY-185]] | los tests del motor (`test_pitch.py`, `test_state.py`...) | Refactor: siguen pasando con los imports nuevos |
| [[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas\|GLORY-218]], [[R5 - Pausas y estado en vivo#GLORY-298 — `/state` con pelota y posiciones\|GLORY-298]] | `test_match_broadcast.py`, `test_match_pauses.py`, `test_match_state_endpoint.py` | Payloads de pausa; el estado devuelve pelota, posición y `current_action`; suplentes con `null`; `behavior_id` **solo de los propios** y el del estado en memoria queda **intacto** |
| [[R6 - Comportamientos y DT#GLORY-259 — Guardar el `on_tick` en cada jugador\|GLORY-259]], [[R6 - Comportamientos y DT#GLORY-260 — Cambiar el comportamiento durante el partido\|GLORY-260]] | `test_behavior_executor.py`, `tests_match_player_service.py` | `test_behavior_executor`: asigna `behavior_id` y `on_tick` con código válido, **conserva el anterior si el código es inválido**, devuelve la intención de cada jugador según **su** función; `tests_match_player_service`: asigna el nuevo, falla si el jugador no existe o es del rival, conserva el anterior si el código falla ⚠️ ver nota abajo |
| [[R6 - Comportamientos y DT#GLORY-299 — Elegir el DT de la liga\|GLORY-299]], [[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor\|GLORY-297]] | `test_choose_coach.py`, `test_execute_coach.py` | Guarda el DT sin afectar otros equipos/ligas; `None` lo quita; rechaza no-`COACH` y liga no en curso; el DT corre **una vez por tick** y **una vez al empezar cada pausa** |
| [[R7 - Penales#GLORY-291 — Opción de penales en la liga\|GLORY-291]], [[R7 - Penales#GLORY-281 — Marcador de penales\|GLORY-281]], [[R7 - Penales#GLORY-293 — Exponer el resultado y adaptar la tabla\|GLORY-293]] | `test_penalties_enabled.py`, `test_score_penalties.py`, `test_penalties_result.py` | Liga con/sin penales; `prepare_match` los habilita solo si la liga los tiene; `phome/paway` en `None` sin penales; **ganador por penales 2 puntos, perdedor 0, goles sin modificar** |

> ⚠️ `tests_match_player_service.py` no sigue el patrón `test_*.py` de pytest y **probablemente no se ejecuta** en una corrida normal. Está anotado en [[05 - Inconsistencias conocidas]] (#15). Conviene correr `pytest --collect-only` y renombrarlo (decidirlo con el equipo).

## 5. Preguntas típicas sobre tests
- **¿Por qué SQLite en memoria y no Postgres?** → Rápido y aislado en cada test; con `TEST_DATABASE_URL` los mismos tests corren contra Postgres para lo que SQLite no controla.
- **¿Por qué `StaticPool`?** → Una base en memoria vive en **una** conexión; el pool estático fuerza a reutilizarla.
- **¿Qué es un Mock y cuándo lo usan?** → Un objeto falso que acepta cualquier llamada y cuenta cuántas veces se lo llamó; lo usamos para reemplazar piezas que no nos interesan (`execute_behavior`, `notify`).
- **¿Cómo prueban código con `sleep` de 20 segundos?** → `monkeypatch` del `sleep` (o duración 0 por parámetro) y un espía.
- **¿Cómo prueban que algo ocurre una sola vez?** → Cuento llamadas (`call_count`) o eventos publicados y verifico que son 1.
- **¿Cómo prueban un error?** → `pytest.raises(Error)` y además verifico que **el estado no cambió**.
- **¿Cómo prueban un endpoint sin servidor?** → `TestClient(app)` con `dependency_overrides[get_db]` que entrega la sesión del test.
- **¿Qué cubrieron y qué no?** → Unitarios y de endpoint por ticket; **no** hay pruebas de carga ni de recuperación tras reinicio del servidor.
- **¿Cómo se asegura que un refactor no rompe nada?** → Los tests existentes tienen que seguir pasando sin cambiar su lógica ([[GLORY-95 Unir game_types, geometry_types e intentions en data_types|GLORY-95]], [[GLORY-183 Unir constants y timing en metrics|GLORY-183]]).
- **¿Cómo se prueba un test que depende del azar (sorteos)?** → Se evita el azar: `side_effect` con valores fijos, o se verifica una **propiedad** (el orden contiene exactamente a los titulares) y no un valor.

## 6. Cómo ejecutarlos
```
pytest                                   # todos
pytest tests/unit/test_match_pauses.py   # un archivo
pytest -k penalty                        # los que tengan "penalty" en el nombre
pytest -x                                # frena en el primer fallo
TEST_DATABASE_URL=postgresql://... pytest   # contra Postgres real
```
