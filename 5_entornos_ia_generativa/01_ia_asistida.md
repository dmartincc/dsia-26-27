# Sesión 16 nov 2026 — IA generativa aplicada al desarrollo

| Recurso | Fichero |
| --- | --- |
| Demo | [`../sesiones/2026-11-23_asistentes/ejemplos/modulo_deuda.py`](../sesiones/2026-11-23_asistentes/ejemplos/modulo_deuda.py) (módulo con deuda deliberada) |
| Prompts | [`prompts/catalogo_prompts.md`](prompts/catalogo_prompts.md) · [`../sesiones/2026-11-23_asistentes/ejemplos/prompts_clase.md`](../sesiones/2026-11-23_asistentes/ejemplos/prompts_clase.md) |
| Ejercicio de clase | [`ejercicios/E6_asistentes.md`](ejercicios/E6_asistentes.md) · [`ejercicios/E7_refactor_tests_docs.md`](ejercicios/E7_refactor_tests_docs.md) (release candidate) |
| Base previa | [`../4_desarrollo_end_to_end/`](../4_desarrollo_end_to_end/) (contratos, robustez, despliegue) · tests con `pytest` + CI |
| Herramientas | Cursor · Claude Code · Gemini CLI (cualquiera vale; el protocolo es el mismo) |
| Enunciado | [`../proyectos/proyecto_iii/`](../proyectos/proyecto_iii/) (criterio 8: política de uso de IA) |

## Historia de la sesión (léela primero)

Hoy no aprendes “qué asistente es mejor”. Aprendes a **trabajar con uno sobre código que ya existe** (tu Trabajo Final) sin perder el control: que cada sugerencia pase por tu revisión, por los tests y por el registro antes de llegar a `main`:

```text
1. Tú firmas el código: la IA sugiere          →  Parte A1  (responsabilidad · riesgos · AI_USAGE.md)
2. Le das contexto acotado y prompts en cadena  →  Parte A2  (explicar → plan → un cambio → tests → review)
3. Aceptas solo lo que supera el protocolo      →  Parte A3  (diff entendido · APIs reales · tests verdes · sin secretos)
4. Lo aplicas sobre un módulo con deuda         →  Parte B   (demo en vivo con modulo_deuda.py)
5. Lo llevas a tu proyecto y lo citas           →  Parte B   (agentes en el repo · review de seguridad · AI_USAGE.md)
```

**Regla de oro:** si no puedes explicar línea a línea un cambio generado por la IA, o no hay un test que lo respalde, **no se commitea**.

En clase: exposición (~30 min, Parte A + demo) → ejercicio [`E6`](ejercicios/E6_asistentes.md) (~30 min) → [`E7`](ejercicios/E7_refactor_tests_docs.md): release candidate del Trabajo Final con el mismo protocolo (resto de la sesión).

---

## 0. Objetivos

### Obligatorios (Parte A)

1. Asumir la **responsabilidad** sobre el código generado y conocer sus **riesgos** típicos (APIs inventadas, inseguridad, dependencias sorpresa).
2. Escribir prompts con **contexto acotado** y encadenarlos: explicar → planificar → implementar un paso → testear → revisar.
3. Aplicar un **protocolo de aceptación** antes de cada commit y **citar** el uso de IA en `AI_USAGE.md`.

### Deseables (Parte B)

4. Usar un asistente (Cursor, Claude Code o Gemini CLI) sobre **código existente**, no sobre un repo vacío.
5. Refactorizar en **pasos pequeños** con diffs revisables (< 80 líneas).
6. Generar tests con IA y **descartar** los que alucinen.
7. Hacer un **review de riesgos** asistido (secretos, validación, timeouts, errores) y triar hallazgos reales.
8. Conectar el protocolo con el **Trabajo Final (Proyecto III)** y su política de citas.

---

## 1. Por qué importa en DSIA

```text
Tema 1  →  Clean Code / SOLID         (qué es código mantenible)
Tema 2  →  pytest + CI                (el freno de mano)
E2E I–III →  contratos, robustez, deploy  (lo que no puedes romper)
Hoy     →  IA asistida                (acelerar sin romper nada de lo anterior)
Después →  release candidate y presentación del Trabajo Final
```

La IA hace barato **escribir** código; no hace barato **entenderlo, probarlo ni mantenerlo**. Todo lo construido hasta ahora (contratos, tests, CI, `/health`, logs) es lo que te permite **aceptar o rechazar** una sugerencia con evidencia, no con intuición.

