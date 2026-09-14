# E1 — Procesamiento de datos con pandas (30 min)

**Sesión:** 14 sep 2026  
**Dataset:** `../Datos/ventas.csv`

## Preparación (antes de arrancar)

```bash
cd 1_programacion_avanzada_python
# opcional: trabaja en el notebook 02_pandas_datos.ipynb
# o en un script propio ejercicios/solucion_e1.py
```

## Parte 1 — Diagnóstico (8 min)

1. Carga el CSV con `pathlib.Path` (sin rutas absolutas hardcodeadas).
2. Imprime `shape`, `dtypes` y `isna().sum()`.
3. Lista por escrito (comentario o markdown) **qué filas parecen inválidas** y por qué.

## Parte 2 — Validación (10 min)

Implementa:

```python
def validar_ventas(frame: pd.DataFrame) -> tuple[pd.DataFrame, pd.DataFrame]:
    ...
```

Reglas mínimas:

- `unidades` numérica y `> 0`
- `precio_unitario` numérico y `> 0`
- columnas derivadas: `importe = unidades * precio_unitario` solo en válidos

Devuelve `(validos, errores)`.

**Checkpoint:** con el CSV del curso debes obtener **140 válidas** y **10 inválidas**.

## Parte 3 — Agregaciones (8 min)

Sobre `validos`:

1. Importe total por `region` (ordenado desc).
2. Top 3 `producto` por importe.
3. `cliente_id` con más de una compra.

## Parte 4 — Exportación (4 min)

Escribe:

- `Datos/ventas_limpias.csv`
- `Datos/calidad_datos.json` con:
  - `filas_totales`, `filas_validas`, `filas_invalidas`, `importe_total`

## Extensión si terminas pronto

- Añade `fecha` como `datetime` y un agregado por semana.
- Refactoriza en funciones `cargar` / `validar` / `agregar` / `exportar` (adelanto sesión 22).

## Entrega orientativa

Código en tu repo del Proyecto I + captura o commit donde se vea el JSON de calidad.
