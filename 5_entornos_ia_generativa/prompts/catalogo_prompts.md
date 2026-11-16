# Catálogo de prompts (plantillas)

Adapta estas plantillas a tu código. Sustituye los bloques entre `« »`.

## 1. Explicar código existente

```text
Eres un revisor senior de Python.
Explica qué hace este módulo, sus responsabilidades y riesgos.
No reescribas el código todavía.

«pega el código»
```

## 2. Refactorizar hacia Clean Code / SOLID

```text
Refactoriza el siguiente código aplicando SRP y nombres claros.
Mantén el comportamiento observable.
Devuelve solo el código y una lista breve de cambios.

«pega el código»
```

## 3. Generar tests pytest

```text
Escribe tests pytest para este módulo.
Incluye casos felices, errores esperados y un caso límite.
Usa fixtures si ayuda. No inventes dependencias externas.

«pega el código»
```

## 4. Documentación

```text
Genera un README de 20–30 líneas para este proyecto:
instalación, uso CLI, variables de entorno y cómo ejecutar tests.

Estructura del repo:
«pega tree o rutas»
```

## 5. Diseño de prompt para una feature

```text
Quiero añadir «feature» a este proyecto.
Propón:
1) cambios de archivos,
2) interfaz pública,
3) tests mínimos,
4) riesgos.
No implementes aún.
```

## 6. Arranque del agente en un repo (entorno nuevo)

```text
Lee AGENTS.md (o «fichero de contexto»).
Resume en 5 líneas: qué hace el repo, cómo se ejecutan los tests
y qué ficheros o directorios no debes tocar.
No edites nada ni ejecutes comandos todavía.
```

## 7. Review de riesgos (antes del release)

```text
Revisa este diff/módulo SOLO buscando:
1) secretos o datos personales, 2) validación de entradas,
3) llamadas externas sin timeout, 4) manejo de errores, 5) complejidad innecesaria.
Para cada hallazgo: fichero, línea, riesgo concreto y propuesta mínima.
No incluyas consejos genéricos.
```

> Con modelos con razonamiento configurable (p. ej. Nemotron, `enable_thinking`), activa el razonamiento para las plantillas 1, 5 y 7 (entender, diseñar, revisar) y desactívalo para 2–4 y 6 (cambios mecánicos y arranque).
