# Historial de cambios

Todas las versiones se validaron con agentes simulados; desde la 2.0.1, además, en Windows con Claude Code y Codex CLI reales.

## 2.2.1

Aislamiento de la configuración de los agentes. Motivo: la revisión del 26-10-2026 y las pruebas 12, 13 y 14.

- **Hueco corregido, existente desde la 2.0.0:** el implementador Claude recibía `Edit` y `Write` sin ruta en `--allowedTools`, lo que le permitía escribir en cualquier parte del disco. La prueba 13 lo detectó: creó un archivo en la carpeta Descargas del usuario sin ninguna denegación. Ahora recibe `Edit(/**)`, anclado al worktree, que según la documentación de Claude Code rige también a `Write`. Las escrituras fuera del worktree se deniegan.
- Claude se lanza con `--setting-sources project`: no carga la configuración global del usuario (directorios adicionales, permisos, plugins y hooks extra) ni la `local`, que puede crear el agente y git ignora.
- STOP si el implementador crea o modifica algo en `.claude/` o `.codex/` del worktree (incluidos archivos ignorados por git). Esa configuración la cargarían las sesiones siguientes.
- Modo revisar: STOP antes de lanzar tests o agentes si el candidato añade o modifica `.claude/` o `.codex/` (por ejemplo, hooks en una PR ajena).
- Codex:
  - El implementador se lanza con `-c sandbox_workspace_write.writable_roots=[]`, sin las carpetas con escritura extra del `config.toml` global.
  - El implementador y el revisor se lanzan sin plugins ni apps (`features.plugins=false`, `features.apps=false`), con cada servidor MCP de `config.toml` desactivado (`mcp_servers.<nombre>.enabled=false`, a partir de `codex mcp list --json`, con timeout de 60 s) y sin búsqueda web integrada (`web_search=disabled`). Si no se puede obtener la lista de servidores, STOP.
- Nuevas pruebas en `pruebas/`: 12 (el implementador crea un hook en `.claude/`), 13 (escritura en Descargas) y 14 (revisar un candidato con `.claude/settings.json`).

## 2.2.0

- `resultado.md` incluye una sección **Reabrir sesiones** con el comando exacto para cada agente de cada iteración: `claude --resume <id>`, `claude --desktop --resume <id>` (app de escritorio) y `codex resume <id>`.
- Las sesiones de Claude se lanzan con `--session-id` y un nombre `orq-<id>-<iteración>-<rol>`; el identificador de Codex se lee de su salida.
- El aviso de cambios sin aprobar distingue entre cambios **a medias** (el implementador no terminó) y **no aprobados** (terminó, pero no hubo aprobación).

## 2.1.0

- **Ctrl+C limpio:** la primera pulsación detiene a los agentes y deja informe completo con estado STOP; la segunda sale de inmediato.
- Nuevo `worktree-estado-final.txt` en cada ejecución.

## 2.0.3

- Claude recibe en el prompt la lista de comandos que puede ejecutar y la orden de lanzarlos de uno en uno. Motivo: en Windows, Claude encadenaba `npm test; node -e …` y el comando entero se denegaba.
- Claude se lanza con `--strict-mcp-config`, sin servidores MCP ni conectores.

## 2.0.2

- Cada regla de permisos `Bash(...)` se duplica como `PowerShell(...)`. En Windows, Claude Code ejecuta comandos con su herramienta PowerShell, que tiene reglas propias.

## 2.0.1

- El implementador de Codex se lanza sin red (`-c sandbox_workspace_write.network_access=false`), aunque el `config.toml` global la permita.
- Los argumentos con `=` o `;` se entrecomillan en Windows.

## 2.0.0

- Tabla veredicto → acción (`verdict_map`) con vocabulario configurable; por defecto `PASS`, `FAIL_RETRYABLE`, `FAIL_STOP` e `INCONCLUSIVE`. Solo los fallos reintentables vuelven al implementador.
- Baseline (y candidato) con SHA completo obligatorio en ejecución real.
- Hashes SHA-256 del packet, el diff, los tests y las respuestas, y `manifiesto.sha256` de todos los artefactos.
- `commit_policy`: `none` (por defecto) u `orchestrator`.
- Los artefactos se guardan fuera del repositorio principal; el script comprueba al final que este quedó intacto.
- Lista explícita de fuentes con y sin autoridad en todos los prompts.
- STOP si el implementador hace commit o cambia de rama por su cuenta.
- El timeout mata también a los procesos hijo (grupo de procesos en Linux y macOS, `taskkill /T` en Windows).

## 1.0.0

- Prototipo inicial (no publicado): packet, worktree aislado, implementador, tests y revisor en solo lectura, con Claude Code o Codex en cada rol.
