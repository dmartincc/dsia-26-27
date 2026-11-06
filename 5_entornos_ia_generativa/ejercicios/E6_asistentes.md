# E6 — Taller con asistentes sobre código existente (30 min)

**Sesión:** 16 nov 2026  
**Prompts:** `../prompts/catalogo_prompts.md`

## Reglas

- Trabaja sobre **tu Proyecto III**, no sobre un repo vacío.
- No commits de código que no entiendas.
- Todo uso relevante de IA va a `AI_USAGE.md`.

## Parte 1 — Elegir objetivo (5 min)

Elige un smell concreto (función > 40 líneas, sin tests, nombres pobres, duplicación).

Anota en `AI_USAGE.md` el fichero y el problema en 2 líneas.

## Parte 2 — Explicar (7 min)

Usa el prompt “Explicar código existente”. Pega un resumen de 5–8 líneas en `AI_USAGE.md` (no el volcado entero).

## Parte 3 — Plan + un cambio (10 min)

1. Pide un **plan de refactor** (sin código).
2. Implementa **solo el ítem #1** del plan (tú o con IA, pero revisado).
3. Diff pequeño: ideal < 80 líneas.

## Parte 4 — Tests asistidos (8 min)

1. Pide 3 tests pytest.
2. Ejecuta `pytest -q`.
3. Si un test alucina APIs inexistentes → bórralo y documenta el rechazo en `AI_USAGE.md`.

## Hecho cuando…

Hay commit/PR de refactor + `AI_USAGE.md` con explicación, plan, aceptación/rechazo.
