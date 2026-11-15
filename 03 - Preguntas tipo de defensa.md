---
tags: [glory, defensa, preguntas]
---
# 03 — Preguntas tipo de defensa (con respuesta corta)
> Volver a [[00 - Inicio]] · Respuestas "de equipo" y salidas de emergencia: [[06 - Comodines para la defensa]]. Método: **tapar la respuesta**, contestar en voz alta, comparar.

## A. Arquitectura (las 4 preguntas generales)

### A1. ¿Por qué routes → services → repositories → models? ¿Cómo se comunican?
- **routes**: capa HTTP. Recibe el request, valida con **Pydantic**, obtiene el usuario del **token** y llama a un service. No tiene reglas de negocio.
- **services**: las **reglas de negocio** (quién puede qué, en qué estado, qué orden). Acá se hace `commit`/`rollback` y se **publican los eventos después del commit**.
- **repositories**: solo hablan con la base (`select`, `update`). Sin reglas.
- **models**: las tablas (SQLAlchemy 2.0) y sus restricciones (`CheckConstraint`, `UniqueConstraint`).
- **Ventaja principal**: **separación de responsabilidades** → cada capa se prueba sola (un service se testea sin HTTP), se puede cambiar una capa sin romper las otras y el equipo trabaja en paralelo sin pisarse.
- **Flujo**: `route → service → repository → model/DB` y la respuesta vuelve por el mismo camino. Los errores son `AppError` y un **handler global** los convierte en JSON (`{"error": "...", "message": "..."}`) con el status correcto.
- **Excepción conocida**: los errores del **motor** (`SubstitutionError`, `MatchInitializationError`) **no** son `AppError`: el motor no sabe de HTTP; el service los traduce ([[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]]).
- Ejemplo mío: sustitución = `substitution_endpoint` (route) → `validate_substitution` / `apply_substitution` (service) → `state.substitute_player` (motor).

