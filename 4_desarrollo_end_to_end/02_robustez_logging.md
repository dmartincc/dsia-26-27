# Sesión 6 nov 2026 — E2E II: robustez, errores y logging
 
| Recurso | Fichero |
| --- | --- |
| Demo | [`plantilla_proyecto/`](plantilla_proyecto/) (`app.py`, cliente IA, `tests/`) |
| Ejercicio de clase | [`ejercicios/E8b_robustez.md`](ejercicios/E8b_robustez.md) |
| Datos | [`../3_automatizacion_e_ia/Datos/ventas.csv`](../3_automatizacion_e_ia/Datos/ventas.csv) → `data/clean.csv` |
| Base previa | [`01_arquitectura_e2e.md`](01_arquitectura_e2e.md) (contratos, capas, mock por defecto) |
| Enunciado | [`../proyectos/proyecto_iii/`](../proyectos/proyecto_iii/) |
 
## Historia de la sesión (léela primero)
 
Hoy no aprendes “cómo poner un `try/except`”. Aprendes a que tu flujo **falle bien**: que cada fallo tenga nombre, que el cliente reciba una respuesta limpia y que tú puedas reconstruir qué pasó **solo con el log**:
 
```text
1. Clasificas el fallo (taxonomía)              →  Parte A1  (400 · 502/503 · 504 · 500)
2. Lo registras con contexto y sin secretos     →  Parte A2  (request_id · latencia · proveedor)
3. El servicio dice cómo está (health)          →  Parte A3  (ok / degraded, accionable)
4. Lo traduces a respuestas HTTP coherentes     →  Parte B   (excepciones propias + handler central)
5. Lo provocas y lo testeas                     →  Parte B   (timeouts, reintentos, tests de fallo)
```
 
**Regla de oro:** si ante un fallo el cliente recibe una traza, o tú no sabes qué pasó sin reproducirlo, **aún no es robusto**.
 
En clase: exposición (~30 min, Parte A) → ejercicio [`E8b`](ejercicios/E8b_robustez.md) (~50 min, A + B).
 
---
 
## 0. Objetivos
 
### Obligatorios (Parte A)
 
1. Aplicar la **taxonomía de errores**: tipo de fallo → código HTTP → nivel de log.
2. Dejar **logs útiles** en cada petición: `request_id`, latencia, proveedor y resultado, sin datos sensibles.
3. Hacer que `/health` diga algo **accionable** (`ok` / `degraded`) y conocer la lista de “no hacer”.
### Deseables (Parte B)
 
4. Centralizar el manejo de errores con **excepciones propias** y un handler único.
5. Poner **timeouts** y **reintentos** solo donde tienen sentido.
6. **Provocar** cada tipo de fallo en local con el mock.
7. Escribir **tests de robustez** (códigos, cuerpo sin traza, logs sin secretos).
8. Conectar el patrón al **Trabajo Final (Proyecto III)**.
---
 
## 1. Por qué importa en DSIA
 
```text
Tema 2  →  pytest + CI              (confianza en el código)
Tema 3  →  job CLI, exit codes      (señales para máquinas)
E2E I   →  flujo completo + contratos
E2E II  →  robustez + errores + logging (hoy)   (el flujo falla bien)
Deploy  →  lo que hoy ves en local, mañana lo lees en un servidor sin pantalla (03_despliegue.md)
```
 
En la sesión 5, las señales eran `exit 0/1/2` + `metrics.json` + `run.log`. En un servicio, las señales equivalentes son **código HTTP + cuerpo de error + log estructurado + `/health`**:
 
| Job (sesión 5) | Servicio (hoy) |
| --- | --- |
| exit `0` | `200` |
| exit `2` (input roto) | `400` / `422` |
| exit `1` (gate de calidad) | `503` / `degraded` (según el caso) |
| `run.log` | log por petición con `request_id` |
| `metrics.json` | latencias, contadores, estado en `/health` |
 