| Sin protocolo | Con protocolo |
| --- | --- |
| “Hazme la API” → 400 líneas que no entiendes | Un paso del plan → diff de 30 líneas revisado |
| Import de una librería que no existe | `pytest` y `pip check` lo detectan antes del commit |
| Clave pegada en el prompt o en el código | Contexto sin secretos; `git grep` antes de commitear |
| “Lo hizo la IA” | `AI_USAGE.md`: qué se pidió, qué se aceptó, qué se rechazó y por qué |

| Idea | Analogía |
| --- | --- |
| Asistente de IA | Becario muy rápido y muy seguro de sí mismo, que a veces se inventa cosas |
| Contexto acotado | Darle el plano de la pieza, no las llaves de toda la fábrica |
| Prompt de explicación | “Cuéntame qué hace esta máquina antes de tocarla” |
| Prompt de plan | Presupuesto antes de la obra |
| Un paso cada vez | Cambiar una pieza y comprobar que la cadena sigue funcionando |
| Tests | Control de calidad: nada sale sin pasar por él |
| Alucinación | Una pieza de catálogo que no existe |
| `AI_USAGE.md` | Albarán: quién aportó qué en el producto final |

---

# PARTE A — Tres conceptos clave

> Pregunta guía: ***¿cómo uso la IA para ir más rápido sin perder el control de lo que entrego?***  
> Orden: **responsabilidad → contexto y prompts → aceptación**. La práctica con herramientas viene en la Parte B.

---

## A1 — Tú firmas: responsabilidad, riesgos y citación

La IA **sugiere**; tú **entregas**. A efectos académicos y de producto, el código del repo es tuyo, lo haya escrito quien lo haya escrito.

Flujo profesional:

```text
Contexto ──► Prompt ──► Borrador (IA) ──► Revisión humana ──► Tests ──► Commit (+ AI_USAGE.md)
                              ▲                    │
                              └──── rechazo / ajuste del prompt ◄──┘
```

Riesgos típicos del código generado:

| Riesgo | Cómo se manifiesta | Cómo se detecta |
| --- | --- | --- |
| **Alucinación** | Funciones, parámetros o librerías que no existen | Ejecutar: `pytest`, `python -c "import x"`, documentación oficial |
| **API obsoleta** | Sintaxis de una versión antigua (p. ej. pandas `append`) | Tests + avisos de deprecación |
| **Dependencia sorpresa** | `import requests_cache` que nadie pidió | Diff de `requirements.txt`; `pip check` |
| **Inseguridad** | Claves en el código, `debug=True`, sin timeout, `eval` | Review de riesgos (B5) + `git grep` |
| **Cambio de comportamiento** | “Refactor” que cambia el resultado | Tests **antes** y **después** del cambio |
| **Sobre-ingeniería** | Clases, patrones y capas que el problema no necesita | Diff grande para un problema pequeño |
| **Fuga de datos** | Pegar datos personales o secretos en un prompt en la nube | Regla: nunca, salvo permiso explícito y datos anonimizados |

Citación obligatoria (política del Proyecto III):

- Cualquier aportación **relevante** de una IA (código, tests, documentación, diseño) se registra en `AI_USAGE.md`.
- Se registra también lo **rechazado** y por qué: demuestra que revisaste.
- No hace falta citar autocompletados triviales (cerrar un paréntesis, un nombre de variable).

**Checklist:** sé enumerar 4 riesgos del código generado · sé cómo detectar cada uno · sé qué va en `AI_USAGE.md`.

---

## A2 — Contexto acotado y prompts en cadena

Un asistente responde según el **contexto** que recibe. Demasiado poco → inventa; demasiado → se pierde y toca lo que no debe.

| Contexto | Ejemplo | Resultado típico |
| --- | --- | --- |
| Insuficiente | “Mejora mi código” | Respuesta genérica, cambios arbitrarios |
| **Acotado** | Un módulo + su firma pública + restricciones | Cambio pequeño y revisable |
| Excesivo | Todo el repo “para que lo arregle” | Diff enorme, cambios no pedidos |

Un buen prompt para código incluye:

| Elemento | Ejemplo |
| --- | --- |
| Rol / perspectiva | “Eres un revisor senior de Python” |
| Objetivo concreto | “Extrae el cálculo de importes a una función pura” |
| Contexto | El fichero, la firma pública, los tests existentes |
| Restricciones | “Sin dependencias nuevas · mantén el comportamiento · no toques `app.py`” |
| Formato de salida | “Solo el diff y una lista de cambios” / “Plan en 5 pasos, sin código” |

