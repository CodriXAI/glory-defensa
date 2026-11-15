---
tags: [glory, desarrollado, penales, back, motor]
ticket: GLORY-292
epica: Penales (GLORY-277)
tipo: Story
puntos: 5
reviewer: Orlando Vigetti
estado: Done
---
# GLORY-292 — Ejecutar la tanda de penales en el motor
> Épica: [[Penales (GLORY-277)]] · Bloqueado por [[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]], [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]] (y 283/284) · Fue reportado por Noah, lo implementé yo

## En una frase
Cuando un partido de liga **con penales habilitados** termina empatado, `run_match` juega la tanda **antes** de pasar a `FINISHED`: ronda a ronda, hasta que haya un ganador (máximo 3 rondas y, si sigue el empate, sorteo).

## Qué pedía el ticket
- Una ronda = un penal de cada equipo; tanda de máximo 3 rondas.
- Penales solo en partidos de liga con la opción activa (los amistosos no tienen).
- Reutilizar la lógica de un tick, pero el gol va al marcador de penales (`phome`/`paway`), no al del partido.
- Un penal termina con gol o a los 20 ticks sin gol.
- Después de cada ronda: si convirtió uno solo, gana; si convirtieron los dos o ninguno, otra ronda; tras 3 rondas empatados, se sortea y se le suma 1 gol de penal (nunca queda empatado).
- Espera de unos segundos entre un penal y otro con el estado congelado.
- La pelota, al cruzar la línea de gol, se queda quieta para que el tick detecte el gol.
- `save_match_result` guarda los goles de penales; `make_on_finished` le pasa `phome`/`paway`.

## Dónde está en el código
| Qué | Archivo | Función |
|---|---|---|
| Dónde se decide jugar la tanda | `src/engine/match_runner.py` | `run_match` (después del bucle de ticks) |
| La tanda completa | `src/engine/penalties.py` | `play_shootout` |
| Un penal | `src/engine/penalties.py` | `_play_penalty` |
| Sumar gol al que patea | `src/engine/penalties.py` | `_add_penalty_goal` |
| Un tick sin tocar marcador ni reloj | `src/engine/match_runner.py` | `play_tick` (la comparte `run_tick`) |
| Pelota quieta en la línea de gol | `src/engine/_04_to_physics/simulation.py` | `_move_ball` (la línea de gol cuenta como "contacto") |
| Constantes | `src/engine/_01_to_world/metrics.py` | `PENALTY_MAX_TICKS=20`, `PENALTY_MAX_ROUNDS=3`, `PENALTY_PAUSE_SECONDS=3.0` |
| Guardar el resultado | `src/services/match_start_service.py` | `save_match_result`, `make_on_finished` |

## Cómo funciona
1. En `run_match`: al terminar los ticks, **si** hay `play_shootout`, `state.penalties_enabled` y `score.home == score.away` → avisa `PENALTIES_STARTED`, `await play_shootout(...)`, avisa `PENALTIES_FINISHED`.
2. `play_shootout` llama a `prepare_shootout` ([[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]]) y entra a un `while round_index < 3 and phome == paway`.
3. Cada vuelta: espera `PENALTY_PAUSE_SECONDS` (salvo antes del primero), juega un penal del **host** (patea host, ataja visitante), espera, juega uno del **visitante**.
4. `_play_penalty`: `prepare_turn` ([[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]]), aviso de preparación + espera ([[GLORY-310 Cuadro de preparación antes de cada penal|GLORY-310]]), y luego ticks hasta que haya gol o pasen 20 ticks.
5. Si tras la última ronda `phome == paway`, se sortea con `random.choice([host, away])` y se le suma 1 gol.
6. `run_match` pone `FINISHED` y llama a `on_finished`, que guarda `home_goals`, `away_goals`, `home_penalty_goals`, `away_penalty_goals` **en un solo guardado**.

