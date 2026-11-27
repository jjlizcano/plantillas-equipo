# Plan de instalación · Equipo de desarrollo

**Stack:** Python 3.13 con uv · FastAPI · Node.js LTS · PostgreSQL en Docker · Git + GitHub CLI (`gh`) · DBeaver · Bruno · Claude Code · Orca.
**Sistemas soportados:** Windows 11 (Home o Pro), Linux (Ubuntu/Debian) y macOS.
**Cómo usarlo:** ejecuta las fases 0 a 6 en orden. Cada fase termina con su verificación, y la fase 7 es la verificación de punta a punta. Puedes hacerlo a mano o pedírselo a Claude Code con `PROMPT-SETUP-CLAUDE.md`.

Convenciones:

- **(admin)**: PowerShell abierto como administrador. Si no se indica, se usa una PowerShell normal.
- **🪟 / 🐧 / 🍎**: comandos para Windows, Linux o macOS.
- **No desinstales ni bajes de versión** nada que ya funcione. Si algo ya está instalado y cumple la versión, sáltalo.

---

## 1. Reglas que dependen de cada máquina

### 1.1 Requisitos mínimos

| Recurso | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB o más |
| Disco libre | 20 GB | 50 GB (las imágenes de Docker crecen) |
| CPU | 64 bits, virtualización activada en la BIOS/UEFI (para Docker) | 4 núcleos o más |

### 1.2 Windows 11 Home y Pro

| | **Home** | **Pro** |
|---|---|---|
| Motor de Docker | **WSL 2, obligatorio.** Home no tiene Hyper-V | **WSL 2, estándar del equipo.** Hyper-V solo si hay un motivo concreto |
| Límite de memoria | `.wslconfig` (obligatorio, ver 1.3) | `.wslconfig` si usa WSL 2. Si usa Hyper-V: *Docker Desktop → Settings → Resources*, con el mismo 40 % |
| Distribución de Linux | No hace falta. Se trabaja desde PowerShell, y Docker crea su propia distribución `docker-desktop` | Igual |

Para saber qué edición tienes:

```powershell
(Get-CimInstance Win32_OperatingSystem).Caption
```

### 1.3 Memoria de WSL: el 40 % de la RAM física

Si no se limita, WSL 2 (y con él Docker) puede quedarse con buena parte de la RAM y volver lento el equipo.

**Regla:** `memory = redondear hacia abajo(RAM física en GB × 0,4)`, con un **mínimo de 4 GB**. `processors` = la mitad de los hilos lógicos, con un mínimo de 2.

| RAM física | `memory` |
|---|---|
| 8 GB | 4GB (el mínimo; 3,2 no alcanza) |
| 16 GB | 6GB |
| 32 GB | 12GB |
| 64 GB | 25GB |

Script (PowerShell normal). Calcula los valores, y crea `%USERPROFILE%\.wslconfig` **solo si no existe**:

```powershell
$ramGB = [math]::Round((Get-CimInstance Win32_PhysicalMemory | Measure-Object Capacity -Sum).Sum / 1GB)
$memGB = [math]::Max(4, [math]::Floor($ramGB * 0.4))
$cpus  = [math]::Max(2, [math]::Floor((Get-CimInstance Win32_ComputerSystem).NumberOfLogicalProcessors / 2))
$ruta  = "$env:USERPROFILE\.wslconfig"
"RAM física: $ramGB GB -> memory=${memGB}GB, processors=$cpus"

if (Test-Path $ruta) {
    "Ya existe $ruta; no se sobrescribe. Contenido actual:"; Get-Content $ruta
    "Edita a mano solo las claves memory/processors/swap de la sección [wsl2]."
} else {
    @"
[wsl2]
memory=${memGB}GB
processors=$cpus
swap=2GB
"@ | Set-Content -Path $ruta -Encoding ascii
    "Creado $ruta"
}
```

Para aplicarlo, guarda tu trabajo y ejecuta `wsl --shutdown`, que **detiene Docker y todas las distribuciones**. Después vuelve a abrir Docker Desktop.

**Verificar:** `wsl -d docker-desktop -- free -g` muestra un total cercano al valor de `memory`.

### 1.4 Linux y macOS

