# Plantillas del equipo

| Archivo | Qué es |
|---|---|
| [`PLAN-INSTALACION-EQUIPO.md`](PLAN-INSTALACION-EQUIPO.md) | Instalación del entorno por fases (Windows, Linux y macOS), con sus verificaciones |
| [`PROMPT-SETUP-CLAUDE.md`](PROMPT-SETUP-CLAUDE.md) | Prompt para que Claude Code audite tu máquina e instale solo lo que falta según el plan |
| [`GUIA-FLUJO-EQUIPO-IA.md`](GUIA-FLUJO-EQUIPO-IA.md) | Cómo trabajamos: uv, Orca + worktrees, ciclo de cada feature hasta el merge y la arquitectura |
| [`CLAUDE-TEMPLATE.md`](CLAUDE-TEMPLATE.md) | Base para el `CLAUDE.md` de cada repositorio nuevo |

Además, [`postgres/`](postgres/) contiene el `compose.yaml` y el `.env.example` base de PostgreSQL para copiar en cada proyecto.

## Orden para un miembro nuevo

1. Prepara tu máquina con `PLAN-INSTALACION-EQUIPO.md`, a mano o pegándole a Claude Code el prompt de `PROMPT-SETUP-CLAUDE.md`.
2. Lee `GUIA-FLUJO-EQUIPO-IA.md` antes de tocar cualquier repositorio.
3. Al crear un repositorio, copia `CLAUDE-TEMPLATE.md` como `CLAUDE.md` y completa sus `<huecos>`.
