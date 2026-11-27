# Guía de flujo de trabajo con IA

Este manual explica cómo trabaja el equipo en los repositorios de software e IoT. Es obligatorio. Está basado en el flujo real que seguimos en `tienda-acuarios`, con Python 3.13, uv, FastAPI, pytest, Git/GitHub CLI, Orca con worktrees y Claude Code.

> **Regla de oro:** nadie trabaja directamente sobre `main`. Cada tarea vive en su propio worktree, su propia rama y su propio PR.

---

## 1. Python con `uv`, nunca con `pip` global

**Por qué:**

- **Un entorno por proyecto.** Cada repo tiene su `.venv` y no se mezclan versiones entre proyectos ni con el Python del sistema.
- **Reproducible.** `uv.lock` fija la versión exacta de cada dependencia. El mismo `uv sync` deja el mismo entorno en todos los PCs, en CI y en Docker.
- **La versión de Python también se fija.** Va en `.python-version` (usamos 3.13), y uv la descarga si no la tienes.
- **Rápido.** uv instala en segundos, así que crear y destruir worktrees no cuesta nada.

`pip install` global rompe todo lo anterior: instala paquetes que nadie más tiene, no deja registro en el repo y produce el clásico "en mi máquina funciona".

**Uso:**

```sh
uv init --app --no-package mi-proyecto   # app sin src/ ni build-system (ver reglas abajo)
uv add fastapi sqlalchemy      # dependencia de la app (actualiza pyproject.toml + uv.lock)
uv add --dev pytest httpx      # dependencia solo de desarrollo/tests
uv sync                        # instala exactamente lo que dice uv.lock
uv run pytest                  # ejecuta cualquier comando dentro del entorno del proyecto
uv run uvicorn app.main:app --reload
```

**Reglas:**

- **No edites a mano** las dependencias en `pyproject.toml`. Usa siempre `uv add` o `uv remove`.
- **Commitea siempre `uv.lock`.** Nunca commitees `.venv/`.
- **No necesitas activar el entorno.** `uv run <comando>` ya usa el `.venv` correcto.
- **Apps que no se instalan.** Si el proyecto no es una librería instalable (una API o un servicio), no necesita `src/` ni `[build-system]`. Créalo con `uv init --app --no-package`: desde uv 0.12, `uv init` (incluso con `--app`) genera un paquete en `src/`. Eso pasó en `tienda-acuarios`: `uv init` generó un paquete en `src/` que la app no usaba. Al borrarlo, `uv` falló porque seguía intentando construirlo. La solución fue quitar `[build-system]` y `[project.scripts]` y añadir esto a `pyproject.toml`:

  ```toml
  [tool.uv]
  package = false
  ```

---

## 2. Trabajo en paralelo: Orca + Git Worktrees

Un **worktree** es una segunda (o tercera, o décima) copia de trabajo del mismo repositorio, en otra carpeta y con otra rama, que comparte el mismo historial de `.git`. **Orca** crea y gestiona esos worktrees y abre en cada uno su terminal, su agente de Claude Code y su navegador.

**Por qué:** cinco personas, y varios agentes de IA, pueden trabajar a la vez en features distintas sin cambiar de rama, sin guardar cambios a medias y sin pisar archivos ajenos. Cada worktree tiene su carpeta, su rama y su `.venv`.

**Reglas:**

1. **Un worktree por tarea o feature.** No reutilices un worktree para otra cosa.
2. **Nunca trabajes en `main`.** La carpeta principal del repo (la que tiene `main`) solo se usa para hacer `git pull` y para probar lo ya fusionado.
3. **Nombra el worktree como la rama:** `feat/carrito-whatsapp`, `fix/stock-negativo`, `iot/sensor-ph`.
4. **Los archivos ignorados por Git no se copian.** Cada worktree nuevo nace sin `.env`, sin base de datos local (`*.db`) y sin `.venv`. Copia `.env.example` → `.env` y ejecuta `uv sync` en cada worktree.
5. **`git stash` es compartido entre todos los worktrees** del repo. No uses `git stash` / `git stash pop` a secas, porque puedes sacar el stash de otro. Para apartar trabajo, haz un commit temporal (`wip: ...`) en tu rama.
6. **Una rama no puede estar abierta en dos worktrees a la vez.** Por eso `gh pr merge --delete-branch` no puede borrar la rama local mientras su worktree exista; la borra solo en GitHub. Al cerrar la tarea, elimina el worktree (paso 9 del ciclo).
7. **Usa puertos distintos** si levantas servidores en varios worktrees a la vez: `--port 8001`, `--port 8002`, etc.

---

## 3. Ciclo de vida de una funcionalidad

```
Crear worktree → Leer/actualizar CLAUDE.md → Implementar con Claude Code → uv run pytest
→ Proteger secretos → Commit + Push → gh pr create → Revisión y merge → Eliminar worktree
```

