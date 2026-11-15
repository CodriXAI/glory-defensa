---
tags: [glory, inconsistencias]
---
# 05 — Inconsistencias conocidas (para no sorprenderme)
> Volver a [[00 - Inicio]]. Son cosas que **detecté al leer el código y los tickets**; **no se modificó nada**. Conviene saberlas por si preguntan, y decidir con el equipo si se corrigen.

## Back / contrato
| # | Qué pasa | Dónde |
|---|---|---|
| 1 | **Un 0-0 por falla de preparación** en una liga **con penales** cuenta como empate (1 punto c/u), pero el contrato dice que con penales no hay empates | [[R2 - Iniciar y correr la liga\|GLORY-241]] / [[R7 - Penales\|GLORY-293]] |
| 2 | **GLORY-241 tiene criterios de aceptación que se contradicen** ("queda pending" vs "se cierra 0-0"). Los tests muestran que son **dos capas**: `begin_league_match` deja el partido `PENDING` y sin alineación si no se puede preparar, y `start_league_match` lo cierra 0-0 | GLORY-241 |
| 3 | `LOGO_TOO_LARGE` está en el contrato pero **no se valida** en el back | contrato |
| 4 | `main.py` incluye el router de ligas **dos veces** y `CORS_ORIGINS` se envuelve como un solo string | `src/main.py` |
| 5 | `docs/alcance.md` está **desactualizado** | docs |
| 6 | `ABOUT_DESIGN.md` referencia `ABOUT_FORMULAS` que **no existe** | `src/engine/_05_to_docs/` |
| 7 | El **DT elegido vive en memoria** (`coach_registry`): se pierde si se reinicia el servidor | [[R6 - Comportamientos y DT\|GLORY-299]] |
| 14 | **"Ver prueba de comportamiento" no está implementada** (ni en back ni en front) aunque estaba en los requerimientos del Sprint 3 y hay tickets redactados (16/10). Ver [[06 - Comodines para la defensa#2. "¿Por qué no hicieron la prueba de comportamiento (vista previa)?"|comodín 2]] |
| 15 | **`tests/unit/tests_match_player_service.py`** (tests de GLORY-260) se llama `tests_...` y no `test_...`: no coincide con el patrón por defecto de pytest (`test_*.py`) y `pytest.ini` no define `python_files`, así que **probablemente no se ejecuta** en `pytest` normal. Verificar con `pytest --collect-only` y renombrar a `test_match_player_service.py` | GLORY-260 |

## Front ↔ Back
| # | Qué pasa |
|---|---|
| 8 | El front llama a `GET /teams/{id}` y `POST /auth/logout`, que el back **no tiene** |
| 9 | "Editar perfil" lleva a `/profile/edit`, que **no tiene ruta** en `App.jsx` |
| 10 | El README del front menciona ESLint, pero el repo usa **oxlint** |

## Datos del proceso
| # | Qué pasa |
|---|---|
| 11 | Los CSV exportados tienen **44 tickets** (17 desarrollados + 27 revisados), no 47: confirmar contra Jira si falta alguno |
| 12 | En el CSV de revisados el campo "Reviewer" está vacío en casi todos (solo GLORY-242 me nombra); confirmar en Jira. Quién desarrolló cada uno sale del campo "Desarrollador" |
| 13 | [[GLORY-301 Correcciones pendientes de ubicar jugadores\|GLORY-301]] existe porque se mergeó una rama sin aplicar los cambios del review |

## Cómo usarlo en la defensa
Si me preguntan por algo de esta lista: **reconocerlo, explicar por qué quedó así y qué haría** (por ejemplo: persistir el DT en una tabla; unificar el criterio del 0-0). Mostrar que se **detectó** es mejor que no saberlo.
