# Selección de modelos LLM (BE-46)

Comparativa de `gemini-2.5-flash-lite` (primario actual), `gpt-4o-mini` (fallback actual) y
`gemini-2.5-flash` (candidato "más grande", misma familia que el primario) corriendo los 4
datasets de `evals/` de forma **aislada por modelo** (sin el fallback de `AIExtractorContext`,
ver `evals/compare-models.ts`), para poder atribuir cada resultado a un único modelo.

Corridas reales, sin mocks, contra las APIs de Google e OpenAI — `pnpm eval:compare-models`.
Reportes crudos: `evals/reports/2026-10-30-model-comparison.md`/`.json` (corrida completa) y
`evals/reports/2026-10-30-model-comparison-flash-supplement.md`/`.json` (muestra adicional
reducida de `gemini-2.5-flash`, ver más abajo por qué).

## ⚠️ Hallazgo urgente, separado de la elección de modelo

**La misma `GEMINI_API_KEY` de `.env` usada para esta comparativa es la de producción**
(confirmado con el usuario). El proyecto de Google Cloud está en el **free tier**, que tiene un
**límite de 20 requests/día por modelo** (`GenerateRequestsPerDayPerProjectPerModel-FreeTier`,
confirmado con el mensaje real de la API: *"Generate content requests per day limit exceeded on
Free Tier"*, `retryDelay` del orden de 14-15 horas) — además del límite por minuto ya documentado
en `ai.google.dev/gemini-api/docs/rate-limits` (10 req/min para `gemini-2.5-flash-lite`, 5 req/min
para `gemini-2.5-flash`).

**Esto significa que en producción, después de ~20 búsquedas reales en el día, `gemini-2.5-flash-lite`
(el modelo primario) empieza a devolver 429 y el sistema cae en silencio al fallback `gpt-4o-mini`
por el resto del día** — comportamiento indistinguible de una app funcionando normal (el usuario
final nunca ve un error, `AIExtractorContext` hace el fallback transparente), pero con costo y
latencia de `gpt-4o-mini` en vez de los de `gemini-2.5-flash-lite` sin que nadie se entere. Esta
comparativa lo confirmó en vivo: la corrida completa de hoy agotó la cuota diaria de
`gemini-2.5-flash-lite` después de apenas ~16 llamadas reales, y la de `gemini-2.5-flash` después
de ~20.

**Recomendación inmediata, independiente de qué modelo se elija:** habilitar facturación
(pay-as-you-go) en el proyecto de Google Cloud de este API key. El tier pago de Gemini no tiene
límite diario de requests (solo el límite por minuto, mucho más alto) — sin esto, **cualquier**
modelo de Gemini elegido como primario va a degradar a las ~20 búsquedas/día, sin importar cuál sea
"mejor" en esta comparativa.

## Limitación metodológica de esta corrida

Por el límite diario de arriba, **la muestra de los dos modelos de Gemini quedó muy por debajo del
dataset completo** en esta corrida — a diferencia de `gpt-4o-mini`, que sí corrió sin ningún error
sobre los 4 datasets completos (160/160 casos). La tabla de abajo muestra el tamaño de muestra real
(`n`) junto a cada número — **tratar los resultados de Gemini como señal direccional, no como
comparación estadísticamente equivalente a la de `gpt-4o-mini`**. Una vez habilitada la facturación
(ver arriba), se puede re-correr `pnpm eval:compare-models` con el dataset completo para una
comparación sólida de verdad.

## Resultado por operación

### extract-intent (Agente 1 — extracción de texto libre)

| Modelo | n ejecutados | operation | property_type | currency | country | bedrooms | zones (prec/rec) | key_features (prec/rec) | p50 | p95 |
|---|---|---|---|---|---|---|---|---|---|---|
| gemini-2.5-flash-lite | 10/78 | 100.0% | 100.0% | 100.0% | 100.0% | 100.0% | 65.0% / 65.0% | 100.0% / 100.0% | 813ms | 4336ms |
| gpt-4o-mini | **78/78** | 97.4% | 94.9% | 97.4% | 94.9% | 98.7% | 64.7% / 64.7% | 99.4% / 100.0% | 1250ms | 1956ms |
| gemini-2.5-flash | 8/78* | 100.0% | 100.0% | 100.0% | 100.0% | 100.0% | 100.0% / 93.8% | 100.0% / 100.0% | 1935ms | 6880ms |

\*la tanda de 4 casos adicionales (`--max-cases 4`, para no gastar la poca cuota diaria que
quedaba) re-testeó `ei-001`–`ei-003` (ya incluidos en los 8 originales) antes de agotar la cuota en
`ei-004` — confirmó el mismo 100% en los 3 repetidos, sin agregar casos únicos nuevos. n=8 es el
conteo de casos únicos.

### zone (Agente 2 — intención de zona)

| Modelo | n ejecutados | zona correcta | dormitorios | operacion | p50 | p95 |
|---|---|---|---|---|---|---|
| gemini-2.5-flash-lite | 0/30 | sin datos (cuota agotada antes de correr esta operación) | — | — | — | — |
| gpt-4o-mini | **30/30** | 100.0% | 100.0% | 96.7% | 1265ms | 1896ms |
| gemini-2.5-flash | 7/30 | 100.0% | 100.0% | 100.0% | 2106ms | 3199ms |

### segment (Agente 0 — segmentador)

| Modelo | n ejecutados | cantidad correcta | texto exacto | p50 | p95 |
|---|---|---|---|---|---|
| gemini-2.5-flash-lite | 0/27 | sin datos | — | — | — |
| gpt-4o-mini | **27/27** | 100.0% | 66.7% | 1099ms | 1859ms |
| gemini-2.5-flash | 1/27 | 100.0%* | 100.0%* | 1910ms | 1910ms |

\*un solo caso — no concluyente, incluido solo por transparencia.

### excel-mapping

| Modelo | n ejecutados | accuracy (field→header) | p50 | p95 |
|---|---|---|---|---|
| gemini-2.5-flash-lite | 10/25 | 99.1% | 1230ms | 1786ms |
| gpt-4o-mini | **25/25** | 97.1% | 2650ms | 3338ms |
| gemini-2.5-flash | 0/25 | sin datos (cuota agotada antes de correr esta operación) | — | — |

## Costo (USD)

Una búsqueda real de producción hace 1 llamada de `extract-intent` + 1 de `zone` por segmento (ver
BE-09, hoy son 2 llamadas separadas). `segment`/`excel-mapping` son flujos distintos, no se suman.

| Modelo | Costo/llamada extract-intent | Costo/llamada zone | **Costo estimado/búsqueda** | **Costo est./1.000 búsquedas** |
|---|---|---|---|---|
| gemini-2.5-flash-lite | $0.000163 | sin datos (asumido $0 — subestimado, ver nota) | $0.000163* | $0.16* |
| gpt-4o-mini | $0.000234 | $0.000125 | $0.000359 | $0.36 |
| gemini-2.5-flash | $0.000511 | $0.000259 | $0.000771 | $0.77 |

\*El costo de `gemini-2.5-flash-lite` está **subestimado**: no hay ninguna llamada exitosa de
`zone` en la muestra (cuota agotada antes de llegar a ese dataset), así que el costo real por
búsqueda es más alto que $0.000163 — probablemente similar al costo por llamada de
`extract-intent` del mismo modelo (ambas usan el mismo modelo con prompts de tamaño comparable),
es decir, cerca de $0.00030-0.00035/búsqueda, no $0.000163.

## Decisión

**Mantener `gemini-2.5-flash-lite` como primario y `gpt-4o-mini` como fallback — sin cambios en la
configuración actual.** La evidencia disponible (aunque parcial para Gemini) no muestra una razón
de peso para cambiar:

- En los campos donde `gemini-2.5-flash-lite` sí tiene muestra (`extract-intent`, 10 casos;
  `excel-mapping`, 10 casos), su accuracy es igual o mejor que la de `gpt-4o-mini` sobre el dataset
  completo, y es el modelo más barato de los tres.
- `gpt-4o-mini` demostró ser la opción más confiable operativamente: **0 errores de infraestructura
  en 160/160 casos**, sin ningún límite de cuota diario encontrado — justifica mantenerlo como
  fallback real (no solo teórico).
- `gemini-2.5-flash` (el candidato "más grande") mostró accuracy más alta en `zone`
  (100% vs. 96.7% de `gpt-4o-mini` en `operacion`, aunque con muestra chica) y es 2-5× más caro que
  `gemini-2.5-flash-lite` — no justifica reemplazar al primario con la evidencia actual, pero es un
  candidato razonable para una futura etapa de re-rank dirigido (BE-43) si esa ventaja de accuracy
  se confirma con muestra completa.

**Esta decisión queda condicionada al punto crítico de arriba:** sin habilitar facturación en el
proyecto de Gemini, el primario va a seguir degradando a `gpt-4o-mini` después de ~20
búsquedas/día en producción, sin importar cuál sea "mejor" en papel. Ese es el problema a resolver
primero, no la elección de modelo en sí.

**Próximo paso recomendado:** una vez habilitada la facturación, re-correr
`pnpm eval:compare-models` con el dataset completo (sin `--max-cases`) para una comparación con
muestra pareja entre los 3 modelos — en especial `zone`/`segment`/`excel-mapping` de
`gemini-2.5-flash`, que hoy casi no tienen datos.
