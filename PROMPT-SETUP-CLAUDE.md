# Prompt de configuración del equipo con Claude Code

Sirve para que Claude Code audite tu máquina, te muestre qué tienes y qué te falta, e instale **solo lo que falta** según `PLAN-INSTALACION-EQUIPO.md`.

## Antes de empezar

1. **Instala Claude Code a mano.** Es lo único que no puede instalar él mismo.
   - 🪟 Primero Git (`winget install --id Git.Git -e`) y luego, en PowerShell: `irm https://claude.ai/install.ps1 | iex`.
   - 🐧🍎 `curl -fsSL https://claude.ai/install.sh | bash`.
2. **Copia los archivos** `PLAN-INSTALACION-EQUIPO.md` y `PROMPT-SETUP-CLAUDE.md` a una carpeta, por ejemplo `~/Desarrollo/plantillas/`.
3. **Abre una terminal en esa carpeta.** Sirven `cmd`, PowerShell o bash. 🪟 Ábrela **sin** permisos de administrador: Claude te pedirá a ti los pasos que requieren admin.
4. **Ejecuta `claude`,** inicia sesión si te lo pide y pega el prompt de abajo, completo.
5. **Durante el proceso:**
   - Claude te pedirá aprobar los comandos. Lee lo que vas a aprobar.
   - Cuando haga falta un inicio de sesión interactivo (por ejemplo `gh auth login`), Claude te pedirá escribirlo con `!` delante, en la misma sesión: `! gh auth login`.
   - Si la máquina necesita reiniciarse, Claude te avisará. Después vuelve a abrir `claude` en la misma carpeta y escribe: `continúa el setup desde donde quedó`.

---

## Prompt (copiar desde aquí)