| Idea | Analogía |
| --- | --- |
| Taxonomía de errores | Etiqueta de triaje en urgencias: cada caso, su protocolo |
| `request_id` | Número de albarán: sigue el pedido por toda la fábrica |
| Log útil | Libro de incidencias con hora, causa y responsable |
| `degraded` | Piloto ámbar: sigue funcionando, pero revisa algo |
| Timeout | No esperar eternamente a un proveedor que no contesta |
| Reintento | Volver a llamar solo si el fallo fue pasajero |
| Stacktrace al cliente | Dejar la factura de la fábrica a la vista de la calle |
 
---
 
# PARTE A — Tres conceptos clave
 
> Pregunta guía: ***¿qué pasa cuando algo falla y cómo lo sé?***  
> Orden: **clasificar → registrar → señalizar**. Cómo se implementa viene en la Parte B.
 
---
 
## A1 — Taxonomía de errores
 
Cada fallo tiene **una causa**, **un código** y **un nivel de log**. Decidirlo de antemano evita improvisar en producción.
 
| Tipo | HTTP típico | Log | ¿De quién es la culpa? |
| --- | --- | --- | --- |
| Input inválido | `400` | `WARNING` | Del cliente |
| Dependencia caída | `502` / `503` | `ERROR` | Del proveedor / infraestructura |
| Timeout | `504` | `ERROR` | Del proveedor (o de tu timeout) |
| Bug propio | `500` | `ERROR` + `request_id` | Tuya |
 
Cómo elegir:
 
```text
¿Lo puede arreglar el cliente cambiando la petición?   → 4xx   (WARNING)
¿Falló algo externo que no controlo?                   → 502/503/504 (ERROR)
¿No debería haber pasado nunca?                        → 500   (ERROR + traza en log)
```
 
Matices:
 
| Caso | Código | Nota |
| --- | --- | --- |
| Falta `text` o está vacío | `400` | Si el framework usa `422` para validación de schema, **fija uno** y úsalo siempre |
| Proveedor devuelve respuesta inválida | `502` | Respuesta mala de un tercero |
| Proveedor no disponible / cuota agotada | `503` | Puede incluir `Retry-After` |
| Proveedor no responde a tiempo | `504` | Hay timeout configurado |
| Excepción no prevista | `500` | Mensaje genérico al cliente, detalle en el log |
 
Regla: **WARNING no es ERROR**. Un `400` es normal (alguien se equivocó); si alertas por cada `400`, ignorarás las alertas reales.
 
**Checklist:** clasifico un fallo en menos de 10 segundos · sé qué se loguea en cada tipo · distingo `502`, `503` y `504`.
 
---
 
## A2 — Logging útil
 
Un log útil responde a: **¿qué petición?, ¿cuándo?, ¿cuánto tardó?, ¿con qué proveedor?, ¿cómo acabó?**
 
```python
import logging, time, uuid
logger = logging.getLogger("dsia.e2e")
 
request_id = str(uuid.uuid4())
t0 = time.perf_counter()
# ... llamada IA ...
logger.info(
    "analyze_ok request_id=%s latency_ms=%.1f provider=%s",
    request_id,
    (time.perf_counter() - t0) * 1000,
    "mock",
)
```
 
Campos mínimos:
 
| Campo | Para qué |
| --- | --- |
| `request_id` | Unir todas las líneas de una misma petición (y devolverlo al cliente en errores) |
| evento (`analyze_ok`, `analyze_failed`) | Buscar y contar por tipo de suceso |
| `latency_ms` | Detectar lentitud antes de que sea caída |
| `provider` | Saber qué proveedor se usó (mock o real) |
| causa del fallo (`error_type`) | Diagnosticar sin reproducir |
 
Niveles:
 
| Nivel | Cuándo |
| --- | --- |
| `INFO` | Petición correcta |
| `WARNING` | Error del cliente (`400`), reintento realizado |
| `ERROR` | Dependencia caída, timeout, bug (`5xx`) |
| `DEBUG` | Detalle de desarrollo (apagado fuera de local) |
 
Qué **nunca** va al log:
 