Cadena de prompts (del catálogo):

```text
1. Explicar   →  “Explica qué hace este módulo, sus smells y riesgos. No reescribas código.”
2. Planificar →  “Propón un plan de refactor en ≤ 5 pasos, sin código.”
3. Un paso    →  “Implementa SOLO el paso 1. Diff mínimo. Sin dependencias nuevas.”
4. Tests      →  “Escribe tests pytest: caso feliz, error esperado y caso límite. Sin red.”
5. Review     →  “Revisa este diff buscando: secretos, validación, timeouts, errores, complejidad.”
```

Por qué en cadena:

| Decisión | Motivo |
| --- | --- |
| Explicar antes de cambiar | Detectas si la IA (y tú) entiende el código |
| Plan sin código | Revisas la **intención** antes de revisar 200 líneas |
| Un paso cada vez | Diffs pequeños = revisables y reversibles |
| Tests tras cada paso | El comportamiento observable no cambia sin que lo sepas |
| Review al final | Segunda pasada con otro objetivo (riesgos, no estilo) |

**Checklist:** mis prompts tienen objetivo, contexto y restricciones · nunca pido “reescríbelo todo” · sé qué prompt del catálogo toca en cada paso.

---

## A3 — Protocolo de aceptación

Antes de **cada** commit con código generado, en voz alta:

```text
[ ] ¿Lo entiendo línea a línea en el diff?          → si no, pide explicación o recházalo
[ ] ¿Los imports y APIs existen de verdad?           → ejecútalo; no te fíes de que “suena bien”
[ ] ¿Pasan los tests (nuevos y antiguos)?            → pytest -q en verde
[ ] ¿Hay dependencias sorpresa?                      → git diff requirements.txt
[ ] ¿Hay secretos o datos personales?                → git grep / revisión del diff
[ ] ¿El diff es mínimo para lo que se pidió?         → si toca más, separa o recorta
[ ] ¿Está registrado en AI_USAGE.md?                 → aceptado y rechazado
```

| Si falla… | Acción |
| --- | --- |
| No lo entiendo | Prompt de explicación sobre el diff; si sigue sin estar claro, **rechazar** |
| API inexistente / test que alucina | Borrar ese fragmento y registrarlo como rechazo |
| Tests rojos | No “arreglar el test” para que pase: revisar si cambió el comportamiento |
| Diff demasiado grande | `git restore` y pedir un paso más pequeño |
| Secretos | No commitear; si ya llegó a git, **rotar** la clave (ver E2E III, A2) |

Qué **no** hacer:

- Aceptar todo con “Accept all” sin leer el diff.
- Pedir a la IA que “haga pasar los tests” modificando los tests.
- Pegar secretos, `.env` o datos personales en el prompt.

**Checklist:** aplico el protocolo antes de cada commit · sé qué hacer cuando falla cada punto · registro también los rechazos.

---

## A4 — Mapa de los 3 conceptos

```text
A1 responsabilidad  →  el código es tuyo; conoces los riesgos y citas el uso
A2 contexto/prompts →  la IA trabaja sobre lo justo y en pasos pequeños
A3 aceptación       →  nada llega a main sin entenderlo, testearlo y registrarlo
        │
        └── demo, agentes, review de riesgos y AI_USAGE.md (Parte B) lo aplican al Trabajo Final
```

**Siguiente paso en clase:** [`ejercicios/E6_asistentes.md`](ejercicios/E6_asistentes.md).

---

# PARTE B — Aplicar el protocolo con asistentes

> La Parte A dice *qué* debe cumplir el uso de IA en el desarrollo.  
> La Parte B enseña *cómo se hace* con un asistente real (demo sobre un módulo con deuda → tu proyecto) y *cómo se deja evidencia* (tests, review, `AI_USAGE.md`).

---

## B1 — Anatomía: herramientas y piezas

| Herramienta | Tipo | Encaje en clase | Fichero de contexto del proyecto |
| --- | --- | --- | --- |
| **Cursor** | IDE con chat y edición multiarchivo | Refactors guiados viendo el diff en el editor | Reglas del proyecto (`.cursor/rules/`) |
| **Claude Code** | Agente en terminal sobre el repo | Tareas de varios pasos: leer, editar, ejecutar tests | `CLAUDE.md` |
| **Gemini CLI** | Agente en terminal | Exploración rápida, prototipos, explicar código | `GEMINI.md` |

