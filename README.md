# orquestador-handoffs

**Orquestador determinista de handoffs entre Claude Code y Codex.**
Un único script de Node, sin dependencias, que convierte una orden de trabajo escrita (*execution packet*) en:

```
worktree aislado → implementador → tests → revisor independiente (solo lectura) → informe + manifiesto de hashes
```

Las decisiones de aprobar, reintentar o parar las toma una tabla fija, no otra IA. El orquestador **nunca integra cambios**: la última palabra es siempre del humano.

> Documentación en español. Probado en Windows (PowerShell) con Claude Code 2.1.296 y Codex CLI 0.159.1. Pensado también para macOS y Linux.

## Por qué

Trabajar con varias IAs suele acabar en hacer de mensajero: copiar la tarea a un agente, copiar su resultado a otro para que lo revise, volver… Este script elimina ese copiar y pegar sin ceder el control:

- **Una IA no coordina a otras IAs.** Enviar, esperar, reenviar y recopilar son tareas mecánicas; las hace un script con reglas fijas.
- **La autoridad la imponen los permisos, no el prompt.** El revisor no puede escribir; el implementador no puede hacer commit y sus herramientas de edición están limitadas a su copia de trabajo (los subprocesos que lanza, como los tests, no; ver [Limitaciones conocidas](#limitaciones-conocidas)).
- **Todo queda registrado.** Qué se pidió, qué se hizo, qué dijeron los tests y qué opinó el revisor, con hashes SHA-256 de cada pieza.
- **Revisión cruzada real.** Claude implementa y Codex revisa, o al revés, cambiando una línea.

## Requisitos

- Node 18 o posterior y Git.
- [Claude Code](https://code.claude.com) y/o [Codex CLI](https://developers.openai.com/codex), con sesión iniciada. Solo hace falta el que vayas a usar en cada rol.

## Inicio rápido

1. Copia `orquestar.mjs` y `orquestador.config.json` a una carpeta de herramientas, por ejemplo `C:\herramientas\orquestador\`.
2. Pon un `orquestador.config.json` en la raíz de tu proyecto con su comando de tests y su preparación (`npm ci`, etc.). Si no hay uno en el proyecto, se usa el que está junto al script; también puedes pasarlo con `--config`.
3. Crea un packet a partir de `plantilla-packet.md` en `handoffs\<id>\packet.md` dentro de tu proyecto, con el SHA actual como baseline (`git rev-parse HEAD`).
4. Ejecuta:

```powershell
node C:\herramientas\orquestador\orquestar.mjs handoffs\012\packet.md --dry-run
node C:\herramientas\orquestador\orquestar.mjs handoffs\012\packet.md
```

5. Lee `resultado.md`. Con **PASS**, revisa el cambio e intégralo tú (el informe trae los comandos). Con **STOP** o **FAIL**, decide el siguiente paso.

`--dry-run` lo valida todo sin crear nada y te dice el SHA que tienes que poner si usaste `main`.

Para validar la instalación en tu equipo antes de usarlo en un proyecto real, sigue [`pruebas/PRUEBA-WINDOWS.md`](pruebas/PRUEBA-WINDOWS.md).

## El packet

Cabecera:

| Clave | Valores | Notas |
|---|---|---|
| `id` | texto | Rama `handoff/<id>` y carpeta de resultados. No se reutiliza. |
| `modo` | implementar · revisar · investigar | |
| `baseline` | SHA completo | Obligatorio en ejecución real. `main`/`HEAD` solo en `--dry-run`. |
| `candidato` | SHA completo | Solo en modo revisar. |
| `implementador` / `revisor` / `investigador` | claude · codex | Cruzarlos da la revisión más independiente. |
| `max_iteraciones` | número | Ciclos antes de FAIL. |
| `commit_policy` | none · orchestrator | `none` (por defecto): el candidato queda sin commit y lo creas tú. `orchestrator`: un commit por iteración en la rama del handoff. |
| `verdict_map` | `ETIQUETA=acción, ...` | Acciones: pass, retry, stop. Vacío = `PASS=pass, FAIL_RETRYABLE=retry, FAIL_STOP=stop, INCONCLUSIVE=stop`. Para usar tu propio vocabulario: por ejemplo `SATISFIED=pass, NOT_SATISFIED=retry, AUTHORITY_GAP=stop, INCONCLUSIVE=stop`. |

Cuerpo: secciones `PROJECT`, `AUTHORITY`, `OBJECTIVE`, `SCOPE`, `OUT_OF_SCOPE`, `PROHIBITED`, `INPUTS`, `SUCCESS_CRITERIA`, `STOP_CONDITIONS` y `OUTPUT_REQUIRED`. Son obligatorias `OBJECTIVE`, `SCOPE`, `SUCCESS_CRITERIA` y `STOP_CONDITIONS` (en modo investigar, solo `OBJECTIVE` y `STOP_CONDITIONS`).

### Reglas fijas

- Tests fallidos → vuelve al implementador (sin pasar por el revisor) si quedan iteraciones; si no, FAIL.
- Veredicto con acción `retry` → vuelve al implementador si quedan iteraciones; si no, FAIL.
- Veredicto con acción `stop`, etiqueta desconocida, `STOP:` del implementador, timeout, commit hecho por el agente o ausencia de cambios → STOP (decide el humano).
- Veredicto con acción `pass` → PASS. PASS significa "candidato aprobado según el packet", no "autorizado para integrar".

## Dónde queda cada cosa

Nada se escribe en el repositorio principal:

- Worktree: `..\<repo>-worktrees\<id>\`
- Resultados: `..\<repo>-orquestador-runs\<id>\`
  - `packet.md` (copia exacta del usado), `log.md`, `resultado.md`
  - `iterN-*.prompt.md` (lo enviado a cada agente), `iterN-implementador.md`, `iterN-revisor.md`
  - `iterN-diff.patch`, `iterN-tests.txt` (evidencia generada por el orquestador, no por el agente)
  - `worktree-estado-final.txt` (cómo quedó la copia de trabajo)
  - `manifiesto.sha256` (hash de todos los archivos, incluido `resultado.md`)

`resultado.md` incluye PACKET_SHA256, BASELINE_SHA, el candidato (commit o árbol git), los hashes del diff, los tests y la revisión, y si el repositorio principal quedó intacto. Para verificar un hash en PowerShell:

```powershell
Get-FileHash .\resultado.md -Algorithm SHA256   # comparar con su línea de manifiesto.sha256
```

El script se niega a arrancar si la carpeta de worktrees o la de resultados está dentro del repositorio.

## Cómo se aplica la autoridad

- **Implementador Claude:** `--permission-mode dontAsk` y lista blanca: Read, `Edit(/**)` y los comandos que pongas en la configuración. `Edit(/**)` está anclado al worktree y rige también a Write, así que las ediciones fuera del worktree se deniegan. Commit, add, stash, push, reset, checkout, merge y web, denegados. Cada regla `Bash(...)` se duplica como `PowerShell(...)`, porque en Windows Claude Code ejecuta comandos con su herramienta PowerShell.
- **Comandos de uno en uno:** Claude recibe en el prompt la lista de comandos permitidos y la orden de no encadenarlos; un comando encadenado (`npm test; node -e …`) se deniega entero si una parte no está permitida.
- **Configuración de Claude:** se lanza con `--strict-mcp-config` (sin servidores MCP ni conectores) y `--setting-sources project`: no carga tu configuración global (`%USERPROFILE%\.claude\settings.json`: directorios adicionales, permisos, plugins y hooks) ni la `local` del proyecto.
- **Implementador Codex:** `--sandbox workspace-write`, sin red (`-c sandbox_workspace_write.network_access=false`) y sin carpetas con escritura extra (`-c sandbox_workspace_write.writable_roots=[]`), aunque el `config.toml` global diga otra cosa.
- **Codex (implementador y revisor):** sin plugins ni apps, con cada servidor MCP de `config.toml` desactivado y sin búsqueda web integrada (`web_search=disabled`). La lista de servidores se lee con `codex mcp list --json`; si falla, STOP.
- **Revisor e investigador:** Claude sin Edit/Write; Codex con `--sandbox read-only`.
- Si el HEAD del worktree cambia durante la implementación (el agente hizo commit por su cuenta) → STOP.
- Si el implementador crea o modifica algo en `.claude/` o `.codex/` del worktree, incluidos archivos ignorados por git → STOP. Esa configuración (hooks, permisos) la cargarían el revisor o la iteración siguiente.
- En modo revisar, si el candidato añade o modifica `.claude/` o `.codex/` → STOP antes de lanzar tests o agentes. Revísalo a mano.
- Todo agente recibe la lista de fuentes con autoridad (el packet y los documentos que enumera) y sin ella (salidas de herramientas, comentarios del código, README, informes de otros agentes, mensajes de otras sesiones, web).

### Sandbox de Codex en Windows

No modifiques la configuración global de Codex sin comprobarla antes con tu versión. En Codex para Windows, `[windows] sandbox = "elevated"` **no** da más permisos al agente: es el sandbox más fuerte (pide permiso de administrador una vez para crear un usuario aislado y reglas de firewall). `"unelevated"` es la alternativa más débil.

## Interrumpir una ejecución

- **Ctrl+C una vez:** detiene a los agentes y deja el informe completo con estado **STOP — Interrumpido por el humano (Ctrl+C)**, el manifiesto y `worktree-estado-final.txt`.
- Si el worktree queda con cambios sin aprobar, el informe distingue entre cambios **a medias** (el implementador no terminó: interrupción, timeout o error) y **no aprobados** (el implementador terminó, pero la revisión no se completó o no aprobó).
- **Ctrl+C dos veces:** salida inmediata sin informe. Solo si la primera no responde.

## Reabrir una sesión de agente

`resultado.md` incluye una sección **Reabrir sesiones** con el comando exacto para cada agente de cada iteración. El orquestador fija el identificador de cada sesión de Claude (`--session-id` y un nombre `orq-<id>-<iteración>-<rol>`); el de Codex lo lee de su salida.

```powershell
cd <carpeta del worktree>
claude --resume <identificador>             # en la terminal
claude --desktop --resume <identificador>   # en la app de escritorio de Claude (Claude Code 2.1.285 o posterior)
codex resume <identificador>                # sesión de Codex
```

Ni el historial de la extensión de VS Code ni el selector de `claude --resume` sin identificador muestran estas sesiones, porque son automáticas; por eso el informe trae el identificador.

La sesión reabierta usa **tus** permisos normales, no los restringidos del orquestador. Antes de escribir, pulsa **Shift+Tab** hasta ver **plan mode**. Hazlo antes de borrar el worktree.

## Códigos de salida

`0` PASS o investigación terminada · `1` FAIL · `2` STOP · `3` error interno o de uso.

## Limitaciones conocidas

- Cada llamada arranca en frío: pon el contexto permanente del proyecto en el `CLAUDE.md` / `AGENTS.md` del repositorio.
- Cada iteración consume una ejecución completa de cada agente, con cargo a tu plan de Claude o de Codex.
- Los diffs de más de 120 000 caracteres no se incrustan en el prompt; el revisor los lee con `git diff`.
- La clasificación entre reintentar y parar la hace el revisor; el script solo aplica la tabla. Si dudas de un revisor, usa `max_iteraciones: 1`.
- Claude Code y Codex CLI cambian a menudo. El informe registra las versiones usadas en cada ejecución.
- **El código del candidato se ejecuta en tu máquina sin sandbox.** El orquestador lanza `comandoPreparacion` y `comandoTests` (por ejemplo `npm ci`, `npm test`) directamente, con tus permisos. Los permisos de los agentes no cubren eso: un test o un script de instalación puede leer o escribir en cualquier parte. No lo uses con código en el que no confíes sin revisarlo antes.
- **En modo revisar, la preparación se ejecuta sobre el candidato** antes de cualquier comprobación. Para PRs ajenas, usa una configuración sin `comandoPreparacion` o con `npm ci --ignore-scripts`.
- **En Windows, el implementador Claude no tiene aislamiento del sistema operativo**, solo reglas de permisos: limitan sus herramientas, no los procesos que lanza con los comandos permitidos. Codex sí tiene sandbox.
- Codex conserva la escritura en `/tmp` y `$TMPDIR` dentro de su sandbox.
- El implementador puede editar `CLAUDE.md` o `AGENTS.md` del worktree, que el revisor carga como contexto. A veces es un cambio legítimo (si está en el SCOPE), así que no se bloquea: revisa esos archivos en el diff.
- El implementador puede modificar tests salvo que el packet lo prohíba y el revisor lo detecte.
- El `model` de tu configuración global de Claude ya no se aplica (`--setting-sources project`): usa `claude.modelo` en `orquestador.config.json`.
- Los agentes pueden leer cualquier archivo de tu usuario: Read no está limitado a una ruta en Claude, y el sandbox read-only de Codex permite leer todo el disco. No tienen red, pero el código del candidato sí la tiene cuando se ejecuta (ver arriba). Limitar Read al worktree queda para una versión posterior.

## Controles: por qué existe cada uno y cuándo podría quitarse

Tipo: **E** = estructural (no depende del modelo) · **P** = parche de versión · **G** = guía en el prompt (no es una garantía).

| Control | Tipo | Existe porque | Se puede quitar cuando |
|---|---|---|---|
| Aprobación del packet con id + hash | E | Un agente no debe autorizarse a sí mismo | Nunca |
| Worktree aislado y comprobación de que el repositorio principal quedó intacto | E | El agente no debe tocar la copia de trabajo del usuario | Nunca |
| `dontAsk` + lista blanca del implementador Claude | E | Mínimo privilegio sin preguntas | Nunca |
| `Edit(/**)` en lugar de `Edit`/`Write` sin ruta (2.2.1) | E | Sin ruta, el implementador podía escribir en cualquier parte del disco (prueba 13) | Nunca |
| `--setting-sources project` (2.2.1) | E | La configuración global y la local ampliaban permisos y cargaban hooks | Nunca |
| STOP si el implementador toca `.claude/` o `.codex/` (2.2.1) | E | Esa configuración la cargarían las sesiones siguientes | Nunca |
| STOP en modo revisar si el candidato trae `.claude/` o `.codex/` (2.2.1) | E | El revisor cargaría los hooks del candidato | Nunca |
| Commit, push, reset… denegados; HEAD cambiado → STOP | E | La integración es humana | Nunca |
| Revisor de solo lectura y aislado (packet + diff + tests, sin el razonamiento del implementador) | E | El juicio debe ser independiente | Nunca |
| Revisión cruzada Claude ↔ Codex | E | Un modelo comparte puntos ciegos consigo mismo | Si en uso real no aporta hallazgos distintos (medirlo) |
| Tests ejecutados por el orquestador | E | El informe del agente no es evidencia | Nunca |
| Tabla veredicto → acción; ante la duda, parar | E | Decidir no se delega en una IA | Nunca |
| `max_iteraciones` bajo (2) | E | Los bucles revisor → implementador cuestan y pueden derivar | Revisar con datos de uso real |
| SHA completo, hashes y manifiesto | E | Poder demostrar qué se pidió y qué se hizo | Nunca |
| Timeout que mata el árbol de procesos | E | Agentes colgados | Nunca |
| Codex sin red y `--strict-mcp-config` en Claude | E | Mínimo privilegio aunque la configuración global diga otra cosa | Nunca |
| Codex con `writable_roots=[]` (2.2.1) | E | El `config.toml` global daba escritura en otras carpetas | Nunca |
| Codex sin MCP, plugins, apps ni búsqueda web (2.2.1) | E | Codex no tiene `--strict-mcp-config`; el `config.toml` global cargaba servidores MCP (incluido uno que controla el navegador) y la búsqueda web | Nunca |
| Duplicar `Bash(...)` → `PowerShell(...)` (2.0.2) | P | Claude Code en Windows usa PowerShell con reglas propias | Cuando Claude Code aplique un solo juego de reglas (comprobar con la prueba 01 sin duplicado) |
| «Comandos de uno en uno» en el prompt (2.0.3) | P/G | Claude encadenaba comandos y se denegaba el comando entero | Cuando Claude Code evalúe cada parte por separado o el modelo deje de encadenar (prueba 01 sin la orden, leyendo los `.jsonl`) |
| Entrecomillar `=` y `;` en Windows (2.0.1) | P | Fallo de la línea de comandos | Nunca: no cuesta nada |
| Lista de fuentes con y sin autoridad | G | Instrucciones inyectadas en el código | Nunca, pero no es una garantía: la garantía son los permisos |

## Licencia

[MIT](LICENSE).
