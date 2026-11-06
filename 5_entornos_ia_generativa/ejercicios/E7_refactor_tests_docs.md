# E7 — Release Candidate (30 min de taller + integración)

**Sesión:** 16 nov 2026

## Parte 1 — Review de riesgos (8 min)

Prompt a la IA centrado en: secretos, validación, timeouts, errores, complejidad.

Triaje: elige **3 hallazgos reales** y corrígelos. Ignora ruido genérico.

## Parte 2 — README de demo (8 min)

Debe permitir a un desconocido:

1. Crear venv e instalar
2. Configurar `.env`
3. Arrancar API
4. Ejecutar 2 curls de ejemplo
5. Correr tests

## Parte 3 — Tests del camino crítico (8 min)

Mínimo:

- happy path `/analyze`
- error de validación
- (si aplica) fallo de dependencia

`pytest -q` en verde.

## Parte 4 — Tag RC (6 min)

```bash
git tag v0.9.0-rc1
git push origin v0.9.0-rc1   # cuando el remoto exista
```

Crea `RELEASE.md` con: versión, issues conocidos, guion de demo 5–7 min.

## Hecho cuando…

Tag RC + README reproducible + suite verde + guion de exposición.