| Pieza del proyecto | Rol |
| --- | --- |
| `prompts/catalogo_prompts.md` | Plantillas: explicar, refactor, tests, docs, diseño de feature |
| Fichero de contexto (`CLAUDE.md`, `GEMINI.md`, reglas) | Cómo se ejecutan los tests, convenciones, qué no tocar |
| Rama de trabajo (`git switch -c refactor/...`) | Aislar los cambios de la IA de `main` |
| `pytest` + CI | Freno de mano |
| `AI_USAGE.md` | Registro de aportaciones y rechazos |

> La herramienta concreta es secundaria: las tres sirven si aplicas el mismo protocolo. Lo que se evalúa es el **proceso** (diffs pequeños, tests, registro), no el asistente.

Fichero de contexto mínimo (adapta el nombre a tu herramienta):

```markdown
# Contexto del proyecto
- Python 3.12, Flask. Tests: `pytest -q` (sin red; AI_PROVIDER=mock por defecto).
- Capas: ingest.py (ingesta) · core/ (procesamiento puro) · ai_client.py (borde IA) · app.py (API).
- No añadir dependencias sin preguntar. No tocar el contrato de /analyze.
- Nunca leer ni escribir `.env`.
```

---

## B2 — Demo en vivo: refactor de `modulo_deuda.py`

Módulo con deuda deliberada: nombres pobres, función larga, `print` como efecto lateral, `== None`, fichero sin cerrar.

```python
def f(a):
    # a es un dict con keys u, p
    r=[]
    for i in a:
        u=a[i].get('u')
        p=a[i].get('p')
        if u==None:
            continue
        ...
        r.append(u*p)
    s=0
    for x in r:
        s=s+x
    print('total', s)  # side effect
    return s
```

### 1. Red de seguridad: test del comportamiento actual (antes de tocar nada)

```python
# test_modulo_deuda.py
from modulo_deuda import f

def test_total_ignora_lineas_invalidas():
    data = {
        "a": {"u": 2, "p": 3.0},     # 6.0
        "b": {"u": None, "p": 1},    # ignorada
        "c": {"u": 1, "p": -5},      # ignorada
        "d": {"u": 4, "p": 0.5},     # 2.0
    }
    assert f(data) == 8.0
```

```bash
cd sesiones/2026-11-23_asistentes/ejemplos
pytest -q            # verde ANTES del refactor
```

### 2. Explicar → plan → un paso

```text
Prompt 1: Explica qué hace este módulo, sus code smells y riesgos. No reescribas código todavía.
Prompt 2: Propón un plan de refactor en máximo 5 pasos (sin código), priorizando legibilidad y testabilidad.
Prompt 3: Implementa SOLO el paso 1 del plan. Diff mínimo. Sin dependencias nuevas.
```

Plan típico que debería salir (y que tú validas):

1. Renombrar `f` → `total_importes` y extraer la validación de una línea a una función pura.
2. Quitar el `print` (efecto lateral) y devolver solo el valor.
3. Usar `is None` y `sum(...)`.
4. Cerrar el fichero en `load_and_run` (`with open(...)`).
5. Añadir tipos y docstring.

### 3. Revisar el diff y ejecutar tests

```bash
git diff             # ¿lo entiendo? ¿solo toca el paso 1?
pytest -q            # mismo resultado que antes
```

### 4. Tests asistidos y rechazo de alucinaciones

```text
Prompt 4: Escribe tests pytest para la lógica de cálculo de importes. No uses red. Incluye un caso con valores inválidos.
```

| Qué observar | Por qué |
| --- | --- |
| El test de la red de seguridad sigue verde | El refactor no cambió el comportamiento |
| Algún test usa una función o fixture que no existe | Alucinación: se **borra** y se registra en `AI_USAGE.md` |
| Algún test “arregla” el resultado esperado para que pase | Señal de que el código cambió de comportamiento |
| El diff del paso 1 es pequeño | Revisable en un minuto |

---

## B3 — Prompts del catálogo, con contexto real

Las plantillas de [`prompts/catalogo_prompts.md`](prompts/catalogo_prompts.md) funcionan mejor si añades **restricciones** del proyecto:

