# Consolidación elo-gol-live

Este repo nace de comparar el código real (no solo los README) de los 6
repos de fútbol de erikolivo: `alertas-apuestas`, `Predicciones-Elo`,
`ole-ole`, `elo-nuevo`, `flow-elo`, `GEM1`. Ninguno estaba en producción,
así que se pudo elegir libremente sin preocuparse por migración.

## Qué se descartó y por qué

- **alertas-apuestas / Predicciones-Elo / ole-ole**: son etapas
  anteriores del mismo diseño. `lpi_engine.py` y `odds_validation.py` de
  ole-ole son los borradores tempranos de lo que hoy son `momentum.py` y
  `cuotas_reales.py`. No tienen código único que no exista ya, mejorado,
  en elo-nuevo/flow-elo/GEM1.
- **elo-nuevo**: mismo diseño que flow-elo/GEM1 pero con solo 8 commits
  (vs 134 de flow-elo) — versión menos iterada, sin las mejoras de cuotas
  reales.

## Base elegida: flow-elo (134 commits)

Se usó como esqueleto porque es la versión más iterada y mejor
documentada de la línea moderna (rating propio Glicko-2 + momentum en
vivo separado de la expectativa pre-partido + resolución de país por
equipo + cuotas reales).

## Qué se trajo de GEM1

- **`storage.py`**: capa única de lectura/escritura de JSON. Se incluye
  como utilidad disponible, pero **no se forzó su uso en el resto de
  los módulos** (ver "Decisión deliberada" abajo).
- **`poisson_model.py`**: la versión de GEM1 pondera el rating por el RD
  (incertidumbre) de Glicko-2 antes de convertirlo en goles esperados
  (`aplicar_rd()`), y usa promedios de goles dinámicos por equipo en vez
  de un promedio de liga fijo. Verificado con pruebas manuales tras la
  integración (ver abajo).

## Bug encontrado y corregido durante la consolidación

El "promedio de goles dinámico" de GEM1 dependía de que el diccionario
de `goal_index` trajera `goles_favor_prom` / `goles_contra_prom` por
equipo. `fetch_data.calcular_goal_index()` sí los calculaba, pero
`goal_index.py::_mezclar()` los descartaba al combinar forma reciente +
temporada — esos dos campos nunca llegaban a `poisson_model.py`, así que
la mejora de GEM1 era código muerto (caía siempre al promedio de liga
por defecto) tanto en GEM1 como en un flow-elo hipotético que la hubiera
adoptado sin revisar. Se corrigió `_mezclar()` para propagar ambos
campos con el mismo blend 60/40 que ya se usaba para `goal_index`.
Confirmado con pruebas manuales: con `goal_index` disponible, la
probabilidad del favorito cambia de forma sensata (61.3% → 88.4% en un
caso de prueba con buen ataque/mala defensa marcados); con RD alto
(equipo con pocos partidos observados) el resultado converge hacia
50/50, como debe ser.

También se actualizó `seleccionar_partidos.py` para pasar `rd_home` /
`rd_away` (que ya calculaba pero no usaba) y los diccionarios completos
de `goal_index` a `evaluar_favorito()`, siguiendo la firma nueva.

## Decisión deliberada: NO se forzó el refactor a storage.py en todo el repo

Al comparar `cerrar_resultados.py`, `monitor.py`, `reporte_diario.py`,
`resumen.py` y `cuota_odds_api.py` entre flow-elo y GEM1, los diffs son
grandes (300-700 líneas). Dos archivos pequeños (`cuota_api_football.py`,
`ratings_store.py`) se verificaron equivalentes -- GEM1 solo los
condensó y les cambió el I/O -- pero no se auditaron línea por línea los
archivos grandes. Ya se encontró un caso real (`team_resolver.py`, ver
abajo) donde el refactor de GEM1 perdió funcionalidad en silencio. Por
eso se prefirió dejar esos módulos con la versión de flow-elo (probada
en 134 commits) en vez de arriesgar una regresión no detectada solo por
prolijidad arquitectónica. `storage.py` queda disponible para quien
quiera hacer ese refactor de forma incremental y verificada, módulo por
módulo.

## Por qué NO se tomó team_resolver.py ni seleccionar_partidos.py completos de GEM1

Se comparó línea por línea. La versión de GEM1 de `team_resolver.py`:

- **No cachea el país en disco** — cada corrida repreguntaría a la API
  el país de equipos ya resueltos antes, gastando cupo innecesariamente.
- No tiene la verificación cruzada por confederación completa (CONMEBOL/
  UEFA/CONCACAF/AFC/CAF) que sí tiene flow-elo -- solo compara contra el
  país directo del rival.

Por eso se mantuvo el `team_resolver.py` de flow-elo tal cual.

## Resumen de origen por archivo

| Archivo | Origen | Cambios |
|---|---|---|
| `poisson_model.py` | GEM1 | ninguno |
| `storage.py` | GEM1 | ninguno (disponible, sin forzar su uso) |
| `goal_index.py` | flow-elo | fix: propaga goles_favor_prom/goles_contra_prom |
| `seleccionar_partidos.py` | flow-elo | adaptado a la firma nueva de poisson_model |
| Todo lo demás | flow-elo | sin cambios |