### A2. ¿Por qué los cupos de ligas son siempre pares? ¿Cómo se justifica técnicamente?
> Versión corta para decir en voz alta: [[06 - Comodines para la defensa#1. "¿Por qué la cantidad de equipos es siempre par?"|comodín 1]].
- El fixture es un **todos contra todos** (algoritmo del círculo / *round-robin*). Con **N par**: **N−1 fechas** y **N/2 partidos por fecha**: **todos juegan en todas las fechas**.
- Con **N impar** cada fecha **sobra un equipo** ("fecha libre"/*bye*): habría que inventar un equipo fantasma, partidos con rival `null`, fechas con distinta cantidad de partidos, equipos con distinta cantidad de partidos en un mismo momento (tabla difícil de comparar) y más casos en `advance_league`.
- Lo impusimos **en tres capas** (defensa en profundidad):
  1. **Base**: `CheckConstraint capacity_valid` → `max_teams IN (4,6,8,10,12)` ([[R1 - Ligas|GLORY-118]]).
  2. **Service**: al iniciar, si hay cantidad impar → `OddTeamsCountError` (y `MIN_TEAMS_TO_START = 4`) ([[R2 - Iniciar y correr la liga|GLORY-239]]).
  3. **Algoritmo**: `build_round_robin` lanza `ValueError` con impar.
- Es una **decisión de alcance** (documentada en la aclaración del 08/10), no una limitación del algoritmo: con *byes* se podría hacer, pero cuesta complejidad que el laboratorio no pedía.

### A3. Si en una liga todos los partidos fallan y quedan 0-0, ¿gana el de mayor team ID?
- **Corrección**: el desempate final es el de **menor `team_id`**, no el mayor.
- `standings_sort_key` ordena por: **puntos → diferencia de gol → goles a favor → menos goles en contra → `-team_id`**, con `reverse=True`. Al negar el id, el id **más chico** queda **primero**.
- Si todos los partidos quedaron 0-0: todos tienen los mismos puntos (1 por partido), diferencia 0, goles 0 → decide **el menor `team_id`**.
- Es **determinista** (la tabla siempre es la misma) y **no tiene empates** en la posición.
- **Matiz honesto**: es una regla técnica de desempate, no deportiva. Está documentada y se usa para que la tabla siempre tenga un orden único.
- **Ojo**: un 0-0 por falla de preparación en una liga **con penales** cuenta como empate (1 punto c/u) aunque el contrato dice que con penales no hay empates ([[05 - Inconsistencias conocidas]]).

### A4. SQLAlchemy 2.0: lo básico que usamos
Ver [[04 - Glosario y patrones]] → sección SQLAlchemy.
- Modelos con `Mapped[...]` y `mapped_column(...)`; un `Base` común con convención de nombres.
- Consultas con `select(...)` y `session.scalars(...)` (**no** el estilo viejo `session.query`).
- Una **sesión por request** (`get_db`): se abre al empezar y se cierra al terminar.
- `selectinload` para traer relaciones sin N+1; `.in_()`, `.distinct()`, `order_by`, `offset/limit`.
- `session.get(..., with_for_update=True)` para bloquear una fila.
- **UPDATE condicional** como "reclamo" (`claim_pending_league_match`).
- **Alembic** para migraciones (cadena lineal; cabeza actual `8f23a963366d`).

---
## B. Preguntas por tema

### Penales
- ¿Cuándo hay penales? → Solo partido de **liga** con `penalties_enabled` y **empate** al terminar ([[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]]).
- ¿Qué pasa si siguen empatados tras 3 rondas? → Se sortea y se suma 1 gol de penal; nunca queda empatado.
- ¿Cuántos puntos da ganar por penales? → 2 puntos; el que pierde por penales recibe 0 (como cualquier derrota); el empate sin penales da 1 y ganar en tiempo reglamentario 3 ([[R7 - Penales|GLORY-293]]). Los goles de penales no cuentan en goles a favor/en contra.
- ¿Por qué 0,3 m entre pelota y pateador? → Menor al alcance mínimo (0,5 m con CONTROL 20) ([[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]]).
- ¿Por qué `play_shootout` entra por parámetro a `run_match`? → Evita un import circular (`penalties.py` importa `play_tick` de `match_runner.py`).
- ¿Por qué `play_tick` no tiene `await`? ¿Se pasa por parámetro? → No se pasa por parámetro (lo que se pasa es `play_shootout`). No tiene `await` para ser **atómica**: nadie puede ver el estado a medias.
- ¿Qué diferencia hay entre `run_tick` y un tick de penales? → Ambos usan `play_tick`; `run_tick` suma al marcador del partido y avanza el reloj; en penales el gol va a `phome/paway` (al que patea) y el reloj no avanza. Los avisos WS y los `sleep` los hace quien llama.
- ¿Por qué el primer aviso salía tarde? → Salía después del primer tick: ya se había resuelto el tiro ([[GLORY-310 Cuadro de preparación antes de cada penal|GLORY-310]]).

### Pausas y sustituciones
- ¿Cuándo hay pausas? → 25 %, 50 % y 75 % (240 → 60/120/180; 600 → 150/300/450 ticks). Cualquier otro total: sin pausas ([[GLORY-216 Lógica de ticks en pausas|GLORY-216]]).
- ¿Qué pasa con el reloj en la pausa? → Se detiene; no hay `TICK_UPDATE`.
- ¿Por qué `time.monotonic()`? → No salta si cambia la hora del sistema.
- ¿Por qué la sustitución no tiene `await`? → Chequeo y cambio son **una sola operación**; nadie se puede colar en el medio ([[GLORY-215 Función substitute_player en el motor|GLORY-215]]).
- ¿201 vs 202? → 201: se aplicó (había pausa). 202: quedó planificado para la próxima ([[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]]).
- ¿Qué pasa si el plan ya no es válido cuando empieza la pausa? → Se descarta silenciosamente.
- ¿Por qué se invierten los `side` de **todos** (suplentes incluidos) en el Half-Time? → Para que los que entren después ya tengan el lado correcto ([[GLORY-217 Cambio de posición tras Half-Time|GLORY-217]]).
- ¿Cómo sigue el marcador si cambian los lados? → Se acredita a quien **marcó** según `host_side`, no a `HOME`.

### DT y comportamientos
- ¿Qué puede hacer un DT? → Cambiar el comportamiento de un titular y sustituir; solo sobre su equipo ([[GLORY-296 Primitivas de director técnico|GLORY-296]]).
- ¿Cómo se evita que un jugador use las primitivas del DT? → Dos capas: validación AST al guardar y que `PlayerAPI` no tiene esos métodos ([[GLORY-288 Separar comportamientos de jugador y de DT|GLORY-288]]).
- ¿Se pierde el DT si se reinicia el servidor? → Sí: está en `coach_registry` (memoria) ([[R6 - Comportamientos y DT|GLORY-299]]).
- ¿Qué pasa si el código del DT lanza un error? → El equipo queda como estaba y el partido sigue (`_run_coaches`).

### Ligas
- ¿Cómo se evita arrancar dos veces el mismo partido? → UPDATE condicional (`claim_pending_league_match`) ([[R2 - Iniciar y correr la liga|GLORY-241]]).
- ¿Y si falla el fixture al iniciar? → Una transacción: la liga queda `PENDING` sin partidos.
- ¿Cuándo se publica un evento? → **Después** del commit.
- ¿Por qué un cambio de alineación no afecta un partido en curso? → Se **copia** al partido al arrancar ([[R1 - Ligas|GLORY-230]]).
- ¿Cuándo suma al ranking global? → Solo ligas **públicas** y solo la primera vez que termina el partido ([[R3 - Ranking global|GLORY-205]]).

### Motor
- ¿Por qué un solo reflejo alcanza en el rebote? → Máx. 15 m/tick contra 60 m de cancha ([[GLORY-181 La pelota rebota en los bordes de la cancha|GLORY-181]]).
- ¿Quién saca después de un gol? → El que recibió el gol; la pelota se corre hacia su arco ([[GLORY-180 Saque después de un gol|GLORY-180]]).
- ¿Por qué falla "a veces" `test_join_friendly_motor`? → 3 sesiones compartiendo una conexión SQLite en memoria; en Postgres no pasa ([[GLORY-224 Fix test_join_friendly_motor|GLORY-224]]).

### Si un partido falla en plena ejecución: ¿se guarda el resultado parcial o termina 0-0?
Depende de **cuándo** falla (tres casos):
1. **Falla antes de empezar (preparación)** → se cierra **0-0 `FINISHED`** con `save_match_result(session, match_id, 0, 0)` y se llama a `advance_league` para no trabar la liga (`start_league_match`, [[R2 - Iniciar y correr la liga|GLORY-241]]). No hay marcador parcial porque nunca se jugó.
2. **El motor se cae durante el partido** → `_on_task_done` (`match_start_service.py`) lo registra y llama a `on_finished(state)` con el **marcador del momento**: se guarda el **resultado parcial** con `save_match_result`, el partido pasa a `FINISHED` y la liga avanza. No lo hace si la fase ya era `FINISHED` (el error fue guardar el resultado) ni si el registro todavía tiene el estado de ese partido (lo tiene otro motor). En `run_match`, el `finally` libera el registro **después** de persistir.
3. **Se cae el servidor entero** → no se guarda nada: el estado vivía en memoria. El partido queda `IN_PROGRESS` en la base y la liga trabada. No encontré código de recuperación al arrancar (`lifespan`/`startup`): limitación conocida; mejora posible = al iniciar, cerrar o reiniciar los `IN_PROGRESS`.
- **No es una caída**: un jugador o DT que falla en un tick queda en `idle` / sin cambios; un aviso WebSocket que falla solo se registra.
- **Matiz**: el parcial se guarda tal cual; si estaba empatado en una liga con penales queda como empate (misma inconsistencia #1 de [[05 - Inconsistencias conocidas]]).

### "¿Qué pasaría si…?" (genéricas)
- …se cae el servidor en medio de un partido → `_on_task_done` guarda el marcador actual para que no quede `IN_PROGRESS` para siempre.
- …falla un `notify` (WebSocket) → se registra el error; el partido **no** se detiene.
- …llegan dos pedidos a la vez → depende: motor sin `await` (atómico); base con `for_update` o UPDATE condicional.
- …el front manda datos inválidos → Pydantic responde 422/400 con código de error del contrato; el back **no confía** en el front.

## Para repasar
- [ ] Puedo contar la vida de una liga sin mirar ([[01 - La historia de una liga]]).
- [ ] Puedo explicar los 4 patrones: `for_update`, claim, evento después del commit, notify tolerante a fallas.
- [ ] Puedo explicar la diferencia 201/202.
- [ ] Puedo decir por qué `play_tick` no tiene `await`.
