# E8b — Robustez y logging (30 min)

**Sesión:** 6 nov 2026

## Parte 1 — Correlación (8 min)

En `/analyze`, genera `request_id = str(uuid.uuid4())` y:

- loguealo en INFO
- devuélvelo en el JSON de respuesta

## Parte 2 — Fallo de dependencia (8 min)

Añade un modo o monkeypatch que haga fallar el cliente IA.

La API debe responder **503** (o 502) con `{"error": "...", "request_id": "..."}` sin traceback al cliente.

Escribe un test que fuerce ese camino.

## Parte 3 — Latencia (8 min)

Mide ms de la llamada IA y loguea `latency_ms`.

## Parte 4 — Health rico (6 min)

`GET /health` devuelve checks (`api`, `data_path` o similar). Si un check falla → `status: degraded` y código 503.

## Hecho cuando…

Hay test de fallo de dependencia y `/health` reporta más que un ok vacío.
