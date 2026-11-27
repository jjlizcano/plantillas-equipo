# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

<!--
PLANTILLA DE LA EMPRESA. Copia este archivo como CLAUDE.md en la raíz del repo nuevo y:
  1. Reemplaza todo lo que está entre <ángulos>.
  2. Borra las secciones o líneas que no apliquen (p. ej. frontend/ si es un monolito).
  3. Borra este comentario.
Mantenlo corto: Claude lo lee en cada sesión. Actualízalo en el mismo PR que cambie
comandos, estructura, variables de entorno o reglas de negocio.
-->

## Proyecto

<Una o dos frases: qué es, para quién y el stack. Ej.: "API FastAPI + SQLite que sirve un frontend estático para la tienda X.">

Idioma: textos para usuarios y nombres en el código en <español/inglés>. Mantén el que ya existe.

## Reglas obligatorias

Estas reglas no son sugerencias. Si una tarea parece exigir romper alguna, detente y pregunta.

### Python y dependencias: solo `uv`

- **Usa siempre `uv`.** Nunca `pip install`, `python -m pip`, `poetry` ni `conda`, y nunca instales nada en el Python global.
- **Añade dependencias con `uv add <pkg>`** (dev/tests: `uv add --dev <pkg>`) y quítalas con `uv remove <pkg>`. No edites a mano las listas de dependencias de `pyproject.toml`.
- **Ejecuta todo con `uv run <cmd>`.** No actives el `.venv` ni llames a `python` o `pytest` directamente.
- **`uv.lock` se commitea siempre;** `.venv/` nunca.
- **No cambies la versión de Python** (`.python-version`) sin que te lo pidan.

### Estructura de carpetas

- **El código de la app vive en `<app/>`.** Los tests viven en `tests/`.
- **No crees carpetas de primer nivel nuevas** (`src/`, `lib/`, `utils/`…) sin que te lo pidan. Si algo no encaja en la estructura de abajo, pregunta.
- **<Monolito: el frontend es estático en `static/`, sin paso de build.>**
  **<Separado: el frontend vive en `frontend/` (Vite + React) y se gestiona con `npm`. uv es solo para Python.>**

### Secretos y `.env`

- **Nunca leas, muestres ni commitees `.env`.** `.env`, `*.db` y `.venv/` deben estar en `.gitignore`.
- **Toda variable de entorno nueva** se documenta en el mismo cambio, en `.env.example` (con un **valor ficticio**) y en la sección "Variables de entorno" de abajo.
- **Ningún dato real en archivos versionados,** tampoco en `.env.example`, los tests ni los seeds: nada de teléfonos, correos, tokens, claves ni URLs internas. Usa placeholders como `573000000000` o `cambia_esto`.
- **Si encuentras un secreto ya versionado,** avisa. No lo "arregles" en silencio: el historial de Git lo sigue conteniendo.

### Tests antes de cada commit

- **Ejecuta `uv run pytest` antes de cada commit.** Si falla, **no hagas commit**: arregla el código o informa del fallo con su salida.
- **Todo cambio de comportamiento trae tests** del caso feliz y del caso de error principal.
- **Los tests nunca usan la BD ni los servicios reales.** Usan la configuración de `tests/conftest.py`.
- **No borres, saltes (`skip`) ni debilites tests** para que la suite pase.

### Git y flujo de trabajo

- **Nunca hagas commit en `main`.** Cada tarea vive en su propio worktree de Orca y su propia rama.
- **No uses `git stash` a secas.** El stash es compartido entre todos los worktrees del repo. Para apartar trabajo, haz un commit `wip:` en la rama.
- **Haz commit, push, PR y merge solo cuando te lo pidan.** Nunca fusiones un PR por iniciativa propia.
- **Mensajes de commit** con prefijo convencional: `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`.
- **Antes de commitear,** revisa `git status --short` y `git diff --staged` para confirmar que no se cuela `.env`, ninguna BD ni archivos ajenos a la tarea.
- **Si cambias un código de respuesta, una dependencia o algo que no se pidió,** dilo explícitamente en tu respuesta.

## Comandos

```sh
uv sync                                                   # instalar dependencias
uv run <comando para levantar la app>                     # ej.: uv run uvicorn app.main:app --reload --env-file .env
uv run pytest                                             # toda la suite
uv run pytest tests/<archivo>.py::<test>                  # un solo test
<docker compose up -d --build>                            # si aplica
<comando de lint/format, si existe>
```

## Variables de entorno

Copia `.env.example` a `.env`. <La app no carga `.env` sola: arráncala con `--env-file .env`.>

| Variable | Para qué | Si no está definida |
|---|---|---|
| `<NOMBRE>` | <descripción> | <comportamiento por defecto> |

## Arquitectura

<Solo lo que no se ve leyendo un archivo: flujo de datos entre módulos, invariantes, decisiones no obvias. Ejemplos de lo que vale la pena anotar:>

- <Cómo arranca la app (creación de tablas, seeds, migraciones o la falta de ellas).>
- <Dónde viven las reglas de negocio críticas (precios calculados en el servidor, stock, permisos).>
- <Convenciones de datos (p. ej. dinero en enteros COP, fechas en UTC).>
- <Cosas acopladas a un motor o proveedor concreto (funciones SQL propias, APIs externas).>
- <Cómo se comunica el frontend con la API.>
