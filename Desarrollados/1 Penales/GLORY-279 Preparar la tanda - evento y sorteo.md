---
tags: [glory, desarrollado, penales, back, motor]
ticket: GLORY-279
epica: Penales (GLORY-277)
tipo: Story
puntos: 3
reviewer: Noah Casini
estado: Done
---
# GLORY-279 — Preparar la tanda de penales: evento y sorteo
> Épica: [[Penales (GLORY-277)]] · Siguiente en la cadena: [[GLORY-280 Ubicar jugadores e inyectar comportamientos de penales|GLORY-280]] → [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]]

## En una frase
Deja lista la tanda antes de jugarla: **sortea una sola vez** el orden en que patean y atajan los titulares de cada equipo, deja el marcador de penales en 0-0, y define los avisos `PENALTIES_STARTED` y `PENALTIES_FINISHED` para el WebSocket del partido.

## Qué pedía el ticket
- Sortear el orden de pateadores y arqueros (solo titulares), una vez, y guardarlo en el estado.
- Inicializar el marcador de penales en 0-0.
- Armar los payloads de los dos eventos y una función `notify_penalties(state, started)`.
- `run_match` recibe `notify_penalties` como parámetro opcional (y `launch_match` lo reenvía), igual que `notify_pause`.
- Actualizar el contrato con los dos eventos.

## Dónde está en el código
| Qué | Archivo | Función / clase |
|---|---|---|
| Sorteo y marcador 0-0 | `src/engine/penalties.py` | `prepare_shootout` |
| Estado de la tanda | `src/engine/_02_to_match/state.py` | `PenaltyShootout` (`kicker_order`, `keeper_order`, `prepared`) |
| Marcador de penales | `src/engine/_02_to_match/state.py` | `Score.phome`, `Score.paway` |
| Payloads de los avisos | `src/realtime/match_broadcast.py` | `build_penalties_started_payload`, `build_penalties_finished_payload`, `notify_penalties` |
| Envío tolerante a fallas | `src/engine/match_runner.py` | `_notify_penalties_safely` |
| Parámetro opcional | `src/engine/match_runner.py` y `src/services/match_start_service.py` | `run_match(notify_penalties=...)`, `launch_match` |

## Cómo funciona
1. `prepare_shootout(state)` mira si `state.penalty_shootout.prepared` ya es `True`: si sí, **no hace nada**.
2. Si no, arma un `PenaltyShootout` nuevo y, para cada equipo, toma sus **titulares** (`is_starter`) y usa `random.sample` dos veces por separado: una para el orden de pateo y otra para el de atajada.
3. Marca `prepared = True`, guarda la tanda en el estado y pone `score.phome = 0` y `score.paway = 0`.
4. Los avisos salen por el **canal del partido** (no por el global): `{"event": "PENALTIES_STARTED", "match_id": 12}` y `{"event": "PENALTIES_FINISHED", "match_id": 12}`.

## Decisiones de diseño
- **Sorteo una sola vez y con flag `prepared`**: si se pide preparar de nuevo, el orden no cambia. Evita que un evento repetido reinicie la tanda.
- **Solo titulares**: los suplentes no patean ni atajan.
- **Orden de pateo y de atajada sorteados por separado**: así un jugador no necesariamente ataja en la misma ronda en que patea.
- **Canal del partido y no el global**: igual que `PAUSE_STARTED/FINISHED`; solo les importa a quienes miran ese partido.
- **`notify_penalties` opcional con default `None`**: los tests y partidos que no lo pasan siguen funcionando igual que antes.
- **El ticket cambió durante el trabajo**: la versión original decía que "el back escucha un evento WS de penales"; quedó tachada. Al final el motor llama directo a las funciones. Es lo que explica el texto tachado del ticket.

## Qué pasa si falla
- **Se pide preparar dos veces** → no se vuelve a sortear, el orden no cambia.
- **Falla el envío del aviso** (`notify_penalties` lanza error) → `_notify_penalties_safely` lo registra con `logger.exception` y el partido sigue (lo importante es llegar a `FINISHED`).
- **No se pasa `notify_penalties`** → el partido se juega igual, simplemente no avisa.

## Tests
`tests/unit/test_penalties_preparation.py` · `tests/unit/test_match_broadcast.py` · `tests/unit/test_match_runner.py`

## Relacionado con
[[R7 - Penales#GLORY-281 — Marcador de penales|GLORY-281]] (agregó `phome`/`paway` a `Score`) · [[R5 - Pausas y estado en vivo#GLORY-218 — Eventos WS de las pausas|GLORY-218]] (mismo patrón de avisos para las pausas) · [[GLORY-292 Ejecutar la tanda de penales en el motor|GLORY-292]] (quien llama a `prepare_shootout`) · en el front, `startPenalties`/`finishPenalties` ([[F3 - Mis tickets vistos desde el front]]).

## Preguntas que me pueden hacer
- ¿Por qué el sorteo se hace una sola vez? ¿Qué pasa si llega dos veces?
- ¿Por qué los avisos van por el canal del partido?
- ¿Qué pasa si el aviso falla? ¿Se cancela el partido?

## Con mis palabras (completar)
- Qué hice:
- Por qué así:
