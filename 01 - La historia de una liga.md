---
tags: [glory, historia]
---
# 01 — La historia de una liga (el hilo conductor)
> Volver a [[00 - Inicio]]

Si tengo que explicar "qué hice" en 2 minutos, cuento la vida de **una liga** y marco **dónde entra cada ticket mío (🟢)** y **cuáles revisé (🔵)**.

## Acto 1 — Se crea y la gente se une
1. 🔵 Se **crea** la liga: modelo y restricciones ([[R1 - Ligas|GLORY-118]]); la opción de penales ([[R7 - Penales|GLORY-291]]).
2. 🔵 Si es privada, se **verifica la contraseña** ([[R1 - Ligas|GLORY-127]]).
3. 🔵 Se **listan** y se **ven** ([[R1 - Ligas|GLORY-178, 270, 312]]) y se configura la **alineación** ([[R1 - Ligas|GLORY-229, 230]]).
4. 🔵 Los equipos **se unen**; cuando se completa el cupo (o el dueño inicia), se **inicia la liga** ([[R2 - Iniciar y correr la liga|GLORY-239]]): se arma el **fixture** de todos contra todos (por eso el cupo es **par**).

## Acto 2 — Se juegan las fechas
5. 🔵 Se **arranca una fecha** y sus partidos en paralelo ([[R2 - Iniciar y correr la liga|GLORY-242, 241]]).
6. Cada partido corre en el **motor** (ticks de 0,5 s). Acá vive la mayor parte de lo que hice:
   - 🟢 **Reorganización y física**: [[GLORY-95 Unir game_types, geometry_types e intentions en data_types|GLORY-95]], [[GLORY-183 Unir constants y timing en metrics|GLORY-183]] (+ 🔵 [[R4 - Motor refactor|184, 185]]), rebote de la pelota [[GLORY-181 La pelota rebota en los bordes de la cancha|GLORY-181]], saque tras gol [[GLORY-180 Saque después de un gol|GLORY-180]].
   - 🟢 **Pausas**: 3 por partido, reloj detenido ([[GLORY-216 Lógica de ticks en pausas|GLORY-216]]), cambio de lados en el Half-Time ([[GLORY-217 Cambio de posición tras Half-Time|GLORY-217]]) y 🔵 avisos en vivo ([[R5 - Pausas y estado en vivo|GLORY-218, 298]]).
   - 🟢 **Sustituciones**: en el motor ([[GLORY-215 Función substitute_player en el motor|GLORY-215]]) y por endpoint, aplicada o planificada ([[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]]).
   - 🟢 **Comportamientos**: validación por tipo ([[GLORY-288 Separar comportamientos de jugador y de DT|GLORY-288]]) y 🔵 cambio en vivo ([[R6 - Comportamientos y DT|GLORY-259, 260]]).
   - 🟢 **DT**: primitivas ([[GLORY-296 Primitivas de director técnico|GLORY-296]]) y 🔵 elegirlo y ejecutarlo ([[R6 - Comportamientos y DT|GLORY-299, 297]]).
7. Si hay **empate y la liga tiene penales**: 🟢 [[GLORY-279 Preparar la tanda - evento y sorteo|GLORY-279]] (preparar) → [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]] (ubicar) → [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]] (jugar) → [[GLORY-310 Cuadro de preparación antes de cada penal|GLORY-310]] (cuadro previo) (+ [[GLORY-301 Correcciones pendientes de ubicar jugadores|GLORY-301]] correcciones; 🔵 [[R7 - Penales|281, 293]]).
8. Termina el partido: 🔵 se **guarda el resultado** y se **avanza de fecha** ([[R2 - Iniciar y correr la liga|GLORY-243]]), se suma al **ranking global** si la liga es pública ([[R3 - Ranking global|GLORY-205]]).

## Acto 3 — Se cierra
9. 🔵 Termina el último partido: se **cierra la liga** ([[R2 - Iniciar y correr la liga|GLORY-244]]), la **tabla** queda final ([[R2 - Iniciar y correr la liga|GLORY-237]]) y el **ranking** se ve en ([[R3 - Ranking global|GLORY-204]]).

## Y en el medio, los "arreglos"
- 🟢 [[GLORY-224 Fix test_join_friendly_motor|GLORY-224]]: un test intermitente (sesiones de SQLite).
- 🟢 [[GLORY-311 Botón Volver al Inicio en los lobbies|GLORY-311]]: el botón "Volver al Inicio" (único de front).

## Línea de tiempo del partido (el motor)
```
tick 0 ──────── 25% ───────── 50% ───────── 75% ────────── 100%
                 │             │             │              │
          Cooling Break     Half-Time    Cooling Break    ¿empate + penales?
          (10 s, cambios)  (20 s, cambios  (10 s, cambios)   │
                           + cambio de lado)                 ▼
                                                        tanda (≤ 3 rondas + sorteo)
                                                             │
                                                             ▼
                                                         FINISHED → guarda resultado
```
En cada pausa: 1) el DT actúa ([[R6 - Comportamientos y DT#GLORY-297 — Ejecutar el DT en el motor|GLORY-297]]), 2) se aplican los cambios planificados ([[GLORY-214 Endpoint de sustitución - efectuar o planificar|GLORY-214]]), 3) se avisa `PAUSE_STARTED` ([[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas|GLORY-218]]), 4) se espera, 5) (si es Half-Time) se cambian los lados ([[GLORY-217 Cambio de posición tras Half-Time|GLORY-217]]), 6) `PAUSE_FINISHED`.

## Mi mensaje en una frase (completar)
> Mi parte fue…