```text
Actúa como ingeniero DevOps del equipo. Vas a preparar ESTA máquina siguiendo el archivo
PLAN-INSTALACION-EQUIPO.md que está en el directorio actual. Léelo completo antes de ejecutar nada.
Trabaja en español.

## Reglas que no puedes romper
1. Primero auditas, luego preguntas y luego instalas. Las fases A y B son de solo lectura:
   no instales, desinstales ni modifiques nada hasta que yo apruebe el plan de la fase C.
2. Instala SOLO lo que falte o esté por debajo de la versión requerida. Nunca desinstales,
   bajes de versión ni reinstales algo que ya funciona.
3. No sobrescribas configuraciones existentes. Antes de modificar cualquier archivo de
   configuración (.wslconfig, .gitconfig, perfiles de shell, settings de Docker), crea una copia
   <archivo>.bak-<AAAAMMDD-HHMM> y modifica solo las claves necesarias, conservando el resto.
4. No cambies user.name ni user.email de git si ya tienen valor; solo muéstralos.
5. Nunca ejecutes comandos destructivos: wsl --unregister, docker system prune,
   docker volume rm, docker compose down -v sobre proyectos existentes, ni borres carpetas o
   repositorios. Si crees que algo debería borrarse, dímelo y lo decido yo.
6. Antes de `wsl --shutdown` o de reiniciar Docker, avísame: detiene Docker y todas las
   distribuciones de WSL. Espera mi confirmación.
7. Si un paso requiere administrador (sudo, PowerShell elevada), reinicio o inicio de sesión
   interactivo (gh auth login, primer arranque de Docker Desktop, claude login), no lo intentes
   rodear: dame el comando exacto y espera a que te diga que está hecho.
8. No muestres tokens, contraseñas ni contenidos de archivos .env.
9. En Windows usa PowerShell para consultar el sistema. Si un comando no existe en el shell
   actual, usa el equivalente correcto y no inventes salidas: todo dato de las tablas debe
   venir de un comando que hayas ejecutado.

## Fase A · Auditoría (solo lectura)
Detecta y anota:
- Sistema: SO, edición (Windows 11 Home/Pro, distro y versión de Linux, versión de macOS),
  arquitectura (x64/arm64).
- Hardware: RAM física total en GB (en Windows, la suma de Win32_PhysicalMemory), hilos
  lógicos de CPU, disco libre en la unidad donde irá la carpeta Desarrollo, y si la
  virtualización está activa.
- Windows: estado de WSL (wsl --status, wsl -l -v), si existe %USERPROFILE%\.wslconfig y su
  contenido, si el alias de python de la Microsoft Store está activo, y si Documentos o
  Escritorio están redirigidos a OneDrive.
- Versiones instaladas (o "no instalado") de: git, gh (y si hay sesión con gh auth status),
  node, npm (y si node es LTS: node -p "process.release.lts"), uv, Python 3.13 gestionado
  por uv (uv python list --only-installed), docker, docker compose (y si el motor responde),
  PowerShell 7 (Windows), winget/brew/apt/snap según el SO, DBeaver, Bruno, Orca
  (en Linux el CLI es orca-ide; nunca ejecutes "orca" en Linux porque puede ser el lector de
  pantalla de GNOME), y claude (claude --version).
- Configuración de git: user.name, user.email, init.defaultBranch, core.autocrlf, pull.rebase.
- Rutas: si existe la carpeta Desarrollo recomendada (sección 1.5 del plan) y si hay
  repositorios git dentro de Google Drive, OneDrive, iCloud o Dropbox (busca solo en esas
  carpetas, hasta 3 niveles, y solo lista las rutas).

## Fase B · Diagnóstico
Muéstrame:
1. Un resumen del equipo: SO y edición, RAM, CPU, disco libre.
2. Una tabla comparativa con columnas:
   | Componente | Requerido por el plan | Encontrado | Estado | Acción propuesta |
   Estado ∈ {OK, Actualizar, Falta, Mal configurado, Revisar}. Incluye una fila por cada
   herramienta de las fases 0 a 5 del plan, una para .wslconfig (solo Windows), una para la
   configuración de git y una para la regla de rutas.
3. El cálculo de recursos de Docker según la sección 1.3 del plan, mostrando la cuenta:
   memory = max(4, floor(RAM_GB × 0.4)) GB, processors = max(2, floor(hilos / 2)), swap = 2GB.
   En Windows indica si el .wslconfig se crearía nuevo o se modificaría (y qué claves
   cambiarían). En macOS indica el valor para Docker Desktop → Resources. En Linux indica
   que no aplica.
4. Advertencias: disco libre menor de 20 GB, RAM menor de 8 GB, virtualización desactivada,
   repositorios en carpetas sincronizadas, Windows Home con Hyper-V solicitado, etc.
5. El plan de instalación: la lista ordenada y exacta de comandos que ejecutarías, agrupada
   por fase del plan, marcando cuáles requieren admin, reinicio o acción mía.

Luego DETENTE y pregúntame si apruebo el plan (todo, una parte o nada).

## Fase C · Instalación (solo tras mi aprobación)
- Sigue el orden de las fases del plan: 0 Base → 1 Python/uv → 2 Docker/PostgreSQL →
  3 Git/GitHub → 4 Herramientas → 5 Orca → 6 Claude Code.
- Ejecuta solo lo aprobado. Después de cada fase, ejecuta sus comandos de "Verificar" y
  muéstrame el resultado en una línea por comando (OK/FALLA + versión o error).
- Si algo falla, no sigas acumulando cambios: diagnostica con la sección
  "Problemas conocidos" del plan, propón la corrección y espera mi visto bueno si implica
  algo distinto de lo aprobado.
- Si tras instalar algo el comando no se reconoce, recuerda que la terminal puede necesitar
  reabrirse; dímelo en vez de modificar el PATH a mano.
- Windows: crea o ajusta .wslconfig con los valores calculados (regla 3: copia .bak y solo
  claves de [wsl2]), y luego pídeme permiso para aplicar con wsl --shutdown.

## Fase D · Verificación de punta a punta
Ejecuta la Fase 7 del plan completa en una carpeta desechable dentro de Desarrollo
(prueba-setup y prueba-front), incluyendo PostgreSQL con la plantilla compose.yaml de la
sección 2.2, usando un .env temporal con credenciales de prueba y un DB_PORT libre.
Para DBeaver y Bruno, que son gráficos, dame los datos exactos de conexión o de petición
y pregúntame el resultado. Al terminar, limpia solo lo que creaste en esta fase
(docker compose down -v de ESE proyecto de prueba y las carpetas de prueba).

## Informe final
Termina con:
1. La tabla comparativa de la Fase B, actualizada con el estado final.
2. La lista de verificación de la Fase 7 con ✅/❌ por ítem.
3. Lo que quedó pendiente y requiere acción mía (inicios de sesión, reinicios, pasos con
   admin, configuraciones que no toqué por la regla 3), con el comando exacto para cada uno.
4. Las copias .bak que creaste y dónde están.
```
