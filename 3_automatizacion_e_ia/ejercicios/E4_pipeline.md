# E4 — Pipeline de datos robusto (30 min)

**Sesión:** 5 oct 2026  
**Base:** `../ejemplos/pipeline_ventas.py`

## Parte 1 — Contrato de columnas (8 min)

Antes de transformar, verifica columnas mínimas:

`fecha, region, producto, unidades, precio_unitario, cliente_id`

Si falta alguna → log ERROR + `sys.exit(2)`.

## Parte 2 — Umbral de calidad (8 min)

Añade `--max-error-rate` (float, default `0.5`).

Si `rows_dropped / rows_in > max_error_rate` → exit code `1`.

Prueba manual:

```bash
python ejemplos/pipeline_ventas.py --input Datos/ventas.csv --output-dir /tmp/out --max-error-rate 0.05
# debe fallar con el dataset del curso (10/150 ≈ 0.067)
```

## Parte 3 — Observabilidad (8 min)

- Log a consola **y** a `output-dir/run.log`.
- Mantén `metrics.json`.

## Parte 4 — Test puro (6 min)

Test de `transform()` con DataFrame sintético (sin tocar disco). Assert del report.

## Hecho cuando…

Falláis a propósito el umbral, el log existe y el test unitario pasa.