- API keys, tokens, cabeceras `Authorization`.
- Texto completo del usuario si puede contener datos personales (loguea `n_chars` o un hash corto).
- Prompts completos con datos reales.
**Checklist:** cada petición deja ≥1 línea con `request_id` · sé reconstruir una petición fallida solo con el log · no hay secretos en el log.
 
---
 
## A3 — Healthcheck accionable + qué no hacer
 
`/health` no es solo “estoy vivo”: debe decir **algo que alguien pueda usar**.
 
```json
{
  "status": "degraded",
  "checks": {
    "data": "missing: data/clean.csv",
    "provider": "mock"
  }
}
```
 
| Estado | Significa | Acción |
| --- | --- | --- |
| `ok` | Todo lo necesario está disponible | Ninguna |
| `degraded` | Responde, pero falta algo (path de datos inexistente, proveedor en fallo reciente) | Revisar `checks` |
| (sin respuesta) | El proceso no está vivo | Reiniciar / mirar log |
 
Reglas:
 
- `degraded` si el **path de datos no existe** o el **proveedor está en fallo**.
- El check de proveedor **no llama a la IA** en cada petición a `/health`: usa el **resultado de la última llamada real** (o un flag de estado) para que `/health` siga siendo barato.
- El check de datos es local (`Path.exists()`), no lee el fichero entero.
- Dile al cliente **qué** falla, no por qué por dentro: sin rutas absolutas ni trazas.
> Nota respecto a E2E I: allí decíamos que `/health` no debe depender de la IA ni de los datos. Se mantiene: **no hace llamadas externas ni lecturas costosas**. Lo que añadimos hoy es que refleje **estado ya conocido** (¿existe el fichero?, ¿falló el último intento?).
 
### No hacer
 
- `except Exception: pass` (el fallo desaparece y nadie se entera).
- Devolver stacktraces al cliente.
- Loguear API keys o prompts con datos personales.
**Checklist:** `/health` devuelve `ok`/`degraded` con `checks` · lo fuerzo a `degraded` borrando `data/clean.csv` · sé por qué `/health` no llama a la IA.
 
---
 
## A4 — Mapa de los 3 conceptos
 
```text
A1 taxonomía    →  sé qué tipo de fallo es y qué código/nivel le toca
A2 logging      →  puedo reconstruir qué pasó sin estar delante
A3 health       →  el servicio dice cómo está, de forma accionable
        │
        └── excepciones, timeouts, reintentos y tests (Parte B) lo implementan y lo verifican
```
 
**Siguiente paso en clase:** [`ejercicios/E8b_robustez.md`](ejercicios/E8b_robustez.md).
 
---
 
# PARTE B — Implementar y verificar la robustez
 
> La Parte A dice *qué* debe cumplir el servicio ante fallos.  
> La Parte B enseña *cómo se implementa* (excepciones → handler → timeouts → logs) y *cómo se prueba* (provocar fallos + `pytest`).
 
---
 
## B1 — Anatomía: qué se añade a la plantilla
 
| Pieza | Rol | Concepto |
| --- | --- | --- |
| Excepciones propias (`errors.py`) | `InvalidInput`, `UpstreamError`, `UpstreamTimeout` | Taxonomía en código |
| Handler central | Traduce excepción → código HTTP + cuerpo limpio | Una sola puerta de salida |
| `request_id` | Se genera al entrar y viaja por log y respuesta de error | Trazabilidad |
| Configuración de logging | Formato, nivel, destino | Observabilidad |
| Cliente IA con timeout | Límite de espera + reintento acotado | Dependencia controlada |
| `/health` extendido | `ok` / `degraded` + `checks` | Señal accionable |
| `tests/test_robustez.py` | Fallos provocados con el mock | Confianza |
 
> Si tu plantilla usa otros nombres, mantén la **separación**: errores tipados, un único punto de traducción y logs con `request_id`.
 
---
 
## B2 — Demo en vivo: provocar cada tipo de fallo
 
Idea: el **mock** puede simular fallos (por parámetro o variable de entorno), así se enseña y se testea cada rama **sin red**.
 