### 3.1 Crear el worktree en Orca

Desde la interfaz de Orca: **nuevo worktree** en el repo, con un nombre descriptivo y `main` como base.

O desde la CLI:

```sh
orca worktree create --name feat-carrito --branch feat/carrito-whatsapp --no-parent --json
# con un agente ya lanzado y su encargo:
orca worktree create --name feat-carrito --branch feat/carrito-whatsapp --no-parent --agent claude --prompt "Implementa ..." --json
```

Dentro del worktree:

```sh
cp .env.example .env      # y rellena tus valores locales
uv sync
```

### 3.2 Leer y actualizar `CLAUDE.md`

`CLAUDE.md` es el contrato entre el repo y Claude Code: comandos, estructura, reglas y decisiones de arquitectura. Claude lo lee al empezar cada sesión.

- **Antes de pedir nada,** comprueba que `CLAUDE.md` refleja el estado actual. En un repo nuevo, parte de `CLAUDE-TEMPLATE.md` o usa `/init` en Claude Code.
- **Si la tarea cambia** comandos, estructura, variables de entorno o reglas de negocio, el mismo PR debe actualizar `CLAUDE.md`. Un `CLAUDE.md` desactualizado hace que la IA repita errores.

### 3.3 Implementar con Claude Code

- **Da encargos completos y verificables:** qué construir, con qué restricciones y cómo se valida. Ejemplo: "endpoints X e Y, precios en COP como enteros, tests de Z".
- **Pide que verifique de verdad:** que levante el servidor, llame a la API y abra la página en el navegador de Orca. Que escriba el código no basta.
- **Revisa lo que reporta.** Si cambió algo que no pediste (un código HTTP, una dependencia), tiene que decirlo. Si no lo dice, pregúntale.
- **Claude no hace commit, push ni merge** salvo que se lo pidas explícitamente.

### 3.4 Validar con `uv run pytest`

```sh
uv run pytest                        # toda la suite
uv run pytest tests/test_api.py -k pedido   # subconjunto
```

- **Sin tests en verde no hay commit.** Esto también aplica a Claude Code.
- **Los tests nunca tocan la base de datos real.** Usa una BD temporal, configurada en `conftest.py` antes de importar la app.
- **Cada funcionalidad nueva trae al menos** un test del caso feliz y uno del caso de error principal.

### 3.5 Proteger los secretos

- **`.env` va en `.gitignore` siempre,** desde el primer commit.
- **`.env.example` sí se versiona.** Lleva cada variable con un **valor ficticio**, por ejemplo `WHATSAPP_NUMBER=573000000000` o `ADMIN_TOKEN=cambia_esto_en_produccion`.
- **Nunca pongas datos reales en archivos versionados,** tampoco en `.env.example`. En `tienda-acuarios` se subió un número real en `.env.example` y luego se reemplazó, pero **sigue en el historial de Git**. Para borrarlo de verdad hay que reescribir el historial; un commit nuevo no basta. Si un secreto llega a GitHub, considéralo filtrado y cámbialo: rota el token o la clave.
- **Uvicorn no lee `.env` solo.** Arráncalo con `--env-file .env`.
- **Bases de datos locales** (`*.db`), logs y artefactos de build también van en `.gitignore`.

Antes de cada commit:

```sh
git status --short        # ¿aparece .env, *.db o algo inesperado?
git diff --staged         # revisa lo que realmente se va a subir
```

### 3.6 Commit y push

```sh
git add -A
git commit -m "feat: carrito con pedido por WhatsApp"
git push -u origin feat/carrito-whatsapp
```

Usamos prefijos convencionales: `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`.

### 3.7 Pull Request con `gh pr create`

```sh
gh pr create --base main --title "feat: carrito con pedido por WhatsApp" --body "Resumen... Pruebas..."
```

El cuerpo del PR debe decir **qué cambia** y **cómo se probó**: los tests y la verificación manual.

### 3.8 Revisión y merge a `main`

- **Otra persona del equipo revisa** el PR antes de fusionarlo.
- **Merge:**

  ```sh
  gh pr merge <n> --merge --delete-branch
  ```

- **Después, actualiza la carpeta principal:**

  ```sh
  git -C <carpeta-principal> pull --ff-only
  uv sync
  uv run pytest
  ```

### 3.9 Eliminar el worktree

Desde Orca (eliminar worktree), o desde la CLI:

```sh
orca worktree rm --worktree name:feat-carrito --delete-branch
# o solo con git, desde fuera de esa carpeta:
git worktree remove <ruta-del-worktree>
git branch -D feat/carrito-whatsapp
```

---

## 4. Arquitectura: monolito con `static/` o `frontend/` separado