- **🐧 Linux:** Docker Engine nativo, sin VM ni `.wslconfig`. Los contenedores comparten la RAM del sistema. Si un servicio consume demasiado, limítalo en el `compose.yaml` con `mem_limit`.
- **🍎 macOS:** Docker Desktop, que corre en su propia VM. No hay `.wslconfig`: aplica la misma regla del 40 % en *Docker Desktop → Settings → Resources → Memory*.
- **Instaladores:** 🐧 `apt` (más los repositorios oficiales indicados) y 🍎 [Homebrew](https://brew.sh) (`brew`).

### 1.5 Dónde viven los repositorios

- **Una carpeta local dedicada:** 🪟 `C:\Desarrollo` o `%USERPROFILE%\Desarrollo` · 🐧🍎 `~/Desarrollo`.
- **Prohibido** guardar repositorios dentro de **Google Drive, OneDrive, iCloud Drive, Dropbox** o cualquier carpeta sincronizada. La sincronización corrompe `.git`, bloquea archivos de `.venv` y `node_modules` y sube secretos (`.env`) a la nube.
- 🪟 **Cuidado con OneDrive:** si tiene activada la copia de seguridad de carpetas, *Documentos* y *Escritorio* están dentro de OneDrive. No uses esas carpetas. Para comprobarlo: `[Environment]::GetFolderPath('MyDocuments')` no debe contener `OneDrive`.
- 🍎 **Cuidado con iCloud:** si "Escritorio y Documentos" está activado en iCloud, no uses `~/Documents` ni `~/Desktop`.
- **Sin espacios, `ñ` ni tildes** en las rutas de código.

Estructura:

```
Desarrollo/
├── clientes/      un repositorio por cliente o encargo
├── productos/     productos propios
└── plantillas/    guías, CLAUDE-TEMPLATE.md, compose.yaml base
```

---

## Fase 0 · Base: terminal, Git, Node.js y Claude Code

🪟 PowerShell normal (winget pide elevación solo cuando la necesita):

```powershell
winget install --id Microsoft.PowerShell -e          # PowerShell 7
winget install --id Microsoft.WindowsTerminal -e     # viene en Windows 11; si ya está, se omite
winget install --id Git.Git -e                       # Claude Code en Windows necesita Git for Windows
winget install --id OpenJS.NodeJS.LTS -e
irm https://claude.ai/install.ps1 | iex              # Claude Code (instalador nativo)
New-Item -ItemType Directory -Force "$env:USERPROFILE\Desarrollo" | Out-Null
```

En *Windows Terminal → Configuración → Inicio*, pon **PowerShell** (7) como perfil predeterminado.

🐧 Linux (Ubuntu/Debian):

```bash
sudo apt update && sudo apt install -y git curl build-essential ca-certificates
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash - && sudo apt install -y nodejs
curl -fsSL https://claude.ai/install.sh | bash
mkdir -p ~/Desarrollo
```

🍎 macOS:

```bash
xcode-select --install                                # herramientas de línea de comandos (incluye git)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install git node
curl -fsSL https://claude.ai/install.sh | bash
mkdir -p ~/Desarrollo
```

Cierra la terminal y abre otra nueva. **Verificar:**

```sh
git --version
node -v; npm -v
node -p "process.release.lts"    # debe imprimir el nombre de la LTS, no "undefined"
claude --version
claude doctor                    # diagnóstico de la instalación de Claude Code
```

---

## Fase 1 · Python con uv

Python **solo** se gestiona con uv. Nunca se usa `pip install` global. Las razones están en `GUIA-FLUJO-EQUIPO-IA.md`.

🪟:

```powershell
winget install --id astral-sh.uv -e
```

Desactiva el alias falso de Python de la Microsoft Store: *Configuración → Aplicaciones → Configuración avanzada de aplicaciones → Alias de ejecución de aplicaciones*, y apaga `python.exe` y `python3.exe`.

🐧:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

🍎:

```bash
brew install uv
```

En los tres sistemas:

```sh
uv python install 3.13
```

**Verificar:**

```sh
uv --version
uv python list --only-installed          # aparece cpython-3.13.x
uv run --python 3.13 python -c "import sys; print(sys.version)"
```

Herramientas de línea de comandos globales, solo si hacen falta: `uv tool install <paquete>`, nunca `pip`.

---

## Fase 2 · Docker y PostgreSQL

### 2.1 Instalar Docker

🪟 (Home y Pro):

```powershell
wsl --install --no-distribution        # (admin) activa WSL 2 sin instalar Ubuntu; reinicia si lo pide
winget install --id Docker.DockerDesktop -e
```

Abre Docker Desktop y, en *Settings → General*, confirma que está marcado **Use the WSL 2 based engine**. Después crea el `.wslconfig` (sección 1.3).

🐧:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER          # cierra sesión y vuelve a entrar para usar docker sin sudo
```

🍎:

```bash
brew install --cask docker             # Docker Desktop; ábrelo una vez y ajusta Resources → Memory (1.4)
```

**Verificar:**

```sh
docker version                         # muestra Client y Server
docker compose version
docker run --rm hello-world
```

🪟 además: `wsl --status` y `wsl -l -v`. La distribución `docker-desktop` debe aparecer con VERSION 2.

### 2.2 PostgreSQL por proyecto

PostgreSQL **no se instala en el sistema**. Cada proyecto lo levanta en un contenedor. Plantilla `compose.yaml` para la raíz del repo:

```yaml
services:
  db:
    image: postgres:17
    ports:
      - "127.0.0.1:${DB_PORT:-5432}:5432"   # solo accesible desde tu máquina
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      retries: 10

volumes:
  db_data:
```

- **`.env`** (en `.gitignore`) con `DB_NAME`, `DB_USER`, `DB_PASSWORD` y, si quieres, `DB_PORT`. **`.env.example`** con las mismas variables y valores ficticios.
- **Volúmenes con nombre,** nunca carpetas del host montadas (en Windows son lentas y dan problemas de permisos).
- **Si dos proyectos usan PostgreSQL a la vez,** cambia `DB_PORT` (`5433`, `5434`, …) en el `.env` de uno de ellos.
- **`docker compose down -v` borra los datos** del volumen. Úsalo solo a propósito.

**Verificar** (desde la carpeta del proyecto):

```sh
docker compose up -d
docker compose ps                                       # db aparece "healthy"
docker compose exec db psql -U <DB_USER> -d <DB_NAME> -c "select version();"
```

---

## Fase 3 · Git y GitHub

```sh
git config --global user.name  "<Nombre Apellido>"
git config --global user.email "<correo de tu cuenta de GitHub>"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

Saltos de línea:

- 🪟 `git config --global core.autocrlf true`
- 🐧🍎 `git config --global core.autocrlf input`

Instalar GitHub CLI:

- 🪟 `winget install --id GitHub.cli -e`
- 🍎 `brew install gh`
- 🐧 Repositorio oficial:

  ```bash
  (type -p wget >/dev/null || sudo apt install wget -y) \
    && sudo mkdir -p -m 755 /etc/apt/keyrings \
    && wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
    && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && sudo apt update && sudo apt install gh -y
  ```

Iniciar sesión, que es interactivo: GitHub.com → HTTPS → autenticar con el navegador:

```sh
gh auth login
```

**Verificar:**

```sh
git config --global --get user.email
gh auth status                          # "Logged in to github.com"
gh repo list <organización> --limit 5   # ves los repos de la organización de la empresa
```

**Normas:** los repositorios de la empresa viven en la **organización de GitHub**, no en cuentas personales, y son **privados** por defecto. Los secretos van solo en `.env` local o en *GitHub → Settings → Secrets*.

---

## Fase 4 · Herramientas de trabajo

| Herramienta | 🪟 Windows | 🐧 Linux | 🍎 macOS | Para qué |
|---|---|---|---|---|
| **DBeaver** | `winget install --id DBeaver.DBeaver.Community -e` | `sudo snap install dbeaver-ce` | `brew install --cask dbeaver-community` | Cliente de PostgreSQL y otras bases de datos |
| **Bruno** | `winget install --id Bruno.Bruno -e` | `sudo snap install bruno` | `brew install --cask bruno` | Probar APIs. Las colecciones se guardan como archivos dentro del repo |
| **VS Code** (opcional) | `winget install --id Microsoft.VisualStudioCode -e` | `sudo snap install code --classic` | `brew install --cask visual-studio-code` | Editor |
| **Google Chrome** (opcional) | `winget install --id Google.Chrome -e` | paquete `.deb` oficial | `brew install --cask google-chrome` | Pruebas de frontend y la extensión *Claude in Chrome* |

Extensiones de VS Code recomendadas:

```sh
code --install-extension ms-python.python
code --install-extension charliermarsh.ruff
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
```

**Verificar:** DBeaver y Bruno abren. 🐧 `snap list | grep -E "dbeaver|bruno"`.

---

## Fase 5 · Orca

Orca gestiona varios espacios de trabajo en paralelo: cada tarea va en su git worktree, con su terminal, su agente de Claude Code y su navegador. Requiere Git, repositorios clonados y Claude Code con sesión iniciada.

- 🪟 `winget install --id StablyAI.Orca -e`
- 🐧🍎 Descarga el instalador desde el sitio oficial de Orca.
- 🐧 **En Linux, el CLI se llama `orca-ide`.** Fuera de las terminales de Orca, `orca` suele ser el **lector de pantalla de GNOME** (`/usr/bin/orca`), y ejecutarlo empieza a hablar.

Abre Orca, añade la carpeta `Desarrollo/` (o cada repo) y comprueba que Claude Code aparece como agente disponible.

**Verificar:**

```sh
orca status            # 🐧: orca-ide status
orca worktree list
```

---

## Fase 6 · Primer uso de Claude Code

```sh
cd <carpeta de un repo>
claude                 # la primera vez pide iniciar sesión con la cuenta de la empresa
```

- **Cada repositorio tiene su `CLAUDE.md`,** a partir de `plantillas/CLAUDE-TEMPLATE.md` o con `/init`.
- **Mantén Claude Code al día.** El instalador nativo se actualiza solo; `claude doctor` muestra el estado.

---

## Fase 7 · Verificación de punta a punta

Crea un proyecto desechable en `Desarrollo/` (bórralo al terminar):

```sh
cd ~/Desarrollo                     # 🪟: cd $env:USERPROFILE\Desarrollo
uv init --app prueba-setup && cd prueba-setup
uv add fastapi "uvicorn[standard]"
```

Contenido de `main.py`:

```python
from fastapi import FastAPI
app = FastAPI()

@app.get("/salud")
def salud():
    return {"ok": True}
```

Arranca el servidor con `uv run uvicorn main:app --port 8000` y, en otra terminal, comprueba que responde: `curl http://127.0.0.1:8000/salud` → `{"ok":true}`.

Lista de comprobación:

- [ ] `git --version`, `gh auth status` (con sesión iniciada) y `git config --global user.email` correcto.
- [ ] `node -p "process.release.lts"` imprime un nombre de LTS. `npm create vite@latest prueba-front -- --template react-ts`, seguido de `npm install` y `npm run dev`, arranca.
- [ ] `uv run --python 3.13 python -c "print('ok')"` imprime `ok`, y el endpoint `/salud` responde.
- [ ] `docker run --rm hello-world` funciona y `docker compose up -d` (plantilla 2.2) deja `db` en estado *healthy*.
- [ ] DBeaver se conecta a `localhost:<DB_PORT>` con los datos del `.env`.
- [ ] Bruno hace `GET http://127.0.0.1:8000/salud` y recibe 200.
- [ ] 🪟 `.wslconfig` tiene el `memory` calculado (1.3) y `wsl -d docker-desktop -- free -g` lo refleja.
- [ ] `claude --version` responde y `claude` abre dentro de un repo.
- [ ] `orca status` (🐧 `orca-ide status`) responde.
- [ ] Ningún repositorio está dentro de Google Drive, OneDrive o iCloud (1.5).

Limpieza: `docker compose down` (con `-v` si era una BD de prueba) y borra `prueba-setup/` y `prueba-front/`.

---

## Problemas conocidos

| Síntoma | Causa y solución |
|---|---|
| `failed to connect to the docker API` / `Cannot connect to the Docker daemon` | 🪟🍎 Docker Desktop no está abierto: ábrelo y espera a "Engine running". 🐧 `sudo systemctl start docker`, o falta cerrar y abrir sesión tras `usermod -aG docker` |
| Docker Desktop no arranca en Windows | Virtualización desactivada en la BIOS/UEFI, o WSL sin instalar: `wsl --install --no-distribution` (admin) y reiniciar |
| El equipo se pone lento con Docker abierto | Falta el `.wslconfig` (1.3) o el límite de *Resources* (🍎) |
| `python` abre la Microsoft Store | El alias de la Store sigue activo (fase 1) |
| `node`, `gh`, `uv` o `claude` no se reconocen | La terminal se abrió antes de instalar: ábrela de nuevo |
| `port is already allocated` al levantar PostgreSQL | Otro proyecto o un PostgreSQL local usa el 5432: cambia `DB_PORT` en el `.env` |
| Errores raros de git, archivos bloqueados o `.env` en la nube | El repo está en una carpeta sincronizada: muévelo a `Desarrollo/` (1.5) |
| 🐧 Ejecutar `orca` empieza a hablar | Es el lector de pantalla de GNOME: usa `orca-ide` |
| 🪟 `curl` con tildes en el cuerpo falla ("error parsing the body") | Git Bash envía el texto en cp1252: prueba la API con Bruno, con `/docs` o con los tests |

## Espacio en disco aproximado

| Componente | Aproximado |
|---|---|
| Git, Node.js, gh, PowerShell 7, uv | ~0,7 GB |
| Python 3.13 (uv) | ~0,1 GB |
| Docker Desktop + WSL / VM | ~3 a 5 GB |
| Imagen de PostgreSQL y volúmenes | ~0,5 a 1 GB por proyecto |
| DBeaver, Bruno, Orca, Claude Code | ~1,5 GB |
| **Total inicial** | **~6 a 9 GB** |