```bash
cd 4_desarrollo_end_to_end/plantilla_proyecto
python ingest.py --input ../../3_automatizacion_e_ia/Datos/ventas.csv --output data/clean.csv
python app.py
```
 
```bash
# 1. Input inválido → 400, WARNING
curl -s -i -X POST http://127.0.0.1:8000/analyze \
  -H 'Content-Type: application/json' -d '{"text":""}'
 
# 2. Éxito → 200, INFO con latency_ms y provider=mock
curl -s -X POST http://127.0.0.1:8000/analyze \
  -H 'Content-Type: application/json' \
  -d '{"text":"Texto de ejemplo suficientemente largo para el resumen mock del curso DSIA"}'
 
# 3. Dependencia caída / timeout → 503 / 504, ERROR
#    (el mock se configura para fallar; el nombre de la variable es orientativo)
AI_MOCK_MODE=down    python app.py   # → 503
AI_MOCK_MODE=timeout python app.py   # → 504
 
# 4. Salud con datos ausentes → degraded
mv data/clean.csv data/clean.csv.bak
curl -s http://127.0.0.1:8000/health
mv data/clean.csv.bak data/clean.csv
```
 
| Qué observar | Por qué |
| --- | --- |
| El cuerpo del error **no** contiene traza ni rutas | Seguridad y claridad |
| El cuerpo incluye `request_id` | El cliente puede citarlo al reportar |
| El log del mismo `request_id` tiene la causa real | Diagnóstico sin reproducir |
| `/health` pasa a `degraded` al faltar el fichero | Señal accionable |
 
---
 
## B3 — Excepciones propias + handler central
 
Un solo sitio decide qué código y qué mensaje ve el cliente.
 
```python
# errors.py
class AppError(Exception):
    status_code = 500
    code = "internal_error"
    public_message = "Error interno"
 
class InvalidInput(AppError):
    status_code, code = 400, "invalid_input"
    def __init__(self, detail: str):
        self.public_message = detail          # seguro de mostrar
 
class UpstreamError(AppError):
    status_code, code = 502, "upstream_error"
    public_message = "El proveedor de IA devolvió un error"
 
class UpstreamUnavailable(AppError):
    status_code, code = 503, "upstream_unavailable"
    public_message = "El proveedor de IA no está disponible"
 
class UpstreamTimeout(AppError):
    status_code, code = 504, "upstream_timeout"
    public_message = "El proveedor de IA no respondió a tiempo"
```
 
```python
# handler (adapta a tu framework)
def to_response(exc: Exception, request_id: str):
    if isinstance(exc, AppError):
        level = logger.warning if exc.status_code < 500 else logger.error
        level("analyze_failed request_id=%s code=%s status=%d cause=%r",
              request_id, exc.code, exc.status_code, exc)
        body = {"error": {"code": exc.code,
                          "message": exc.public_message,
                          "request_id": request_id}}
        return exc.status_code, body
 
    # Cualquier otra cosa = bug propio
    logger.exception("analyze_bug request_id=%s", request_id)   # traza SOLO en el log
    return 500, {"error": {"code": "internal_error",
                           "message": "Error interno",
                           "request_id": request_id}}
```
 
Reglas:
 
- El **cliente** ve `code`, `message` genérico y `request_id`. Nunca la traza.
- El **log** ve la causa real (`cause=…`) y, en `500`, la traza completa (`logger.exception`).
- Forma de error **siempre igual** → los clientes pueden depender de ella (contrato de A1 en E2E I).
- Traducción de excepciones del proveedor a las tuyas **en el borde** (cliente IA), no en la API.

> En `E8b` basta con un cuerpo plano `{"error": "...", "request_id": "..."}`. La forma anidada de arriba es la recomendada para el Trabajo Final; elige una y mantenla en todos los endpoints.
---
 
## B4 — Timeouts y reintentos
 
Una dependencia externa puede **tardar**, **fallar a ratos** o **caerse**. Sin timeout, tu servicio se cuelga con ella.
 
