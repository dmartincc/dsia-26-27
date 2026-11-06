# Sesión 12 nov 2026 — E2E III: despliegue y puesta en producción (Render)

| Recurso | Fichero |
| --- | --- |
| Demo | [`plantilla_proyecto/`](plantilla_proyecto/) (`app.py`, `ingest.py`, tests) · [`../sesiones/2026-11-16_despliegue/ejemplos/packaging/`](../sesiones/2026-11-16_despliegue/ejemplos/packaging/) |
| Ejercicio de clase | [`ejercicios/E9_despliegue.md`](ejercicios/E9_despliegue.md) |
| Datos | [`../3_automatizacion_e_ia/Datos/ventas.csv`](../3_automatizacion_e_ia/Datos/ventas.csv) → `data/clean.csv` (se genera en el **build**) |
| Base previa | [`01_arquitectura_e2e.md`](01_arquitectura_e2e.md) (contratos, mock por defecto) · [`02_robustez_logging.md`](02_robustez_logging.md) (`/health`, logs con `request_id`) |
| Plataforma | [Render](https://render.com) — *Web Service* de Python, plan gratuito |
| Enunciado | [`../proyectos/proyecto_iii/`](../proyectos/proyecto_iii/) |

## Historia de la sesión (léela primero)

Hoy no aprendes “a pulsar el botón de Deploy”. Aprendes a que tu servicio **arranque en una máquina que no es la tuya**, configurado solo por variables de entorno, y a **demostrar** que funciona (y cómo volver atrás) sin entrar al servidor:

```text
1. Empaquetas: dependencias, Python, build y start fijos   →  Parte A1  (requirements · .python-version · gunicorn)
2. Separas configuración y secretos del código              →  Parte A2  (variables de entorno · .env.example)
3. Defines qué es un release y cómo se deshace              →  Parte A3  (CI verde → tag → deploy → health → smoke → rollback)
4. Lo despliegas en Render                                  →  Parte B   (dashboard o render.yaml)
5. Lo verificas y lo operas desde fuera                     →  Parte B   (smoke contra la URL pública · logs · rollback)
```

**Regla de oro:** si para que funcione en Render tuviste que cambiar código “solo para producción”, o no sabes volver a la versión anterior en 2 minutos, **aún no está desplegado: está subido**.

En clase: exposición (~30 min, Parte A + demo en Render) → ejercicio [`E9`](ejercicios/E9_despliegue.md) (~30 min) → deploy del Trabajo Final (resto de la sesión).

---

## 0. Objetivos

### Obligatorios (Parte A)

1. **Empaquetar** el servicio de forma reproducible: dependencias acotadas, versión de Python fijada, comando de build y de arranque escritos.
2. Sacar toda la **configuración y los secretos** a variables de entorno, con un `.env.example` como contrato.
3. Seguir una **estrategia de release** verificable: CI verde → tag → deploy → `/health` → smoke → rollback.

### Deseables (Parte B)

4. Arrancar en local **como en producción** (`gunicorn`, `$PORT`, sin `debug`).
5. Desplegar la plantilla en **Render** (dashboard y/o *Blueprint* `render.yaml`).
6. Configurar **secretos** y **health check** en Render y leer los **logs** de la plataforma.
7. Pasar el **smoke test** contra la URL pública y ensayar un **rollback**.
8. Dejar el despliegue del **Trabajo Final (Proyecto III)** documentado en el README.

---

## 1. Por qué importa en DSIA

```text
Tema 2  →  pytest + CI                 (confianza en el código)
Tema 3  →  job CLI, exit codes, cron   (señales para máquinas)
E2E I   →  flujo completo + contratos  (funciona en mi máquina, sin claves)
E2E II  →  robustez + logs + health    (falla bien y se sabe qué pasó)
E2E III →  despliegue (hoy)            (funciona en otra máquina, y se puede demostrar)
```

Todo lo que construimos en E2E I y II **se usa hoy**: Render no ve tu terminal; solo ve el **código del repo**, las **variables de entorno**, el **exit code del build**, la respuesta de **`/health`** y lo que escribes en **`stdout`**.

| Pieza que ya tienes | Para qué la usa Render |
| --- | --- |
| `requirements.txt` | Build: instalar dependencias |
| `ingest.py` con exit codes | Build: si sale `≠ 0`, el deploy **falla** y sigue la versión anterior |
| `AI_PROVIDER=mock` por defecto | Arrancar sin claves (y con clave, solo si la configuras) |
| `/health` barato y sin dependencias | *Health check*: decidir si la nueva versión recibe tráfico |
| `logging` a `stdout` con `request_id` | Pestaña *Logs* del dashboard |
| Tests en CI | Puerta antes del deploy |

| Idea | Analogía |
| --- | --- |
| Empaquetado | Receta con cantidades exactas: otra cocina obtiene el mismo plato |
| Build | Preparar la masa en la cocina central |
| Start command | Encender el horno de la tienda |
| Variables de entorno | Ajustes del horno: se cambian sin reescribir la receta |
| Secretos | La llave de la caja: nunca pegada en la receta |
| Health check | El encargado prueba el horno antes de abrir la tienda |
| Smoke test | Probar un bocado antes de servir al primer cliente |
| Rollback | Volver a la receta de ayer porque la de hoy sale quemada |

---

# PARTE A — Tres conceptos clave

> Pregunta guía: ***¿puede arrancarlo otra máquina sin ti, y puedo demostrar que funciona (y deshacerlo)?***  
> Orden: **empaquetar → configurar → liberar**. Cómo se hace en Render viene en la Parte B.

---

## A1 — Empaquetado reproducible (local ≠ producción)

Una plataforma como Render hace siempre lo mismo: **clona el repo → ejecuta el build → ejecuta el start → espera a que `/health` responda**. Si algo de eso depende de tu portátil, falla.

```text
git push ──► Render clona el commit ──► BUILD  (pip install + ingest)  ──► START (gunicorn app:app)
                                          │ exit ≠ 0 → deploy fallido          │
                                          ▼                                   ▼
                                   sigue la versión anterior        GET /health == 200 ? → recibe tráfico
```

| Local (E2E I) | Producción (Render) |
| --- | --- |
| `python app.py` → `app.run(debug=True)` | `gunicorn app:app` (servidor WSGI, sin `debug`) |
| Puerto fijo `8000` | Puerto que te da la plataforma: **`$PORT`** |
| Escucha en `127.0.0.1` | Escucha en **`0.0.0.0`** (si no, nadie llega desde fuera) |
| Un proceso | Varios *workers* (`--workers 2` o `WEB_CONCURRENCY`) |
| `.env` en disco | Variables de entorno del dashboard |
| Tu venv con lo que hayas instalado | Solo lo que dice `requirements.txt` |
| Tu versión de Python | La que fijes en `.python-version` (si no, la que elija la plataforma) |
| Disco persistente | Disco **efímero**: lo que escribas en runtime se pierde al redeploy/reinicio |

Ficheros mínimos del empaquetado:

| Fichero | Contenido | Por qué |
| --- | --- | --- |
| `requirements.txt` | `Flask>=3.0,<4`, `gunicorn>=22,<24`, … | Instalación reproducible (versiones **acotadas**) |
| `.python-version` | `3.12` | Misma versión mayor/menor en local, CI y Render |
| Build command | `pip install -r requirements.txt && python ingest.py …` | Dependencias + datos listos antes de arrancar |
| Start command | `gunicorn app:app --bind 0.0.0.0:$PORT` | Cómo arranca el servicio |
| `.env.example` | Todas las variables, sin valores secretos | Contrato de configuración (A2) |
| `scripts/smoke_test.sh` | `curl` a `/health` y `/analyze` | Verificación post-deploy (A3) |

Reglas:

- **El código de `app.py` no cambia** entre local y Render: `app.run(debug=True)` queda bajo `if __name__ == "__main__":` y `gunicorn` nunca lo ejecuta.
- `debug=True` **nunca** en producción: expone un depurador interactivo.
- Si el build no se puede reproducir en un clon limpio con estos comandos, no se reproducirá en Render.

> `Procfile` (`web: gunicorn …`) y `runtime.txt` son convenciones de Heroku y de otros PaaS. **Render no los lee**: allí el start command se escribe en el dashboard o en `render.yaml`. Puedes mantener el `Procfile` como documentación del comando (lo pide `E9`), pero la fuente de verdad en Render es su configuración.

**Checklist:** sé qué ejecuta Render en build y en start · el start usa `$PORT` y `0.0.0.0` · mi `app.py` no tiene ramas “si estoy en producción”.

---

## A2 — Configuración y secretos fuera del código

Todo lo que cambia entre entornos (local, CI, Render) es **configuración** y vive en **variables de entorno**. El código lee la configuración; no la contiene.

| Variable | Local | CI | Render | ¿Secreto? |
| --- | --- | --- | --- | --- |
| `AI_PROVIDER` | `mock` | `mock` | `mock` (o `real`) | No |
| `AI_API_KEY` | vacío (o tu clave en `.env`) | **no existe** | solo si `AI_PROVIDER=real` | **Sí** |
| `LOG_LEVEL` | `DEBUG` | `INFO` | `INFO` | No |
| `DATA_PATH` | `data/clean.csv` | `data/clean.csv` | `data/clean.csv` | No |
| `PORT` | (no hace falta) | — | **la pone Render** | No |
| `PYTHON_VERSION` | — | — | opcional (alternativa a `.python-version`) | No |

`.env.example` (se sube al repo, **sin valores secretos**):

```bash
# Proveedor de IA: mock (por defecto) | real
AI_PROVIDER=mock
# Solo si AI_PROVIDER=real. Nunca en el repo.
AI_API_KEY=
LOG_LEVEL=INFO
DATA_PATH=data/clean.csv
```

Reglas:

- `.env` va en `.gitignore`. `.env.example` va al repo y lista **todas** las variables que lee el código.
- Valores por defecto **seguros**: sin variables configuradas, el servicio arranca en `mock` (contrato 3 de E2E I).
- Un secreto que ha llegado a git **está comprometido** aunque borres el commit: se **rota** (se genera una clave nueva).
- El servicio **no imprime** su configuración al arrancar si contiene secretos (lista “No hacer” de E2E II).
- Si falta una variable obligatoria (p. ej. `AI_PROVIDER=real` sin `AI_API_KEY`), el servicio falla **al arrancar** con un mensaje claro, no en la primera petición.

```python
import os

AI_PROVIDER = os.getenv("AI_PROVIDER", "mock")
AI_API_KEY = os.getenv("AI_API_KEY")          # None si no está

if AI_PROVIDER == "real" and not AI_API_KEY:
    raise RuntimeError("AI_PROVIDER=real requiere AI_API_KEY")   # falla al arrancar, no en /analyze
```

**Checklist:** `.env.example` lista todas las variables · `git grep -i "api_key="` no devuelve ninguna clave · el servicio arranca en Render sin ninguna variable configurada.

---

## A3 — Release: health, smoke y rollback

Un **release** es una versión concreta (un commit con tag) que se despliega, se verifica y, si falla, se deshace. No es “lo último que subí”.

```text
1. CI verde (pytest sin red)            ──► si falla, no se despliega
2. Tag semver  (v0.3.0)                 ──► versión identificable
3. Deploy en Render                     ──► build (exit 0) + start
4. Health check  GET /health → 200      ──► Render decide si recibe tráfico
5. Smoke test contra la URL pública     ──► tú decides si el release es bueno
6. ¿Falla? Rollback a v0.2.x            ──► desde el dashboard, sin tocar código
```

| Comprobación | ¿Quién la hace? | Qué demuestra | Qué **no** demuestra |
| --- | --- | --- | --- |
| Build `exit 0` | Render | Dependencias y datos listos | Que la app arranque |
| Health check `/health` | Render (automático) | El proceso está vivo y responde | Que `/analyze` funcione |
| Smoke test | Tú / CI, tras el deploy | El endpoint crítico respeta el contrato | Todos los casos (eso son los tests) |
| Logs | Tú, en el dashboard | Qué pasó en cada petición (`request_id`) | — |

**Health check y `degraded`** (conexión con E2E II, B6):

| `/health` devuelve | Efecto en Render |
| --- | --- |
| `200` (`ok` o `degraded`, opción A) | La nueva versión recibe tráfico; `degraded` se ve en el cuerpo y en el log |
| `503` (opción B) | Durante el deploy: la nueva versión **no** recibe tráfico y el deploy falla. Con el servicio ya en marcha: Render puede **reiniciar** la instancia |

Por eso, si usas `/health` como health check de Render, la opción A (`200` + `"status": "degraded"`) suele ser la más segura. Responde `503` solo cuando la instancia **no pueda servir** de ninguna forma.

Rollback:

- En Render, cada deploy queda en el historial del servicio (*Events*). Puedes **volver a un deploy anterior** desde allí sin hacer commits.
- Rollback ≠ arreglar: primero **vuelves** a la versión buena y después arreglas con calma en una rama.
- Si el release cambió el formato de los datos o del contrato, el rollback puede romper a los clientes: por eso los contratos de E2E I se versionan.

**Checklist:** sé qué demuestra cada comprobación · el health check de Render apunta a `/health` · sé hacer rollback sin tocar el repo.

---

## A4 — Mapa de los 3 conceptos

```text
A1 empaquetado     →  otra máquina lo construye y lo arranca igual
A2 configuración   →  el mismo código sirve en local, CI y Render; sin secretos en el repo
A3 release         →  cada versión se verifica (health + smoke) y se puede deshacer
        │
        └── Render (Parte B) lo ejecuta: build → start → health check → logs → rollback
```

**Siguiente paso en clase:** [`ejercicios/E9_despliegue.md`](ejercicios/E9_despliegue.md).

---

# PARTE B — Desplegar y operar en Render

> La Parte A dice *qué* debe cumplir un servicio desplegable.  
> La Parte B enseña *cómo se hace en Render* (ensayo local → servicio → secretos → health check) y *cómo se opera desde fuera* (smoke, logs, rollback).

---

## B1 — Anatomía: qué se añade a la plantilla

| Pieza | Rol | Concepto |
| --- | --- | --- |
| `requirements.txt` (en la carpeta del proyecto) | `Flask`, `gunicorn`, `pandas`… acotados | Empaquetado |
| `.python-version` | `3.12` | Runtime fijado |
| `.env.example` | Contrato de configuración | Configuración |
| `.gitignore` | `.env`, `data/clean.csv`, `__pycache__/` | Sin secretos ni artefactos en el repo |
| `data/raw/ventas.csv` | Entrada de la ingesta, **dentro** del proyecto | Build autocontenido |
| `render.yaml` (opcional) | Servicio descrito como código (*Blueprint*) | Infraestructura reproducible |
| `scripts/smoke_test.sh` | `curl` a `/health` y `/analyze` contra una URL | Verificación post-deploy |
| Sección `## Despliegue` del README | Pasos numerados + URL pública | Evidencia |

```text
mi_proyecto/
  app.py
  ingest.py
  test_app.py
  requirements.txt
  .python-version
  .env.example
  .gitignore
  render.yaml            # opcional (B4)
  data/raw/ventas.csv
  scripts/smoke_test.sh
  README.md
```

> Para el Trabajo Final usa un **repo propio** con esta estructura en la raíz. Así el build no depende de rutas fuera del proyecto (`../../3_automatizacion_e_ia/…`) que no existen en tu repo.

---

## B2 — Demo en vivo: ensayo de producción en local

Antes de tocar Render, arranca **exactamente** como lo hará Render. Si falla aquí, fallará allí (y aquí es más rápido de depurar).

```bash
cd mi_proyecto                        # o 4_desarrollo_end_to_end/plantilla_proyecto
python -m venv .venv-prod && source .venv-prod/bin/activate
pip install -r requirements.txt       # venv limpio: solo lo que dice requirements.txt

# BUILD (lo mismo que pondrás en Render)
python ingest.py --input data/raw/ventas.csv --output data/clean.csv
echo "exit=$?"                        # 0 → sigue · ≠ 0 → en Render el deploy fallaría

# START (lo mismo que pondrás en Render)
export PORT=10000
gunicorn app:app --bind 0.0.0.0:$PORT --workers 2
```

```bash
# En otra terminal: smoke contra el "producción local"
bash scripts/smoke_test.sh http://127.0.0.1:10000
```

| Qué observar | Por qué |
| --- | --- |
| `pip install` en venv limpio no se queja | `requirements.txt` está completo |
| `gunicorn` arranca sin `debug` y en `$PORT` | Mismo comando que en Render |
| Los logs salen por la terminal (`stdout`) | Es lo que verás en *Logs* de Render |
| El smoke imprime `OK` | El contrato se cumple con el servidor de producción |
| Sin `.env` el servicio responde con `provider: mock` | Valores por defecto seguros |

---

## B3 — Render paso a paso (dashboard)

1. **Sube el repo a GitHub** (público o privado) con todos los ficheros de B1 y CI en verde.
2. En Render: **New → Web Service** → conecta tu cuenta de GitHub → elige el repo.
3. Rellena la configuración:

| Campo | Valor |
| --- | --- |
| Name | `dsia-<tu-proyecto>` (forma la URL `https://dsia-<tu-proyecto>.onrender.com`) |
| Language / Runtime | `Python 3` |
| Branch | `main` |
| Root Directory | vacío (o la subcarpeta si el proyecto no está en la raíz) |
| Build Command | `pip install -r requirements.txt && python ingest.py --input data/raw/ventas.csv --output data/clean.csv` |
| Start Command | `gunicorn app:app --bind 0.0.0.0:$PORT` |
| Instance Type | `Free` |

4. **Environment** → añade las variables de `.env.example` (`AI_PROVIDER=mock`, `LOG_LEVEL=INFO`…). Los secretos (`AI_API_KEY`) solo si usas un proveedor real.
5. **Advanced → Health Check Path**: `/health`.
6. **Create Web Service** y sigue el build en directo en *Logs*.
7. Cuando el estado sea **Live**, copia la URL y pasa el smoke (B6).

Qué saber del plan gratuito:

| Característica | Consecuencia | Qué hacer |
| --- | --- | --- |
| Se duerme tras ~15 min sin tráfico | La primera petición puede tardar ~1 min (*cold start*) | Abre la URL un rato antes de la presentación |
| Disco efímero | Lo que escribe la app en runtime se pierde al reiniciar o redesplegar | Genera `data/clean.csv` en el **build**; no guardes estado en disco |
| Recursos limitados (RAM/CPU) | Modelos grandes o muchos *workers* no caben | 1–2 workers; modelos pequeños o proveedor externo |
| Auto-deploy en cada push a `main` | Un push roto puede llegar a producción | CI verde antes de mergear; o auto-deploy solo tras pasar CI |

---

## B4 — Infraestructura como código: `render.yaml` (Blueprint)

Lo mismo que B3, pero **escrito en el repo**: reproducible, revisable en un PR y sin “lo configuré en el dashboard y no me acuerdo”.

```yaml
# render.yaml (en la raíz del repo)
services:
  - type: web
    name: dsia-e2e
    runtime: python
    plan: free
    branch: main
    # rootDir: 4_desarrollo_end_to_end/plantilla_proyecto   # solo si el proyecto no está en la raíz
    buildCommand: >
      pip install -r requirements.txt &&
      python ingest.py --input data/raw/ventas.csv --output data/clean.csv
    startCommand: gunicorn app:app --bind 0.0.0.0:$PORT --workers 2
    healthCheckPath: /health
    envVars:
      - key: AI_PROVIDER
        value: mock
      - key: LOG_LEVEL
        value: INFO
      - key: AI_API_KEY
        sync: false        # Render pide el valor en el dashboard; nunca va en el YAML
```

Uso: en Render, **New → Blueprint** → elige el repo → Render lee `render.yaml` y crea el servicio.

| Dashboard (B3) | Blueprint (B4) |
| --- | --- |
| Rápido para la primera vez | Reproducible y versionado con el código |
| La config vive solo en Render | La config está en el repo (sin secretos) |
| Difícil de revisar | Cambios revisables en un PR |

En el curso: dashboard para la demo; **Blueprint recomendado** para el Trabajo Final.

---

## B5 — Secretos y logs en Render

**Secretos:**

- Se añaden en **Environment** del servicio (o con `sync: false` en el Blueprint). Cambiar una variable provoca un **redeploy**.
- Para varios servicios que comparten configuración: **Environment Groups**.
- Nunca en `render.yaml`, en el README ni en el código. En el README se documenta **el nombre** de la variable, no su valor.

**Logs:**

Render recoge todo lo que tu proceso escribe en `stdout`/`stderr`. El `setup_logging()` de E2E II (`StreamHandler(sys.stdout)`) ya está listo:

```text
2026-11-12 12:01:03 INFO    dsia.e2e analyze_ok request_id=3f2a latency_ms=4.1 provider=mock
2026-11-12 12:01:09 WARNING dsia.e2e analyze_failed request_id=9b1c code=invalid_input status=400
```

| Necesito… | Dónde |
| --- | --- |
| Ver por qué falló el build | *Events* → el deploy fallido → log del build |
| Seguir una petición | *Logs* → buscar `request_id=…` (el que devolvió la API al cliente) |
| Ver errores de arranque (variable que falta, import roto) | *Logs* justo después del *Start* |
| Ver el estado del health check | *Events* / estado del servicio |

> Un `FileHandler` a `run.log` en Render **no sirve**: el disco es efímero y no tienes terminal en la máquina. En producción, los logs van a `stdout`.

---

## B6 — Smoke test post-deploy

El mismo script de `E9`, ahora contra la URL pública:

```bash
#!/usr/bin/env bash
# scripts/smoke_test.sh
set -euo pipefail
BASE=${1:-http://127.0.0.1:8000}
curl -sf --max-time 90 "$BASE/health" | grep -q status          # 90 s: margen para el cold start
curl -sf -X POST "$BASE/analyze" -H 'Content-Type: application/json' \
  -d '{"text":"smoke test DSIA"}' | grep -q summary
echo OK
```

```bash
bash scripts/smoke_test.sh https://dsia-e2e.onrender.com
# OK  → release bueno
# (sin OK, exit ≠ 0) → rollback (B7)
```

Opcional (nivel *Sólido*): un job de GitHub Actions que lance el smoke contra la URL tras cada deploy, o a mano con `workflow_dispatch`:

```yaml
# .github/workflows/smoke.yml
name: smoke
on: workflow_dispatch
jobs:
  smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: bash scripts/smoke_test.sh "${{ vars.PROD_URL }}"
```

> Conexión con la sesión 5: igual que el job terminaba con `exit 0/1/2`, el smoke termina con `exit 0` (bueno) o `≠ 0` (malo). Una máquina (CI) puede decidir con esa señal.

---

## B7 — Release y rollback

```bash
# 1. CI verde en main
# 2. Tag del release
git tag -a v0.3.0 -m "Release v0.3.0: /health con checks"
git push origin v0.3.0
# 3. Render despliega el commit (auto-deploy de main o "Manual Deploy")
# 4-5. Health check (Render) + smoke (tú)
bash scripts/smoke_test.sh https://dsia-e2e.onrender.com
```

Si el smoke falla:

| Opción | Cómo | Cuándo |
| --- | --- | --- |
| **Rollback en Render** | *Events* → deploy anterior que funcionaba → **Rollback** | Lo primero: recuperar el servicio en minutos |
| Revertir en git | `git revert <commit>` + push → nuevo deploy | Después, para que `main` vuelva a estar sano |
| Desplegar un commit concreto | *Manual Deploy* → elegir el commit del tag anterior | Si el historial de deploys no tiene la versión |

> Tras un rollback en Render, revisa el auto-deploy: el siguiente push a `main` vuelve a desplegar. Arregla `main` (revert o fix) antes de seguir subiendo cambios.

Anota en el README qué versión está desplegada (`v0.3.0`) y la URL.

---

## B8 — Propiedades (chuleta de diseño)

| Propiedad | Pregunta | En el proyecto |
| --- | --- | --- |
| Reproducibilidad | ¿Un clon limpio construye lo mismo? | `requirements.txt` acotado + `.python-version` + build escrito |
| Paridad local/prod | ¿El mismo código corre en los dos sitios? | `gunicorn` + `$PORT` en local (B2) |
| Configuración externa | ¿Cambio de entorno sin cambiar código? | Variables de entorno + `.env.example` |
| Seguridad | ¿Hay secretos en repo, YAML o logs? | Secretos solo en Render; `sync: false` |
| Verificabilidad | ¿Puedo demostrar que funciona? | Health check + smoke contra la URL |
| Observabilidad | ¿Sé qué pasó sin entrar a la máquina? | Logs a `stdout` con `request_id` |
| Reversibilidad | ¿Vuelvo atrás en minutos? | Tags + rollback en Render |
| Infra como código | ¿La configuración está versionada? | `render.yaml` |

---

## B9 — Puente al Trabajo Final (Proyecto III)

```text
E2E I   →  contratos + capas + mock por defecto
E2E II  →  errores tipados + logs con request_id + health accionable
E2E III →  empaquetado + variables de entorno + Render + smoke + rollback
Siguiente → IA asistida + release candidate (16 nov) y presentación (30 nov): la URL pública es parte de la demo
```

El Proyecto III exige **despliegue accesible o empaquetado listo con instrucciones verificables** (API y dashboard). Con Render cubres la primera opción; si no puedes desplegar, entrega el ensayo de B2 + `render.yaml` + smoke en local como “listo para desplegar”.

Sección mínima del README:

```markdown
## Despliegue

URL: https://dsia-<proyecto>.onrender.com  ·  Versión: v0.3.0

1. Build:  pip install -r requirements.txt && python ingest.py --input data/raw/ventas.csv --output data/clean.csv
2. Variables de entorno: ver `.env.example` (AI_PROVIDER=mock por defecto; AI_API_KEY solo si real)
3. Start:  gunicorn app:app --bind 0.0.0.0:$PORT
4. Smoke:  bash scripts/smoke_test.sh <URL>
5. Rollback: Render → Events → deploy anterior → Rollback (o redeploy del tag anterior)
```

> Si tu Trabajo Final tiene **dashboard** (Streamlit, Dash…), es un segundo *Web Service* con su propio start command (p. ej. `streamlit run dashboard.py --server.port $PORT --server.address 0.0.0.0`) y la URL de la API como variable de entorno.

---

## B10 — Anti-patrones

1. `app.run(debug=True)` como comando de producción.  
2. Puerto fijo (`8000`) o `127.0.0.1` en el servidor: “Deploy OK” pero nadie llega.  
3. `requirements.txt` generado a mano con lo que “crees” que usas (o un `pip freeze` de 200 paquetes del venv del curso).  
4. Claves de API en el código, en `render.yaml`, en el README o en el historial de git.  
5. Código con `if ENV == "prod": …` que cambia el comportamiento en producción.  
6. Datos generados a mano en tu portátil y subidos al repo, en lugar de generarse en el build.  
7. Guardar estado en el disco del servidor (se pierde en cada redeploy).  
8. `/health` que llama a la IA o devuelve `503` por un `degraded` leve: reinicios en bucle.  
9. “Está desplegado” sin smoke contra la URL pública.  
10. No saber qué versión está en producción ni cómo volver a la anterior.

---

## B11 — Autoevaluación

### Clave

1. ¿Qué ejecuta Render en el build y en el start, y qué pasa si el build termina con `exit ≠ 0`?  
2. ¿Por qué el start command usa `$PORT` y `0.0.0.0`?  
3. ¿Qué va en `.env.example` y qué nunca va en el repo?  
4. ¿Cuáles son los pasos de un release y qué demuestra cada uno?

### Suplementario

5. ¿Qué diferencia hay entre el health check de Render y el smoke test?  
6. ¿Por qué `data/clean.csv` se genera en el build y no en runtime?  
7. ¿Qué ocurre si `/health` devuelve `503` cuando el servicio está `degraded`?  
8. ¿Cómo haces rollback si la última versión rompe `/analyze`, y qué haces después con `main`?

---

## B12 — Checklist de salida

**Parte A**

- [ ] `requirements.txt` acotado, `.python-version`, build y start escritos  
- [ ] `.env.example` completo; `.env` ignorado; sin secretos en el repo  
- [ ] Sé los pasos del release y cómo se hace rollback  

**Parte B**

- [ ] Ensayo local con `gunicorn` + `$PORT` + smoke `OK`  
- [ ] Servicio en Render en estado **Live** con health check en `/health`  
- [ ] Smoke `OK` contra la URL pública  
- [ ] Encuentro una petición en *Logs* por su `request_id`  
- [ ] README con sección “Despliegue” (URL, versión, pasos, rollback)  
- [ ] E9 hecho / deploy del Trabajo Final iniciado  

---

## B13 — Apéndices

### Chuleta de despliegue

```bash
# Ensayo de producción en local
pip install -r requirements.txt
python ingest.py --input data/raw/ventas.csv --output data/clean.csv && echo "exit=$?"
PORT=10000 gunicorn app:app --bind 0.0.0.0:10000 --workers 2
bash scripts/smoke_test.sh http://127.0.0.1:10000

# Render (dashboard)
#   Build:  pip install -r requirements.txt && python ingest.py --input data/raw/ventas.csv --output data/clean.csv
#   Start:  gunicorn app:app --bind 0.0.0.0:$PORT
#   Health: /health
#   Env:    AI_PROVIDER=mock · LOG_LEVEL=INFO · (AI_API_KEY solo si real)

# Release
git tag -a v0.3.0 -m "Release v0.3.0" && git push origin v0.3.0
bash scripts/smoke_test.sh https://dsia-<proyecto>.onrender.com

# Comprobar que no hay secretos en el repo
git grep -nI -E "(api[_-]?key|token|secret)\s*=\s*['\"]?[A-Za-z0-9_-]{12,}"
```

### Glosario

| EN / símbolo | Nota |
| --- | --- |
| deploy | poner una versión del código a funcionar en un entorno |
| PaaS | *Platform as a Service*: plataforma que construye y ejecuta tu app (Render, Fly.io, Heroku…) |
| Web Service (Render) | servicio con URL pública que escucha en `$PORT` |
| build / start command | comandos de construcción / arranque del servicio |
| WSGI / `gunicorn` | interfaz y servidor de producción para apps Python (Flask) |
| worker | proceso que atiende peticiones; varios en paralelo |
| `$PORT` | puerto que asigna la plataforma |
| environment variable | configuración fuera del código |
| secret | variable sensible (clave de API, token) |
| Blueprint / `render.yaml` | infraestructura de Render descrita como código |
| health check | comprobación automática de que el servicio responde |
| smoke test | prueba mínima del endpoint crítico tras el deploy |
| cold start | retraso de la primera petición al despertar un servicio dormido |
| ephemeral filesystem | disco que se borra al reiniciar o redesplegar |
| rollback | volver a una versión anterior desplegada |
| semver / tag | versión `vMAJOR.MINOR.PATCH` marcada en git |

### FAQ

**¿Tengo que usar Render?** No es obligatorio; es la plataforma de referencia del curso porque tiene plan gratuito, despliegue desde GitHub y logs en el dashboard. Fly.io, Cloud Run o Hugging Face Spaces valen si entregas la misma evidencia (URL, pasos, smoke, rollback).  
**¿Necesito tarjeta de crédito?** Para un *Web Service* en el plan gratuito, normalmente no. Si Render te la pide, entrega el ensayo de B2 + `render.yaml` como “listo para desplegar”.  
**¿Y el `Procfile` que pide E9?** Render no lo usa, pero sirve como documentación del start command para otros PaaS. La fuente de verdad en Render es el start command del dashboard o de `render.yaml`.  
**¿Despliego con el proveedor de IA real?** Opcional. Por defecto `mock`; si usas uno real, la clave va como secreto en Render y el servicio debe seguir respondiendo (`503` limpio) si el proveedor falla.  
**¿Por qué la primera petición tarda tanto?** *Cold start* del plan gratuito: el servicio se duerme sin tráfico. El smoke tiene `--max-time 90` por eso.  
**¿Dónde están mis logs de `run.log`?** En ningún sitio: en Render los logs van a `stdout` y se leen en el dashboard.  
**¿Puedo usar Docker?** Sí (Render acepta un `Dockerfile`), pero no es necesario en DSIA: el runtime de Python nativo basta.

### Rúbrica

| Criterio | Insuficiente | Adecuado | Sólido |
| --- | --- | --- | --- |
| Empaquetado | Solo funciona en tu venv | `requirements.txt` + build/start documentados | + versiones acotadas, `.python-version`, ensayo con `gunicorn` |
| Configuración | Claves o rutas en el código | Variables de entorno + `.env.example` | + falla al arrancar si falta un secreto obligatorio; `mock` por defecto |
| Despliegue | No hay URL ni instrucciones | URL en Render **o** dry-run reproducible | + `render.yaml` versionado y health check configurado |
| Verificación | “Funciona, créeme” | Smoke en local | Smoke contra la URL pública (manual o en CI) |
| Operación | Sin logs | Logs visibles en Render | + petición localizada por `request_id` |
| Release | Lo último de `main` | Tags semver | + rollback ensayado y documentado |

---

## Cierre

> **Desplegar = empaquetado reproducible + configuración y secretos fuera del código + release verificable (health + smoke) y reversible (rollback), ejecutado en Render sin cambiar una línea entre local y producción.**

**Ahora:** [`ejercicios/E9_despliegue.md`](ejercicios/E9_despliegue.md).