| Plantilla | Añade siempre | Ejemplo de restricción |
| --- | --- | --- |
| 1. Explicar código | “No reescribas” | “Señala qué partes tocan disco o red” |
| 2. Refactor SOLID | Comportamiento observable fijo | “Los tests de `test_app.py` deben seguir pasando sin cambios” |
| 3. Tests pytest | Sin red, sin dependencias nuevas | “Usa el `MockClient`; `AI_PROVIDER=mock`” |
| 4. Documentación | Comandos reales | “Usa exactamente los comandos de la sección Despliegue” |
| 5. Diseño de feature | “No implementes aún” | “Respeta el contrato de `/analyze` (E2E I, A1)” |

Ejemplo aplicado al Trabajo Final:

```text
Eres un revisor senior de Python.
Contexto: API Flask con capas ingest · core · ai_client · app (ver CLAUDE.md).
Objetivo: añadir validación de longitud máxima de `text` en /analyze (máx. 5000 caracteres).
Restricciones:
- Respeta la taxonomía de errores: entrada inválida → InvalidInput (400).
- Sin dependencias nuevas. No cambies la forma de la respuesta.
- Añade un test en test_app.py.
Salida: solo el diff y una lista de cambios.
```

---

## B4 — Agentes en el repo: control y permisos

Los agentes de terminal (Claude Code, Gemini CLI) y el modo agente de Cursor pueden **leer, editar y ejecutar comandos**. Más potencia = más disciplina.

| Práctica | Motivo |
| --- | --- |
| Trabajar en una **rama** (`git switch -c refactor/importes`) | `main` intacto; el cambio se revisa como PR |
| Partir de un **árbol limpio** (`git status` vacío) | Distingues lo que hizo el agente de lo tuyo |
| Pedir **plan antes de ejecutar** (modo plan / “no edites aún”) | Validas la intención antes de los cambios |
| Revisar los **permisos** de comandos (no aprobar `rm`, `git push`, `pip install` a ciegas) | El agente ejecuta en tu máquina |
| Revisar **`git diff`** después de cada tarea | Ves todo lo que cambió, no solo lo que dice el chat |
| **Commits pequeños** tuyos, con mensaje tuyo | Historial legible y reversible |
| Excluir secretos (`.env` fuera del alcance, `.gitignore`) | El agente lee lo que encuentra en el repo |

```bash
git switch -c refactor/importes
git status                     # limpio
# ... sesión con el asistente: explicar → plan → paso 1 ...
git diff --stat                # ¿qué ficheros tocó?
git diff                       # ¿lo entiendo?
pytest -q && git commit -am "Refactor: extrae total_importes (paso 1/5)"
```

> Si el agente propone instalar algo: para, comprueba que el paquete existe y es el que crees (nombre exacto en PyPI) y añádelo tú a `requirements.txt` con versión acotada.

---

## B5 — Review de riesgos asistido (antes del release)

Prompt de review centrado en lo que ya sabes que importa (E2E II y III):

```text
Revisa este diff/módulo SOLO buscando:
1) secretos o datos personales,
2) validación de entradas,
3) llamadas externas sin timeout,
4) manejo de errores (excepciones tragadas, trazas al cliente),
5) complejidad innecesaria.
Para cada hallazgo: fichero, línea, riesgo concreto y propuesta mínima.
No incluyas consejos genéricos.
```

Triaje de la respuesta:

| Hallazgo | Decisión |
| --- | --- |
| Concreto, con línea y riesgo real (p. ej. `requests.post` sin `timeout`) | **Corregir** (con test si aplica) |
| Genérico (“considera añadir más logging”) | Ignorar |
| Falso positivo (la validación ya existe en otra capa) | Ignorar y anotarlo |
| Cambio grande de arquitectura | Anotar como mejora futura; no ahora |

Regla de [`E7`](ejercicios/E7_refactor_tests_docs.md) (hoy): elige **3 hallazgos reales** y corrígelos; el resto es ruido.

---

## B6 — `AI_USAGE.md`: registro y citación

Un registro útil es **breve, concreto y verificable**. Una entrada por tarea relevante:

```markdown
# Uso de IA en el proyecto

## 2026-11-16 — Refactor de `importes.py`
- **Herramienta:** Claude Code (o Cursor / Gemini CLI) · modelo si se conoce
- **Objetivo:** función de 45 líneas sin tests → función pura testeable
- **Prompts:** explicar → plan (5 pasos) → paso 1 (catálogo #1, #2)
- **Aceptado:** extracción de `total_importes`; `is None`; `sum(...)`
- **Rechazado:** test que usaba `pytest.approx_dict` (no existe); propuesta de añadir `pydantic` (dependencia innecesaria)
- **Verificación:** `pytest -q` → 14 passed · diff de 32 líneas revisado
- **Commit:** `a1b2c3d`
```