| Mecanismo | Regla |
| --- | --- |
| **Timeout** | Siempre en llamadas externas (p. ej. 10 s); al vencer → `UpstreamTimeout` (`504`) |
| **Reintento** | Solo en fallos **transitorios** (timeout, `503`), nunca en `4xx` |
| **Backoff** | Esperar más entre intentos (p. ej. 0,5 s → 1 s → 2 s) |
| **Límite** | Pocos intentos (2–3). Si no, convertirías una caída en una avalancha |
| **Idempotencia** | Reintentar solo operaciones que no dupliquen efectos (analizar texto ✔; cobrar ✘) |
 
```python
def call_with_retry(fn, attempts=3, base_delay=0.5):
    for i in range(attempts):
        try:
            return fn()
        except (UpstreamTimeout, UpstreamUnavailable) as e:
            if i == attempts - 1:
                raise
            logger.warning("retry attempt=%d/%d cause=%r", i + 1, attempts, e)
            time.sleep(base_delay * 2 ** i)
```
 
Conexión con la sesión 5: igual que cron **re-lanza** el job y exigía idempotencia, aquí el reintento **repite** una llamada y exige que sea segura de repetir.
 
---
 
## B5 — Configuración del logging
 
```python
import logging, sys
 
def setup_logging(level: str = "INFO") -> None:
    logging.basicConfig(
        level=level,
        format="%(asctime)s %(levelname)s %(name)s %(message)s",
        handlers=[logging.StreamHandler(sys.stdout)],   # + FileHandler si hace falta
    )
```
 
Buenas prácticas:
 
| Práctica | Motivo |
| --- | --- |
| `logging`, no `print` | Niveles, formato, destino configurables |
| Un logger por módulo (`dsia.e2e`, `dsia.e2e.ai`) | Filtrar por origen |
| Nivel por entorno (`LOG_LEVEL`) | `DEBUG` en local, `INFO` en CI/prod |
| `key=value` en el mensaje | Fácil de buscar con `grep` |
| `request_id` en **todas** las líneas de la petición | Reconstruir la historia completa |
| No loguear el cuerpo de la petición completo | Datos personales |
 
Ejemplo de reconstrucción:
 
```bash
grep "request_id=3f2a" run.log
# 12:01:03 INFO    analyze_start request_id=3f2a provider=mock n_chars=82
# 12:01:13 WARNING retry attempt=1/3 cause=UpstreamTimeout()
# 12:01:23 ERROR   analyze_failed request_id=3f2a code=upstream_timeout status=504
```
 
---
 
## B6 — `/health` extendido
 
```python
def health():
    checks = {
        "data": "ok" if DATA_PATH.exists() else f"missing: {DATA_PATH.name}",
        "provider": LAST_PROVIDER_STATUS,        # "ok" | "failing: upstream_timeout" | "mock"
    }
    degraded = any(v.startswith(("missing", "failing")) for v in checks.values())
    return {"status": "degraded" if degraded else "ok", "checks": checks}
```
 
Decisión de diseño a fijar en el contrato:
 
| Opción | Código HTTP de `degraded` | Cuándo |
| --- | --- | --- |
| A | `200` + `"status": "degraded"` | El servicio sigue sirviendo; solo avisa |
| B | `503` | Un balanceador debe sacarlo de rotación |
 
En el curso: **opción A** por defecto; documenta la tuya en el README.  
En `E8b` se pide la **opción B** (`503`) para practicarla. Ojo al desplegar: si la plataforma usa `/health` como health check, un `503` hace que **no** reciba tráfico o que se reinicie (ver `03_despliegue.md`, A3).
 
---
 
## B7 — Tests de robustez
 
Qué se testea (todo con el mock, **sin red**):
 
| Caso | Aserción |
| --- | --- |
| Input vacío / ausente | `400`, cuerpo con `error.code == "invalid_input"` |
| Proveedor caído | `503`, sin traza en el cuerpo |
| Timeout | `504` |
| Bug interno forzado | `500`, mensaje genérico, `request_id` presente |
| `/health` sin datos | `status == "degraded"` |
| Logs | Contienen `request_id`; **no** contienen la clave ni el texto del usuario |
 
