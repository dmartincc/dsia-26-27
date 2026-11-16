# E6 — Entorno open source + taller con asistentes sobre código existente (50 min)

**Sesión:** 16 nov 2026  
**Prompts:** `../prompts/catalogo_prompts.md`  
**Teoría:** `../01_ia_asistida.md`, B1 (agente ≠ modelo, panorama de herramientas, entorno con Nemotron)

## Reglas

- Trabaja sobre **tu Proyecto III**, no sobre un repo vacío, y en una rama.
- Ninguna clave de API en el repo ni en los prompts.
- No commits de código que no entiendas.
- Todo uso relevante de IA va a `AI_USAGE.md`.

## Parte 0 — Montar el entorno: agente open source + Nemotron (20 min)

Objetivo: un agente **open source** trabajando sobre tu repo con el modelo **Nemotron 3.5 Super VL 120B-A12B** servido en <https://build.nvidia.com/nvidia/nemotron-3.5-super-vl-120b-a12b>. Ese entorno es el que usarás en las Partes 1–4.

### 0.1 — Clave de API (3 min)

1. Entra en la página del modelo con tu cuenta NVIDIA (gratuita) → **Get API Key**.
2. Guárdala **solo** en una variable de entorno:

   ```bash
   export NVIDIA_API_KEY="nvapi-..."        # en ~/.zshrc o ~/.bashrc, nunca en el repo
   echo ".env" >> .gitignore                 # si además usas .env
   ```

### 0.2 — Smoke test del modelo (4 min)

Antes de culpar al agente, comprueba que el modelo responde:

```bash
python -m venv .venv-ia && source .venv-ia/bin/activate
pip install openai
python smoke_nemotron.py                   # script de 01_ia_asistida.md, B1.3
```

1. Ejecútalo con `enable_thinking: False` y con `True`. Anota en `AI_USAGE.md` la diferencia de `usage.completion_tokens` y de tiempo.
2. *(Opcional, multimodal)* Pásale una captura de un error de tu proyecto:

   ```python
   import base64
   img = base64.b64encode(open("error.png", "rb").read()).decode()
   messages = [{"role": "user", "content": [
       {"type": "text", "text": "Explica este error y dónde mirar. No propongas código aún."},
       {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img}"}},
   ]}]
   # misma llamada que el smoke test, con temperature=0.6 (visión)
   ```

   Si la API rechaza el formato de imagen, consulta el ejemplo de la pestaña de código de la página del modelo y anótalo como incidencia.

### 0.3 — Instalar y configurar UN agente (8 min)

| Opción | Instalación | Configuración |
| --- | --- | --- |
| **A. OpenCode** (recomendada) | `npm i -g opencode-ai` (o `curl -fsSL https://opencode.ai/install \| bash`) | `opencode.json` de B1.3 en la raíz del repo → `opencode` |
| **B. Aider** | `python -m pip install aider-install && aider-install` | Variables `OPENAI_API_BASE` / `OPENAI_API_KEY` de B1.3 → `aider --model openai/nvidia/nemotron-3.5-super-vl-120b-a12b --read AGENTS.md` |
| **C. Cline** (VS Code) | Extensión “Cline” del marketplace | Provider **OpenAI Compatible**, base URL, clave e id del modelo de B1.3 |

Comprobaciones:

```bash
git grep -n "nvapi-" || echo "OK: ninguna clave en el repo"
git status                                 # opencode.json sí se puede versionar (usa {env:...}); la clave no
```

### 0.4 — Contexto del proyecto (3 min)

Crea `AGENTS.md` en la raíz de tu Proyecto III con la plantilla mínima de B1.2 (stack, cómo se ejecutan los tests, capas, qué no tocar, “nunca leer `.env`”). Si usas Aider, pásalo con `--read AGENTS.md`.

### 0.5 — Primera prueba (2 min)

En una rama (`git switch -c ia/entorno`), pide al agente:

```text
Lee AGENTS.md. Dime en 5 líneas qué hace este repo, cómo se ejecutan los tests
y qué ficheros no debes tocar. No edites nada.
```

Si responde citando ficheros reales de tu repo, el agente lee el proyecto y llama bien a sus herramientas.

### Hecho cuando (Parte 0)

- [ ] El smoke test responde con y sin thinking
- [ ] Un agente (OpenCode, Aider o Cline) responde sobre tu repo con Nemotron
- [ ] `AGENTS.md` versionado; `git grep nvapi-` vacío
- [ ] Entrada en `AI_USAGE.md`: herramienta, modelo, tokens con y sin thinking, incidencias de instalación

> Si la API de NVIDIA no está disponible (límite, caída), haz las Partes 1–4 con cualquier otra herramienta de B1.2 y apúntalo: el protocolo no cambia.

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

Entorno de la Parte 0 operativo + commit/PR de refactor + `AI_USAGE.md` con herramienta y modelo, explicación, plan, aceptación/rechazo.