| Campo | Para qué |
| --- | --- |
| Herramienta | Transparencia |
| Objetivo | Qué problema resolvía |
| Prompts (resumen) | Reproducibilidad del proceso, no el volcado entero |
| Aceptado / rechazado | Demuestra revisión crítica |
| Verificación | Evidencia: tests, tamaño del diff |
| Commit | Trazabilidad en el historial |

> No pegues conversaciones completas: un resumen de 5–8 líneas por tarea basta (E6, parte 2).

---

## B7 — Propiedades (chuleta de diseño)

| Propiedad | Pregunta | En el proyecto |
| --- | --- | --- |
| Autoría responsable | ¿Puedo explicar cada línea? | Protocolo de aceptación (A3) |
| Contexto acotado | ¿La IA ve solo lo necesario? | Un módulo + restricciones + fichero de contexto |
| Incrementalidad | ¿Cambios pequeños y reversibles? | Un paso del plan por commit |
| Verificabilidad | ¿Un test respalda el cambio? | Red de seguridad antes del refactor |
| Veracidad | ¿Las APIs existen? | Ejecutar, no suponer |
| Seguridad | ¿Hay secretos en prompts, código o logs? | Review de riesgos + `git grep` |
| Trazabilidad | ¿Se sabe qué aportó la IA? | `AI_USAGE.md` con aceptados y rechazados |

---

## B8 — Puente al Trabajo Final (Proyecto III)

```text
E2E I–III  →  contratos + robustez + despliegue (lo que no puedes romper)
Hoy        →  IA asistida con protocolo + AI_USAGE.md
              + release candidate (E7): review de riesgos, README de demo, tag v0.9.0-rc1
30 nov     →  presentación: se pregunta por el uso de IA y se revisa AI_USAGE.md
```

Reutiliza: los tests de E2E como red de seguridad de cualquier refactor · la taxonomía de errores y el contrato de `/analyze` como restricciones en tus prompts · el smoke del despliegue para comprobar que nada se rompió en producción.

Para la evaluación (criterio 8 del enunciado): `AI_USAGE.md` con entradas reales, incluidos rechazos, y capacidad de explicar en la presentación cualquier parte del código.

---

## B9 — Anti-patrones

1. “Hazme el proyecto entero” sobre un repo vacío.  
2. Aceptar todo el diff sin leerlo (“Accept all”).  
3. Pedir a la IA que modifique los tests para que pasen.  
4. Refactorizar sin una red de seguridad de tests previa.  
5. Diffs de cientos de líneas mezclando refactor, features y estilo.  
6. Instalar dependencias sugeridas sin comprobar que existen y hacen falta.  
7. Pegar `.env`, claves o datos personales en el prompt.  
8. Dejar al agente trabajar directamente sobre `main` o aprobarle comandos sin leerlos.  
9. No registrar el uso de IA, o registrar solo lo aceptado.  
10. No poder explicar en la presentación un fragmento “que hizo la IA”.

---

## B10 — Autoevaluación

### Clave

1. ¿Por qué se dice que “tú firmas” el código generado y qué implica en el Trabajo Final?  
2. ¿Qué elementos debe tener un prompt para código y por qué se encadenan (explicar → plan → paso → tests)?  
3. ¿Cuáles son los puntos del protocolo de aceptación?  
4. ¿Qué se registra en `AI_USAGE.md` y por qué también los rechazos?

### Suplementario

5. ¿Cómo detectas que un test generado ha alucinado una API?  
6. ¿Por qué se escribe un test del comportamiento actual **antes** de refactorizar?  
7. ¿Qué precauciones tomas al dejar que un agente ejecute comandos en tu repo?  
8. ¿Cómo distingues un hallazgo real de ruido en un review de riesgos asistido?

---

## B11 — Checklist de salida

**Parte A**

- [ ] Conozco los riesgos del código generado y cómo detectarlos  
- [ ] Mis prompts tienen objetivo, contexto acotado y restricciones  
- [ ] Aplico el protocolo de aceptación antes de cada commit  

**Parte B**