```python
import pytest
 
@pytest.mark.parametrize("mode,status", [("down", 503), ("timeout", 504)])
def test_upstream_failures(client, monkeypatch, mode, status):
    monkeypatch.setenv("AI_MOCK_MODE", mode)
    r = client.post("/analyze", json={"text": "Texto suficientemente largo para el mock del curso"})
    assert r.status_code == status
    body = r.json()["error"]
    assert "request_id" in body
    assert "Traceback" not in r.text
 
def test_logs_do_not_leak_text(client, caplog):
    secret = "dato-personal-123"
    client.post("/analyze", json={"text": f"{secret} texto suficientemente largo para el mock"})
    assert secret not in caplog.text
 
def test_health_degraded_without_data(client, tmp_path, monkeypatch):
    monkeypatch.setattr("app.DATA_PATH", tmp_path / "no_existe.csv")
    assert client.get("/health").json()["status"] == "degraded"
```
 
> Un test de robustez es un test de **lo que pasa cuando algo sale mal**. Si solo pruebas el camino feliz, no sabes si el servicio falla bien.
 
---
 
## B8 — Propiedades (chuleta de diseño)
 
| Propiedad | Pregunta | En la plantilla |
| --- | --- | --- |
| Taxonomía | ¿Cada fallo tiene tipo, código y nivel? | Excepciones propias + tabla A1 |
| Respuesta limpia | ¿El cliente nunca ve la traza? | Handler central |
| Trazabilidad | ¿Puedo seguir una petición de punta a punta? | `request_id` en log y error |
| Observabilidad | ¿Sé qué pasó sin estar delante? | Logs `key=value` + `/health` |
| Resiliencia | ¿Aguanta una dependencia lenta o caída? | Timeout + reintento acotado |
| Seguridad | ¿Hay secretos o datos personales en logs/respuestas? | Redacción + test de fugas |
| Honestidad | ¿Dice la verdad sobre su estado? | `degraded` accionable |
 
---
 
## B9 — Puente al Trabajo Final (Proyecto III)
 
```text
E2E I   →  contratos + capas + mock por defecto
E2E II  →  errores tipados + logs con request_id + health accionable + tests de fallo
Deploy  →  mismo servicio, en Render: lo único que verás de él es el log y /health
```

En [`03_despliegue.md`](03_despliegue.md) el `/health` de hoy se convierte en el **health check** de la plataforma y los logs a `stdout` son lo que leerás en su pestaña *Logs*.
 
Reutiliza: la taxonomía y el handler central en cualquier endpoint nuevo · `request_id` en todos los logs · mock con modos de fallo en CI.  
Para la evaluación, el proyecto debería poder **demostrar un fallo controlado** (p. ej. `AI_MOCK_MODE=down` → `503` limpio + línea en el log).
 
---
 
## B10 — Anti-patrones
 
1. `except Exception: pass` o `except: ...` que se traga el error.  
2. Devolver el stacktrace (o rutas internas) al cliente.  
3. Loguear API keys, tokens o prompts con datos personales.  
4. Devolver `500` para entradas inválidas (o `200` con `{"error": ...}`).  
5. Todo en `ERROR` (o todo en `INFO`): los niveles pierden sentido.  
6. Logs sin `request_id` ni contexto: “falló algo”.  
7. Llamadas externas sin timeout.  
8. Reintentar a ciegas (en `4xx`, sin límite o sin backoff).  
9. `/health` que siempre dice `ok`, o que llama a la IA en cada petición.  
10. Tests solo del camino feliz.
---
 
## B11 — Autoevaluación
 
### Clave
 
1. ¿Qué código HTTP y qué nivel de log corresponde a cada tipo de fallo de la taxonomía?  
2. ¿Qué campos mínimos debe tener una línea de log de petición y por qué?  
3. ¿Cuándo debe `/health` devolver `degraded`?  
4. ¿Qué tres cosas no se hacen nunca (lista “No hacer”)?
### Suplementario
 
5. ¿Diferencia entre `502`, `503` y `504`?  
6. ¿Por qué el cliente recibe `request_id` pero no la traza?  
7. ¿Cuándo tiene sentido reintentar y cuándo no?  
8. ¿Cómo compruebas con un test que un log no filtra datos del usuario?
---
 
