# E9 — Preparación para producción (30 min)

**Sesión:** 12 nov 2026

## Parte 1 — Empaquetado (8 min)

En tu Proyecto III:

- `requirements.txt` revisado
- `Procfile` o comando de start documentado
- nota de versión de Python (3.11/3.12)

## Parte 2 — Secretos (8 min)

- `.env.example` con todas las keys necesarias (vacías)
- Verificar que `.env` está en `.gitignore`
- README: cómo configurar el PaaS

## Parte 3 — Smoke test (8 min)

Script `scripts/smoke_test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
BASE=${1:-http://127.0.0.1:8000}
curl -sf "$BASE/health" | grep -q status
curl -sf -X POST "$BASE/analyze" -H 'Content-Type: application/json' \
  -d '{"text":"smoke test DSIA"}' | grep -q summary
echo OK
```

## Parte 4 — README deploy (6 min)

Sección numerada: build → env vars → start → smoke → rollback.

## Hecho cuando…

Un compañero sigue el README y pasa el smoke en local.