- [ ] Refactor de un smell real en pasos pequeños, en una rama  
- [ ] Red de seguridad de tests antes del cambio; `pytest -q` verde después  
- [ ] Tests generados revisados; los alucinados, borrados y registrados  
- [ ] Review de riesgos con al menos un hallazgo real corregido  
- [ ] `AI_USAGE.md` con explicación, plan, aceptados y rechazados  
- [ ] E6 hecho / protocolo aplicado al Trabajo Final  
- [ ] E7: tag `v0.9.0-rc1` + README reproducible + `RELEASE.md`  

---

## B12 — Apéndices

### Chuleta del protocolo

```bash
git switch -c refactor/<tema>          # rama de trabajo
git status                             # árbol limpio
pytest -q                              # red de seguridad en verde ANTES

# Asistente: explicar → plan → SOLO paso 1
git diff --stat && git diff            # ¿lo entiendo? ¿solo el paso 1?
git diff requirements.txt              # ¿dependencias sorpresa?
pip check                              # ¿dependencias coherentes?
git grep -nI -E "(api[_-]?key|token|secret)\s*=\s*['\"]?[A-Za-z0-9_-]{12,}"   # ¿secretos?
pytest -q                              # verde DESPUÉS

git commit -am "Refactor: <paso 1/5>"  # commit pequeño, mensaje tuyo
# + entrada en AI_USAGE.md
```

### Glosario

| EN / símbolo | Nota |
| --- | --- |
| coding assistant | herramienta de IA que sugiere o edita código (Cursor, Claude Code, Gemini CLI) |
| agent | asistente que lee, edita y ejecuta comandos en el repo en varios pasos |
| prompt | instrucción + contexto que se le da al modelo |
| context window | cantidad de texto que el modelo puede tener en cuenta a la vez |
| hallucination | respuesta segura pero falsa (API, librería o dato inexistente) |
| code smell | síntoma de mal diseño (función larga, nombres pobres, duplicación) |
| refactor | cambio de estructura que **no** cambia el comportamiento observable |
| safety net test | test del comportamiento actual que se escribe antes de refactorizar |
| diff | diferencia entre versiones; lo que se revisa antes de aceptar |
| `AI_USAGE.md` | registro del uso de IA en el proyecto |

### FAQ

**¿Puedo usar la IA en el Trabajo Final?** Sí, con el protocolo y registrándolo en `AI_USAGE.md`. Lo que no se acepta es entregar código que no sabes explicar.  
**¿Qué herramienta uso?** La que prefieras (Cursor, Claude Code, Gemini CLI…). Se evalúa el proceso, no la herramienta.  
**¿Tengo que citar cada autocompletado?** No: solo aportaciones relevantes (funciones, tests, documentación, decisiones de diseño).  
**¿Qué hago si la IA insiste en una librería que no conozco?** Comprueba que existe, que está mantenida y que realmente la necesitas; si no, recházala y regístralo.  
**¿Puedo pegar datos del proyecto en el prompt?** Solo datos de prueba o anonimizados; nunca `.env`, claves ni datos personales.  
**¿Y si el refactor de la IA rompe un test?** No toques el test: revisa si cambió el comportamiento. Si no lo entiendes, `git restore` y pide un paso más pequeño.

### Rúbrica

| Criterio | Insuficiente | Adecuado | Sólido |
| --- | --- | --- | --- |
| Responsabilidad | No explica el código generado | Explica los cambios principales | Explica cualquier línea y sus alternativas |
| Prompts | Genéricos (“mejóralo”) | Con objetivo y contexto | + restricciones del proyecto y cadena explicar → plan → paso |
| Incrementalidad | Diffs enormes | Cambios por pasos | + un paso por commit en rama/PR |
| Verificación | Sin tests | Tests tras el cambio | + red de seguridad previa y alucinaciones descartadas |
| Seguridad | Secretos o datos en prompts/código | Sin secretos | + review de riesgos con hallazgos corregidos |
| Citación | Sin `AI_USAGE.md` | Registro de lo aceptado | + rechazos, verificación y commit enlazado |

---

## Cierre

> **IA asistida = tú firmas + contexto acotado y prompts en cadena + protocolo de aceptación (diff entendido, APIs reales, tests verdes, sin secretos), con cada aportación y cada rechazo registrados en `AI_USAGE.md`.**

**Ahora:** [`ejercicios/E6_asistentes.md`](ejercicios/E6_asistentes.md).