## B12 — Checklist de salida
 
**Parte A**
 
- [ ] Clasifico un fallo: tipo → código → nivel  
- [ ] Cada petición deja log con `request_id`, latencia y proveedor  
- [ ] `/health` devuelve `ok`/`degraded` con `checks`  
**Parte B**
 
- [ ] Excepciones propias + handler central  
- [ ] Timeout en la llamada IA; reintento solo en transitorios  
- [ ] Provoco `400`, `503`, `504` y `500` en local  
- [ ] Tests de robustez pasan sin red; sin traza ni secretos en respuestas/logs  
- [ ] E8b hecho / idea clara para el Trabajo Final  
---
 
## B13 — Apéndices
 
### Chuleta de robustez
 
```bash
cd 4_desarrollo_end_to_end/plantilla_proyecto
python ingest.py --input ../../3_automatizacion_e_ia/Datos/ventas.csv --output data/clean.csv
 
# Levantar y probar fallos
python app.py
curl -s -i -X POST http://127.0.0.1:8000/analyze -H 'Content-Type: application/json' -d '{"text":""}'   # 400
AI_MOCK_MODE=down    python app.py    # → /analyze devuelve 503
AI_MOCK_MODE=timeout python app.py    # → /analyze devuelve 504
curl -s http://127.0.0.1:8000/health                                                                   # ok | degraded
 
# Reconstruir una petición
grep "request_id=<id>" run.log
 
# Tests
pytest -q
```
 
### Glosario
 
| EN / símbolo | Nota |
| --- | --- |
| robustness | capacidad de seguir funcionando (o fallar bien) ante errores |
| error taxonomy | clasificación de fallos: tipo → código → nivel |
| `request_id` | identificador único de una petición, presente en log y errores |
| latency | tiempo de respuesta (ms) |
| timeout | límite de espera de una llamada externa |
| retry / backoff | reintento / espera creciente entre intentos |
| upstream | dependencia externa (proveedor de IA) |
| degraded | funciona, pero con algo en fallo o ausente |
| stacktrace | traza de la excepción; solo en logs, nunca al cliente |
| PII | datos personales: no van a logs ni respuestas de error |
 
### FAQ
 
**¿`400` o `422`?** Cualquiera, pero **uno** y siempre el mismo; documéntalo en el contrato.  
**¿Todo error 5xx se reintenta?** No: solo los transitorios (timeout, `503`). Un `500` por bug se arregla, no se reintenta.  
**¿Puedo loguear el texto que envía el usuario?** Mejor no: usa `n_chars` o un hash. Si necesitas depurar, solo en local y con datos de prueba.  
**¿Por qué `/health` no llama a la IA?** Porque se consulta a menudo y debe ser barato; refleja el estado de la **última** llamada real.  
**¿Necesito una librería de logging avanzada?** No: `logging` estándar con `key=value` basta en DSIA.
 
### Rúbrica
 
| Criterio | Insuficiente | Adecuado | Sólido |
| --- | --- | --- | --- |
| Taxonomía | Todo `500` o `200` con error | Códigos correctos por tipo | + handler central y forma de error uniforme |
| Logging | `print` / sin contexto | `logging` con niveles | + `request_id`, latencia y proveedor en cada petición |
| Seguridad | Traza o secretos expuestos | Sin traza al cliente | + test de que logs y errores no filtran datos |
| Resiliencia | Sin timeout | Timeout | + reintento acotado con backoff |
| Health | Siempre `ok` | `ok`/`degraded` | + `checks` accionables y documentados |
| Tests | Solo camino feliz | Casos de error | Fallos provocados con mock y parametrizados |
 
---
 
## Cierre
 
> **Robustez = taxonomía de errores + logs con contexto y sin secretos + health accionable, implementada con excepciones tipadas, timeouts acotados y tests de fallo.**
 
**Ahora:** [`ejercicios/E8b_robustez.md`](ejercicios/E8b_robustez.md).