## Decisiones de diseño
- **Lógica en un módulo aparte (`penalties.py`)**, y `run_match` la recibe por parámetro (`play_shootout`): `penalties.py` importa `play_tick` de `match_runner`; importarse al revés daría un import circular.
- **`play_tick` separado de `run_tick`**: `play_tick` no suma goles al marcador ni avanza el reloj; eso lo decide quien la llama. Así los penales reutilizan la misma lógica del tick.
- **El gol se le suma al que patea, no al "lado"**: el arco es siempre el izquierdo y los dos equipos patean contra él.
- **La pelota queda quieta en la línea de gol**: `detect_goal` corre al terminar el tick, así que el gol se ve en ese mismo tick aunque el tiro termine fuera de los palos.
- **Nunca empatados**: el sorteo final garantiza que siempre hay un ganador.
- **Los penales se resuelven antes de `FINISHED`**: el resultado se guarda una sola vez, con penales.
- **Las columnas de penales en `Match` se movieron a [[R7 - Penales#GLORY-293 — Exponer el resultado y adaptar la tabla|GLORY-293]]**, porque 292 estaba bloqueado (nota del ticket).

## `play_tick` vs `run_tick` vs `_play_penalty` (lo que más pueden preguntar)
`play_tick` es la **primitiva base**: "juega UN tick" (cada jugador decide, se aplican las intenciones de todos juntas, se detecta el gol) y **devuelve** el lado que convirtió. **No** suma el gol, **no** avanza el reloj y **no tiene `await`**.

| | `run_tick` (tiempo reglamentario) | `_play_penalty` (penales) |
|---|---|---|
| Juega el tick | `play_tick` | `play_tick` (la misma) |
| Quién suma el gol | `score.home/away` según `host_side` | `_add_penalty_goal` → `phome/paway`, **al que patea** |
| ¿Avanza el reloj? | Sí (`current_tick += 1`) | No (`current_tick` queda en `total_ticks`) |
| Quién espera y avisa por WS | el bucle de `run_match` | `_play_penalty` (su propio `sleep` y `_notify_safely`) |
| Corte | se acaban los ticks | gol o `PENALTY_MAX_TICKS = 20` (lo que ocurra primero) |

- **Por qué `play_tick` no tiene `await`**: atomicidad. En `asyncio` otra tarea solo puede colarse en un `await`; sin ninguno, el tick corre completo y nadie ve un estado a medias (ver patrón 7 en [[04 - Glosario y patrones]]).
- **Ojo con la confusión típica**: lo que se pasa **por parámetro** a `run_match` es **`play_shootout`** (para evitar el import circular), **no** `play_tick`. `play_tick` es una función normal que `penalties.py` importa de `match_runner.py`.
- **Corrección frecuente**: `run_tick` **no** manda eventos WS ni espera; es síncrona. Los avisos y los `sleep` están afuera, en `run_match` o en `_play_penalty`.

## Qué pasa si falla
- **Falla el aviso (notify)**: `_notify_safely` lo registra y la tanda sigue.
- **El motor muere con un error inesperado**: `_on_task_done` (en `match_start_service.py`) lo registra y guarda el marcador del momento como resultado final para que el partido no quede colgado en `IN_PROGRESS`. No lo hace si la fase ya era `FINISHED` (el error fue guardar el resultado mismo) ni si otro motor tiene el partido en el registro.
- **Partido de liga que falla antes o durante la ejecución**: ver los 3 casos (0-0 si no se pudo preparar, resultado parcial si se cae el motor, nada si se cae el servidor) en [[03 - Preguntas tipo de defensa]].
- **Nadie convierte en 20 ticks**: el penal termina sin gol.
- **Ambos convierten o ambos fallan**: se juega otra ronda (hasta 3).
- **Sigue empatado tras 3 rondas**: sorteo (`random.choice`) y +1 gol de penal.

## Tests
`tests/unit/test_penalties_shootout.py` · `tests/unit/test_penalties_result.py` · `tests/unit/test_match_runner.py`

## Relacionado con
[[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]] · [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]] · [[GLORY-310 Cuadro de preparación antes de cada penal|GLORY-310]] · [[R7 - Penales#GLORY-293 — Exponer el resultado y adaptar la tabla|GLORY-293]] (puntos y exposición del resultado) · [[R7 - Penales#GLORY-291 — Opción de penales en la liga|GLORY-291]] (opción de la liga) · [[GLORY-181 La pelota rebota en los bordes de la cancha|GLORY-181]] (rebote: también afecta a la pelota en cancha).

## Preguntas que me pueden hacer
- ¿Cuándo se juegan los penales y cuándo no?
- ¿Por qué `play_tick` no tiene `await`?
- ¿Por qué el gol va al que patea?
- ¿Qué pasa si después de 3 rondas siguen empatados? ¿Es justo?
- ¿Por qué la lógica vive en otro módulo?
- ¿Qué pasa si el motor se cae en medio de la tanda?

## Con mis palabras (completar)
- Qué hice:
- Por qué así:
