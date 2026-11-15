---
tags: [glory, defensa, comodines]
---
# 06 — Comodines para la defensa final
> Volver a [[00 - Inicio]] · Preguntas de práctica: [[03 - Preguntas tipo de defensa]] · Cosas conocidas que no cerraron: [[05 - Inconsistencias conocidas]]

**Para qué sirve**: respuestas armadas para las preguntas "de equipo" (no de un ticket puntual) que más pueden caer. Cada una tiene **hechos verificados**, **qué decir**, **qué no decir** y **a qué nota del vault ir** para respaldarla.

> Regla de oro: **no inventar**. Todo lo marcado ✅ lo verifiqué en el código o en los documentos del proyecto. Lo marcado ⚠️ es una inferencia o algo que conviene confirmar con el equipo **antes** del jueves.

## Mapa: comodín → dónde se respalda en el vault
| Comodín | Respaldo en el vault |
|---|---|
| 1. Cantidad par de equipos | [[03 - Preguntas tipo de defensa]] (A2) · [[R1 - Ligas#GLORY-118 — Modelo y persistencia de liga\|GLORY-118]] · [[R2 - Iniciar y correr la liga]] (GLORY-239) |
| 2. Por qué no hicimos la prueba de comportamiento | [[R6 - Comportamientos y DT]] · [[GLORY-288 Separar comportamientos de jugador y de DT\|GLORY-288]] (validación) · [[05 - Inconsistencias conocidas]] (#14) · Comodín 5 |
| 3. Por qué solo 2 y 5 minutos | [[GLORY-216 Lógica de ticks en pausas\|GLORY-216]] (`pause_schedule_for`) · [[GLORY-183 Unir constants y timing en metrics\|GLORY-183]] (`metrics.py`) · [[R2 - Iniciar y correr la liga]] (las fechas se juegan en secuencia) |
| 4. "No me acuerdo bien" | [[01 - La historia de una liga]] · [[04 - Glosario y patrones]] · [[02 - Tabla de todos los tickets]] |
| 5. (Extra) Aislamiento del código de usuario | [[GLORY-288 Separar comportamientos de jugador y de DT\|GLORY-288]] · [[GLORY-296 Primitivas de director técnico\|GLORY-296]] · [[R6 - Comportamientos y DT]] |

---
## 1. "¿Por qué la cantidad de equipos es siempre par?"

**Hechos ✅**
- Cupos válidos: **4, 6, 8, 10, 12** (contrato: error `INVALID_LEAGUE_CAPACITY`; en la base `CheckConstraint capacity_valid`).
- Para iniciar hace falta **cantidad par y mínimo 4** (`OddTeamsCountError`, `MIN_TEAMS_TO_START`).
- El generador de fixture (`build_round_robin`) lanza `ValueError` con cantidad impar.
- Nuestro propio "How To Play" lo dice así: *"siempre par, para que en cada fecha todos jueguen"*.

**Qué decir (30 segundos)**
> "El fixture es todos contra todos. Con cantidad par hay N−1 fechas y en cada una **todos juegan**: N/2 partidos por fecha, todos en paralelo. Con cantidad impar sobra un equipo en cada fecha, y habría que modelar fechas libres: un rival `null`, fechas con distinta cantidad de partidos, equipos con distinta cantidad de partidos jugados en un mismo momento y más casos al avanzar de fecha. Era complejidad sin aportar a los objetivos del laboratorio, así que lo resolvimos **como regla de diseño** y la validamos en tres capas: la base, el servicio y el algoritmo."

**Si insisten ("¿no es una limitación?")**
> "Sí, es una limitación **elegida**: se podría soportar impar con un 'equipo fantasma' (*bye*), pero decidimos priorizar otras funcionalidades. Está documentado como decisión de diseño."

**Qué no decir**: que impar "es imposible". No lo es; es una decisión de alcance.

**Plus**: la liga con cupo lleno **arranca sola**; el cupo par también hace que ese arranque automático siempre sea válido.

---
## 2. "¿Por qué no hicieron la prueba de comportamiento (vista previa)?"

**Hechos ✅**
- Estaba entre los requerimientos del Sprint 3: *"Pruebas de comportamiento: spawnear una cancha y ver un comportamiento jugar antes de anotarlo en una liga"*.
- **Hay tickets redactados** (16/10): T1 servicio en el motor (`run_preview`, 41 frames), T2 endpoint `POST /behaviors/{id}/preview`, T3 UI en el front (mini cancha). Están en el documento `tickets-prueba-de-comportamiento` del proyecto.
- **No está implementada**: no hay `preview` en el back ni en el front.
- En ese mismo documento ya se había decidido **no hacer sandbox** y se anotaba el riesgo para la demo.

**⚠️ Confirmar antes del jueves**: si esos tickets llegaron a **Jira** y en qué estado están. Los profes miran Jira: si existen, la respuesta tiene que coincidir con lo que ven (por ejemplo "quedó en backlog / fuera del sprint").

**Estructura de respuesta (honesta + técnica)**
1. **Reconocerlo sin vueltas**: "Estaba en los requerimientos, la diseñamos (hay tickets), y no llegó a implementarse en este sprint."
2. **Priorización**: "Priorizamos lo que forma el **flujo central** del producto: ligas completas, pausas, sustituciones, penales y DT. La prueba es una funcionalidad de **conveniencia**: el sistema funciona completo sin ella."
3. **Argumentos técnicos reales**:
   - **Fidelidad limitada**: el diseño de la prueba es **un jugador y una pelota, sin compañeros ni rivales**; no permite evaluar comportamientos que dependen de otros jugadores (marcar, pasar). Su valor de validación es acotado.
   - **Ya existe validación temprana**: al guardar, el servidor valida el código (AST: sintaxis, `on_tick`, bucles, imports, primitivas del tipo correcto; [[GLORY-288 Separar comportamientos de jugador y de DT\|GLORY-288]]), y si en un partido el código falla, el jugador queda en `idle` y el partido sigue. El error grueso se detecta antes de jugar.
   - **Riesgo de seguridad**: la prueba ejecutaría código del usuario **fuera de un partido, dentro del request**, y sin sandbox (decisión del equipo). Sumaba superficie de riesgo; en cambio en partido el código corre acotado por las validaciones y el motor.
   - **Alternativa funcional hoy**: se puede ver jugar un comportamiento en un **amistoso** (en la cancha real, con compañeros y rival). ⚠️ Tiene una limitación: un amistoso necesita un segundo usuario que se una (se puede usar una segunda cuenta).
4. **Cierre con plan**: "Está diseñada: reutiliza `run_tick` y la misma validación; son tres tickets (motor, endpoint, front). Sería lo primero del próximo sprint."

**Qué NO decir**
- ❌ "Nunca estuvo en el alcance" → la consigna la lista y los tickets existen.
- ❌ "No es importante / no sirve" → mejor "es de menor prioridad que el flujo central".
- ❌ Inventar que fue una decisión técnica tomada desde el principio. Lo verdadero y fuerte: **priorizamos y quedó como deuda diseñada**.

**Frase corta de emergencia**
> "Fue una decisión de priorización: la dejamos diseñada pero la pusimos después del flujo central de ligas, pausas, penales y DT, porque el producto funciona sin ella y su valor es limitado con un solo jugador y sin rivales."

---
## 3. "¿Por qué los partidos duran solo 2 o 5 minutos?"

**Hechos ✅**
- El creador **sí elige** la duración (liga y amistoso): `match_duration` ∈ {2, 5}; otro valor → 400 `INVALID_SETTINGS`.
- 2 min → **240 ticks**; 5 min → **600 ticks**; 1 tick = 0,5 s de juego = 0,5 s **reales**: el partido se transmite en **tiempo real** por WebSocket.
- Las pausas son **proporcionales** (25 / 50 / 75 %) y suman 40 s que **no** cuentan en el reloj.
- La lista de valores es **una sola constante** (`VALID_MATCH_DURATIONS_MINUTES` en `metrics.py`); el calendario de pausas (`pause_schedule_for`) se **deriva** de esa constante; el front la replica en `constants/matchDurations.js`.

**Cuánto dura una liga (orden de magnitud, sin penales)** — cálculo mío, a partir de que las fechas se juegan en secuencia y los partidos de una fecha en paralelo:
| Equipos | Fechas | Con 2 min (≈ 2 min 40 s c/pausas) | Con 5 min (≈ 5 min 40 s c/pausas) |
|---|---|---|---|
| 4 | 3 | ≈ 8 min | ≈ 17 min |
| 8 | 7 | ≈ 19 min | ≈ 40 min |
| 12 | 11 | ≈ 29 min | ≈ 62 min |

**Qué decir**
> "El requerimiento era que el creador pueda **elegir** la duración, y lo cumplimos con dos opciones. Las acotamos por tres razones técnicas: (1) el partido se **transmite en tiempo real**, un tick cada 0,5 s, así que la duración del partido es tiempo real de servidor; (2) la **liga se juega sola** y las fechas son consecutivas, así que la duración del partido se multiplica por la cantidad de fechas: una liga de 12 equipos con partidos de 5 minutos ronda la hora; (3) el **estado de un partido en juego vive en memoria**: cuanto más corto es el partido, menor es la ventana en la que un reinicio del servidor lo afecta. Además, con valores discretos el calendario de pausas y la validación son exactos y fáciles de probar."

**Si insisten ("¿por qué no 90 minutos / configurable libre?")**
> "Es un parámetro: está en una constante y el calendario de pausas se deriva de ella, así que ampliar los valores es un cambio chico en el back y en la lista del front. No lo abrimos a valores libres para mantener la validación simple y porque no teníamos un criterio de carga para fundamentar valores más largos."

**⚠️ Para confirmar antes**
- **De dónde salieron 2 y 5**: si vinieron de una respuesta de los profesores en clase o del PO, **esa es la mejor justificación** ("el cliente lo definió así"). Si fue una decisión del equipo, usar los argumentos de arriba.
- **No afirmar** que hicimos pruebas de carga: no las hay. Las razones (1)–(3) son de diseño, no mediciones.

**Qué no decir**: "lo hicimos corto para que no falle". Mejor: "lo acotamos por el tiempo real de transmisión y la duración total de una liga".

---
## 4. "Me preguntan algo que no recuerdo bien" (cómo salir del paso con dignidad)

### Protocolo de 6 pasos
1. **Respirar y repetir la pregunta** con tus palabras ("¿lo que querés saber es qué pasa si…?"). Ganás 5 segundos y confirmás que entendiste.
2. **Ubicar la capa**: ¿es de **route, service, repository, model o motor**? Decir en voz alta dónde estaría ("eso debería resolverlo el service…").
3. **Ir al patrón** ([[04 - Glosario y patrones]]): casi todo cae en uno de los 10 patrones.
4. **Decir lo que sí sé** y **separar lo que no**: "El flujo general es este… el detalle exacto del nombre/valor no lo recuerdo".
5. **Mostrar dónde lo verificaría**: "Lo miraría en `league_run_service.py`, en la función que arranca la fecha". Pedir abrir el archivo si lo permiten.
6. **Cerrar con la decisión de diseño**: "Lo hicimos así por simplicidad / atomicidad / alcance".

### Frases puente
- "Déjenme ubicarlo: eso pasa en la capa de servicio, después del commit…"
- "No recuerdo el valor exacto, pero la lógica es: …"
- "Lo tengo en el ticket; la idea central es…"
- "Esa parte la hizo [compañero]; yo la revisé y entiendo que…" (para los ticket revisados: [[R1 - Ligas]]… [[R7 - Penales]]).

### Respuestas comodín por tipo de pregunta
| Si preguntan… | Respuesta base (casi siempre sirve) |
|---|---|
| "¿Por qué lo hicieron así?" | **Simplicidad y alcance académico**, separación de responsabilidades; está en la sección *Decisiones de diseño* del ticket |
| "¿Qué pasa si falla?" | Depende de la capa: **servicio** → rollback y error de contrato (`AppError`); **motor** → se registra y el partido sigue (`_notify_safely`, DT que falla, jugador en `idle`); **evento WS** → nunca frena el partido |
| "¿Y si dos pedidos llegan a la vez?" | Motor sin `await` = atómico; base con `for_update` o UPDATE condicional (*claim*) |
| "¿Se guarda en la base?" | Preguntarse: ¿vive en **memoria** (`match_registry`, `coach_registry`, plan de sustitución, cambio de comportamiento) o en la **base** (partidos, resultados, ligas, alineaciones)? |
| "¿Cómo lo sabe el front?" | Evento WS **global** → el front vuelve a pedir el dato (REST); o WS **del partido** → `TICK_UPDATE` ([[F2 - Como habla el front con el back]]) |
| "¿Por qué no lo hicieron más completo?" | "Mantuvimos un enfoque académico y entendible; lo dejamos documentado como mejora" |
| "¿Cómo lo probaron?" | Tests unitarios por capa (`tests/unit/test_*.py`); la nota de cada ticket tiene su archivo de tests |

### Ubicador rápido del código (qué carpeta abrir)
| Tema | Dónde |
|---|---|
| Endpoints | `src/routes/` |
| Reglas de negocio | `src/services/` |
| Consultas | `src/repositories/` |
| Tablas | `src/models/` |
| Bucle del partido y pausas | `src/engine/match_runner.py` |
| Estado del partido y sustituciones | `src/engine/_02_to_match/state.py` |
| Física | `src/engine/_04_to_physics/simulation.py` |
| Penales | `src/engine/penalties.py`, `src/penalty_behaviors.py` |
| DT | `src/engine/_03_to_builder/coach_api.py` |
| Avisos en vivo | `src/realtime/match_broadcast.py` |
| Constantes de tiempo y pausas | `src/engine/_01_to_world/metrics.py` |

### Lo que nunca hay que hacer
- ❌ Inventar un nombre de función o un número: mejor "no recuerdo el valor exacto".
- ❌ Contradecir lo que dice Jira o el contrato: si dudás, "tendría que verificarlo".
- ❌ Echarle la culpa a un compañero. Usar "el equipo decidió".

---
## 5. Extra recomendado: "¿Cómo aíslan el código de los usuarios?"
La consigna dice que el grupo **tiene que poder explicarlo en la demo** (cómo se aísla, qué límites tiene y qué pasa si falla). Es una pregunta muy probable, y la parte de validación es de mi ticket [[GLORY-288 Separar comportamientos de jugador y de DT\|GLORY-288]].

**Hechos ✅**
- **Capa 1: validación al guardar (AST)**: rechaza Python inválido, falta de `on_tick`, **`for`/`while`**, **imports**, **comprehensions**, el texto **`"__"`**, más de **5000 caracteres** y **primitivas del otro tipo** (jugador vs. DT).
- **Capa 2: ejecución restringida**: el código corre con un conjunto **chico de funciones permitidas** (`abs`, `min`, `max`, `len`, `round`, `float`, `int`, `sum`, `bool`, `str`, `list`, `tuple`, `dict`, `any`, `all`, `sorted`) y los módulos/clases `math`, `Point2D`, `Vector2D`, `random`. Cada API (`PlayerAPI` / `CoachAPI`) solo tiene sus propias primitivas.
- **Qué pasa si falla**: si el `on_tick` de un jugador lanza una excepción, el motor asigna **`idle`** para ese tick y el partido sigue; si falla el DT, el equipo queda igual y el partido sigue.
- **Inmutabilidad** (`frozen=True`) en los datos que ve el usuario, como segunda protección.

**Limitaciones ⚠️ (decirlas antes de que las encuentren)**
- **No hay sandbox de proceso ni límite de CPU/tiempo**: el código corre en el mismo proceso del servidor (decisión del equipo, documentada).
- Al no permitirse **bucles**, el caso típico de código que no termina queda cerrado. **No probé** qué pasa con una recursión infinita (en Python debería terminar en `RecursionError`, que cae en el manejo de excepciones → `idle`); conviene probarlo antes del jueves si preguntan.

**Qué decir**
> "Aislamos en dos capas: validación estática al guardar, que prohíbe bucles, imports y accesos peligrosos, y ejecución con un entorno restringido donde cada jugador solo ve sus primitivas. Si el código falla, el jugador queda quieto ese tick y el partido sigue. Reconocemos que no hay un sandbox de proceso ni un límite de tiempo de CPU; lo documentamos como limitación conocida."

---
## Checklist antes del jueves
- [ ] Confirmar con el equipo **de dónde salieron los 2 y 5 minutos** (comodín 3).
- [ ] Mirar en **Jira** si existen los tickets de prueba de comportamiento y en qué estado están (comodín 2).
- [ ] Practicar en voz alta cada "Qué decir" (máx. 30–40 segundos cada uno).
- [ ] (Opcional) Probar la recursión infinita en un comportamiento (comodín 5).
- [ ] Revisar [[05 - Inconsistencias conocidas]]: ahí está lo que **no** cierra.