| | **Monolito con `static/`** | **`frontend/` separado (Vite + React)** |
|---|---|---|
| Cuándo | MVPs, validación con clientes, paneles internos, demos de IoT | Productos core, UI compleja o con mucho estado, varias personas en el front |
| Stack | FastAPI sirve `index.html` + JS sin build + Tailwind por CDN | Vite + React (+ TypeScript), build propio, Tailwind compilado |
| Despliegue | Un solo proceso y un solo contenedor | Dos piezas: la API y los estáticos compilados (servidos por la API, Nginx o un CDN) |
| Tiempo hasta la primera versión | Horas | Días |
| Señales para migrar | El JS supera unos pocos archivos, hay rutas o estado compartido entre vistas, se necesitan tests de UI o la UI necesita trabajo offline | — |

**Monolito** (como `tienda-acuarios`):

```
app/            # FastAPI: main.py, models.py, schemas.py, database.py, config.py
static/         # index.html, admin.html, js/*.js
tests/
```

- Cada `*.html` lo sirve FastAPI en su ruta (`/`, `/admin`).
- Todo el contenido que venga de la BD y se inserte con `innerHTML` se escapa.
- Tailwind por CDN **necesita internet** y no se optimiza. Es aceptable para un MVP, no para producción a escala.

**`frontend/` separado:**

```
backend/  (o app/)    # FastAPI, solo /api/...
frontend/             # npm create vite@latest frontend -- --template react-ts
docker-compose.yml
```

- **En desarrollo,** Vite (`npm run dev`) redirige `/api` al backend con `server.proxy` en `vite.config.ts`. Así no hay problemas de CORS.
- **El contrato entre ambos** es el OpenAPI de FastAPI (`/docs`, `/openapi.json`). Los cambios de API se hacen en PRs que actualizan las dos partes o que mantienen compatibilidad hacia atrás.
- **Las dependencias de Node** se gestionan con `npm`, con `package-lock.json` commiteado. uv sigue siendo solo para Python.

---

## 5. Comandos esenciales

| Herramienta | Comando | Para qué |
|---|---|---|
| **uv** | `uv init --app --no-package <nombre>` | Crear un proyecto nuevo (app, sin `src/`) |
| | `uv add <pkg>` / `uv add --dev <pkg>` | Añadir una dependencia de la app o de desarrollo |
| | `uv remove <pkg>` | Quitar una dependencia |
| | `uv sync` | Instalar exactamente lo que dice `uv.lock` |
| | `uv run <cmd>` | Ejecutar dentro del entorno del proyecto |
| | `uv run pytest` | Ejecutar los tests |
| | `uv run uvicorn app.main:app --reload --env-file .env` | Servidor de desarrollo |
| **git** | `git status --short` | Ver qué cambió (y que no haya secretos) |
| | `git diff --staged` | Revisar lo que se va a commitear |
| | `git commit -m "feat: ..."` | Hacer commit |
| | `git push -u origin <rama>` | Subir la rama |
| | `git pull --ff-only` | Actualizar `main` sin crear merges accidentales |
| | `git worktree list` | Ver todos los worktrees del repo |
| | `git worktree remove <ruta>` | Eliminar un worktree |
| **gh** | `gh auth status` | Comprobar la sesión de GitHub |
| | `gh repo create <org>/<repo> --private` | Crear el repo remoto (privado por defecto) |
| | `gh pr create --base main` | Abrir un PR |
| | `gh pr view <n>` / `gh pr checks <n>` | Ver el estado y el CI de un PR |
| | `gh pr merge <n> --merge --delete-branch` | Fusionar y borrar la rama remota |
| **orca** | `orca worktree create --name <tarea> --no-parent` | Nuevo espacio de trabajo |
| | `orca worktree list` | Listar espacios de trabajo |
| | `orca worktree rm --worktree name:<tarea> --delete-branch` | Eliminar un espacio de trabajo |
| **docker compose** | `docker compose up -d --build` | Levantar los servicios (reconstruyendo las imágenes) |
| | `docker compose ps` | Ver el estado de los servicios |
| | `docker compose logs -f <servicio>` | Seguir los logs |
| | `docker compose exec <servicio> sh` | Abrir una terminal dentro de un contenedor |
| | `docker compose down` | Parar los servicios (con `-v` también borra los volúmenes y sus datos) |

---

## 6. Problemas que ya nos pasaron

- **Windows y Git Bash:** `curl` con tildes en el cuerpo (`-d '{"cliente":"José"}'`) envía el texto en cp1252 y FastAPI responde "error parsing the body". Para probar la API con texto en español, usa los tests, `/docs` o un script de Python.
- **La consola de Windows muestra `P�rez`** aunque la BD guarde `Pérez` bien. Antes de concluir que hay un bug de codificación, verifica los datos con `ascii(...)` o leyendo directamente la BD.
- **Popups bloqueados:** abrir `wa.me` después de un `fetch` lo bloquea el navegador. La pestaña hay que abrirla en el mismo clic, antes del `fetch`.
- **Precios y stock siempre se calculan en el servidor.** Nunca confíes en los totales que envía el navegador.
