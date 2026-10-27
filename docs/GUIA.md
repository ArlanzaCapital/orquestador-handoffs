# Guía del orquestador de handoffs

Versión del orquestador: **2.2.1** (27-10-2026) · Repositorio: [ArlanzaCapital/orquestador-handoffs](https://github.com/ArlanzaCapital/orquestador-handoffs)

Esta guía explica, en lenguaje sencillo, qué es el orquestador, cómo trabaja por dentro y cómo sacarle el máximo partido. Si algo de aquí no coincide con el [`README.md`](../README.md) del repositorio o con el propio `orquestar.mjs`, mandan ellos.

---

## 1. Qué es, en una frase

Un script (`orquestar.mjs`) que **pone a una IA a hacer un trabajo y a otra IA distinta a revisarlo**, siguiendo reglas fijas, guardando pruebas de todo y **sin que ninguna de las dos pueda integrar nada por su cuenta**. La última palabra siempre la tienes tú.

## 2. Qué problema resuelve

Sin el orquestador, trabajar con dos IAs suele ser así:

1. Le escribes la tarea a Claude Code.
2. Copias lo que ha hecho y se lo pegas a Codex para que lo revise.
3. Copias la revisión y se la devuelves a Claude para que corrija.
4. Repites… y al final no sabes bien qué se pidió, qué se cambió ni quién aprobó qué.

Haces de **mensajero**. Además, cada IA ve lo que ha dicho la otra, así que la revisión deja de ser independiente.

El orquestador hace ese ir y venir por ti, con tres ventajas:

- **No pierdes el control.** Nada se ejecuta sin tu aprobación, y nada se integra sin que lo hagas tú.
- **La revisión es independiente de verdad.** El revisor no ve el razonamiento del que programó: solo la orden, los cambios y los tests.
- **Queda todo registrado.** Qué se pidió, qué se hizo, qué dijeron los tests y qué opinó el revisor, con una "huella digital" (hash SHA-256) de cada pieza.

## 3. Las cinco ideas que lo guían

1. **Una IA no coordina a otras IAs.** Enviar, esperar, reenviar y decidir si se sigue son tareas mecánicas. Las hace un script con una tabla fija, no otra IA que pueda equivocarse o dejarse convencer.
2. **La autoridad la ponen los permisos, no las palabras.** Pedirle a una IA "no toques esto" no basta. El orquestador le **quita la herramienta**: el revisor no puede escribir; el que programa no puede hacer commit ni escribir fuera de su copia.
3. **Ante la duda, parar.** Si algo no está claro, el resultado es STOP y decides tú. Es preferible parar de más que aprobar de más.
4. **Evidencia, no promesas.** Los tests los ejecuta el orquestador, no la IA. Lo que la IA *dice* que ha hecho no cuenta como prueba.
5. **PASS no significa "intégralo".** Significa "el candidato cumple lo que pedía la orden". Mirarlo e integrarlo sigue siendo trabajo tuyo.

## 4. Las piezas

| Pieza | Qué es | Dónde vive |
|---|---|---|
| **Packet** | La orden de trabajo escrita: qué hay que hacer, qué se puede tocar, qué está prohibido, cómo se sabe que está bien y cuándo parar. | `handoffs\<id>\packet.md`, dentro de tu proyecto |
| **Configuración** | Cómo se prepara y se prueba tu proyecto (`npm ci`, `npm test`…), tiempos máximos y modelo. | `orquestador.config.json` (o la que pases con `--config`) |
| **Worktree** | Una copia aislada de tu repositorio, en una carpeta aparte, donde trabajan las IAs. Tu copia principal no se toca. | `..\<repo>-worktrees\<id>\` |
| **Implementador** | La IA que hace el trabajo (Claude o Codex), con permiso para editar solo dentro del worktree. | — |
| **Revisor** | La otra IA, en **solo lectura**, que decide si el trabajo cumple el packet. | — |
| **Investigador** | Una IA en solo lectura que responde a una pregunta sobre el código. | — |
| **Informe** | `resultado.md`, con el estado final, las huellas y qué hacer después; y `manifiesto.sha256`, con la huella de todos los archivos. | `..\<repo>-orquestador-runs\<id>\` |

Si desarrollas el orquestador, tendrás **dos copias del script**: el clon donde desarrollas y la copia que usas (por ejemplo, `C:\herramientas\orquestador\orquestar.mjs`).

## 5. Los tres modos

| Modo | Para qué | Quién trabaja | ¿Escribe algo? |
|---|---|---|---|
| **implementar** | Hacer un cambio pequeño y bien definido. | Implementador → tests → revisor | Solo en el worktree, en una rama `handoff/<id>` |
| **revisar** | Opinar sobre un cambio que ya existe (por ejemplo, una PR). | Solo el revisor | No |
| **investigar** | Responder una pregunta sobre el código ("¿qué cubren estos tests?"). | Solo el investigador | No |

## 6. Cómo trabaja, paso a paso (modo implementar)

```
packet aprobado
   │
   ▼
1. Comprobaciones previas ── algo mal ──► STOP (no se crea nada)
   │
   ▼
2. Crea el worktree aislado (rama handoff/<id>) y prepara el proyecto (npm ci…)
   │
   ▼
3. IMPLEMENTADOR hace el trabajo ─────────────────────────┐
   │                                                       │
   ▼                                                       │
4. Controles automáticos:                                  │
   - ¿tocó .claude/ o .codex/?        → STOP               │
   - ¿dijo "STOP: …"?                 → STOP               │
   - ¿hizo commit por su cuenta?      → STOP               │
   - ¿no cambió nada?                 → STOP               │
   │                                                       │
   ▼                                                       │
5. El orquestador ejecuta los TESTS                        │
   - fallan y quedan intentos ──► vuelve al paso 3 ────────┤
   - fallan y no quedan ────────► FAIL                     │
   │                                                       │
   ▼                                                       │
6. REVISOR (solo lectura) emite un veredicto               │
   - PASS             ──► PASS                             │
   - FAIL_RETRYABLE   ──► vuelve al paso 3 si quedan ──────┘
   - FAIL_STOP        ──► STOP
   - INCONCLUSIVE     ──► STOP
   │
   ▼
7. Escribe resultado.md y manifiesto.sha256. Fin.
```

Detalles importantes:

- **Paso 1:** el baseline debe ser un SHA completo (40 caracteres); el id no puede estar usado; las carpetas de trabajo deben estar fuera del repositorio.
- **Paso 5:** si los tests fallan, el revisor ni siquiera se llama: no tiene sentido revisar algo que no funciona.
- **Intentos:** `max_iteraciones` (por defecto, 2) limita cuántas veces vuelve el trabajo al implementador.
- **Tabla de veredictos:** la decisión la toma esta tabla, no una IA. Puedes usar tu propio vocabulario con `verdict_map` (por ejemplo `SATISFIED=pass, NOT_SATISFIED=retry, AUTHORITY_GAP=stop, INCONCLUSIVE=stop`).
- **Ctrl+C:** una pulsación detiene a los agentes y deja el informe completo; dos pulsaciones salen sin informe.

En **modo revisar** el flujo es más corto: comprueba que el candidato no traiga configuración de agentes (`.claude/`, `.codex/`), saca el diff, ejecuta los tests si hay, y el revisor da su veredicto. En **modo investigar**, el investigador responde y el estado es TERMINADA (o STOP si se cumple una condición de parada).

## 7. Los cuatro resultados posibles

| Estado | Qué significa | Qué haces tú |
|---|---|---|
| **PASS** | El revisor independiente aprobó el candidato según el packet. | Revisas el diff y, si te convence, integras tú (el informe trae los comandos). |
| **FAIL** | Se agotaron los intentos (tests en rojo o defectos sin corregir). | Lees por qué, ajustas el packet o lo haces de otra forma. |
| **STOP** | Hace falta una decisión humana: algo prohibido, una duda, un control de seguridad, un error… | Lees el motivo y decides. Nunca lo "fuerces". |
| **TERMINADA** | (Solo investigar.) La investigación está hecha. | Lees `inv-investigador.md`. |

Código de salida del script: `0` PASS o TERMINADA · `1` FAIL · `2` STOP · `3` error de uso.

## 8. Qué protege y qué no

**Lo que impone el orquestador (no depende de que la IA obedezca):**

- Nadie ejecuta un packet sin tu aprobación (id + primeros 8 caracteres de su hash).
- Las IAs trabajan en un worktree aislado; al final se comprueba que tu repositorio principal quedó intacto.
- El implementador Claude solo puede editar dentro del worktree (`Edit(/**)`) y tiene prohibidos commit, push, reset, checkout, merge y la web.
- El revisor y el investigador no pueden escribir.
- Codex implementa en un sandbox sin red y sin carpetas extra con escritura.
- No se carga tu configuración personal de Claude ni los servidores MCP, plugins o búsqueda web de Codex.
- Si una IA deja configuración de agentes (`.claude/`, `.codex/`) para la sesión siguiente, STOP. En modo revisar, si el candidato la trae, STOP antes de lanzar nada.
- Hay tiempo máximo para cada fase, y al vencer se matan los procesos.

**Lo que NO protege (limitaciones honestas; la lista completa está en [Limitaciones conocidas](../README.md#limitaciones-conocidas)):**

- **El código del proyecto se ejecuta en tu máquina sin sandbox.** La preparación (`npm ci`) y los tests (`npm test`) los lanza el orquestador con tus permisos. Si el código es malicioso, puede hacer daño. **No lo uses con código en el que no confíes sin mirarlo antes.**
- **Las IAs pueden leer cualquier archivo de tu usuario.** No tienen red, pero el código del candidato sí la tiene cuando se ejecuta.
- **En Windows, el implementador Claude no tiene aislamiento del sistema operativo**, solo reglas de permisos.
- El implementador puede tocar tests o `CLAUDE.md`/`AGENTS.md` si el packet no lo prohíbe y el revisor no lo detecta.
- Una IA puede equivocarse al revisar. Por eso PASS no es "integra".

## 9. Cómo usarlo bien

### 9.1. Cuándo sí y cuándo no

**Úsalo** cuando el trabajo necesite una **puerta**: revisión independiente, evidencia guardada o una aprobación antes de integrar. Por ejemplo, un paso concreto del plan de tu proyecto o revisar una PR.

**No lo uses** para:

- Preguntas rápidas o conversación: habla directamente con Claude o Codex.
- Tareas grandes, difusas o de diseño: primero hay que decidir qué se quiere.
- Muchas tareas en paralelo o investigación con muchas fuentes: mejor los workflows de Claude Code (`/deep-research`).

### 9.2. Cómo escribir un buen packet

El packet es lo más importante: **la IA solo puede hacerlo tan bien como esté escrito.** Ningún revisor detecta un error si nunca se dijo qué se quería.

- **OBJECTIVE:** una o dos frases. Una sola cosa.
- **SCOPE:** lista concreta de archivos o carpetas que se pueden tocar. Cuanto más estrecho, mejor.
- **OUT_OF_SCOPE / PROHIBITED:** lo que no se toca aunque parezca relacionado (dependencias, migraciones, tests, documentos canónicos…).
- **SUCCESS_CRITERIA:** criterios **comprobables uno a uno**. "npm test pasa", "1500 minutos devuelve 25:00". Evita "que quede bien".
- **STOP_CONDITIONS:** cuándo debe parar en vez de improvisar.
- **Sé honesto con lo que la IA no puede hacer.** No tiene web: no le pidas que compruebe notas de versión; pídele que liste qué debes comprobar tú.

### 9.3. Buenas costumbres

1. **Tareas pequeñas.** Un packet, un paso. Si el packet crece mucho, divídelo.
2. **Cruza los motores.** Si Codex implementa, que revise Claude, o al revés. Un modelo comparte puntos ciegos consigo mismo.
3. **Pocas iteraciones.** Deja `max_iteraciones: 2`. Si a la segunda no está, el packet probablemente está mal planteado.
4. **Siempre `--dry-run` primero.** Comprueba todo sin crear nada y te dice el SHA si pusiste `main`.
5. **Aprueba con el hash que calculas tú**, no con el que te dice un agente: `Get-FileHash`.
6. **Elige bien la configuración.** Para tu propio código, la normal del proyecto. Para código ajeno (PRs de terceros, Dependabot), una sin preparación ni tests o con `npm ci --ignore-scripts`.
7. **Nunca cambies de rama en tu repositorio principal** para preparar un packet. Para obtener SHA: `git fetch` y `git rev-parse`.
8. **Lee el informe entero**, también con PASS: los hallazgos MENOR suelen traer las comprobaciones pendientes.
9. **Limpia** el worktree cuando acabes; guarda la carpeta de resultados como registro.

## 10. Ejemplos

### Ejemplo 1 (real): revisar una PR de Dependabot

**Situación:** en un proyecto con CI en GitHub, Dependabot propone una PR para actualizar `actions/setup-node` de 6.4.0 a 7.0.0. ¿Es segura?

**Por qué el orquestador:** queremos una revisión independiente y que quede registrada, sin ejecutar código ajeno.

1. Obtener los SHA sin cambiar de rama (`<n>` es el número de la PR):

   ```powershell
   cd C:\proyectos\mi-proyecto
   git fetch origin
   git fetch origin pull/<n>/head
   $cand = (git rev-parse FETCH_HEAD).Trim()
   $base = (git merge-base origin/main $cand).Trim()
   git diff --stat $base $cand        # solo debe salir .github/workflows/ci.yml
   ```

   El baseline es el punto de partida de la PR (`merge-base`), no el `main` actual: así el diff solo contiene lo de la PR.

2. Configuración sin preparación ni tests (`revisar.config.json`):

   ```json
   { "comandoPreparacion": null, "comandoTests": null, "timeoutMinutos": { "revisor": 20 } }
   ```

3. Packet (cabecera y lo esencial):

   ```markdown
   ---
   id: "R001-revisar-setup-node"
   modo: revisar
   baseline: <SHA de $base>
   candidato: <SHA de $cand>
   revisor: claude
   ---
   # OBJECTIVE
   Evaluar si la PR de Dependabot (setup-node 6.4.0 → 7.0.0) es segura de integrar.
   # SCOPE
   - Revisar únicamente el diff entre baseline y candidato.
   # SUCCESS_CRITERIA
   - Solo cambia setup-node, con pin por SHA completo y comentario de versión.
   - No hay permisos nuevos ni cambios en otros jobs.
   - No tienes web: enumera qué debe comprobar el humano en las notas de la versión.
   # STOP_CONDITIONS
   - El diff toca algo más que la versión de setup-node.
   ```

4. Validar, aprobar y lanzar:

   ```powershell
   $O = "C:\herramientas\orquestador\orquestar.mjs"
   $C = "C:\herramientas\orquestador\revisar.config.json"
   node $O handoffs\R001\packet.md --config $C --dry-run
   (Get-FileHash handoffs\R001\packet.md -Algorithm SHA256).Hash.Substring(0,8)
   # → apruebas en la conversación: "Apruebo R001-revisar-setup-node <8 caracteres del hash>"
   node $O handoffs\R001\packet.md --config $C
   ```

5. **Resultado real:** PASS en unos 40 segundos. El revisor comprobó las 5 líneas cambiadas, los pines y los permisos, y listó tres comprobaciones externas: que el SHA corresponde a la etiqueta v7.0.0, los cambios incompatibles de la versión y la CI de la PR. Se hicieron las tres y la PR se integró desde GitHub.

### Ejemplo 2: implementar un cambio pequeño en tu propio código

**Situación:** quieres añadir una validación a una función de tu proyecto.

```markdown
---
id: "R-20261030-validar-duracion"
modo: implementar
baseline: <git rev-parse origin/main, SHA completo>
implementador: codex
revisor: claude
max_iteraciones: 2
commit_policy: none
---
# PROJECT
Tu proyecto (C:\proyectos\mi-proyecto)

# AUTHORITY
- Decisión del responsable del proyecto del 30-10-2026. Este packet es la conformidad previa para los cambios del SCOPE.

# OBJECTIVE
Hacer que calcularDuracion() lance RangeError cuando la salida es anterior a la entrada.

# SCOPE
- src/lib/duracion.ts
- src/lib/duracion.test.ts (solo añadir tests nuevos)

# OUT_OF_SCOPE
- Cualquier otro archivo, y el canon documental (docs/plan.md, docs/DECISIONS.md, AGENTS.md).

# PROHIBITED
- Cambiar dependencias.
- Modificar o borrar tests existentes.

# INPUTS
- AGENTS.md
- src/lib/duracion.ts

# SUCCESS_CRITERIA
- npm run lint, npm run typecheck y npm test en verde (los ejecuta el orquestador).
- Salida anterior a entrada → RangeError, con un test nuevo que lo demuestra.
- El comportamiento con datos válidos no cambia.

# STOP_CONDITIONS
- Hace falta tocar otro archivo o decidir algo que el packet no resuelve.
- Un test existente deja de pasar.

# OUTPUT_REQUIRED
- Archivos tocados, comprobaciones hechas, dudas abiertas.
```

Se lanza con la configuración normal del proyecto (`orquestador.config.json`, con `npm ci` y los tests). Es tu propio código, así que ejecutarlo es aceptable. Con PASS, el cambio queda sin commit en el worktree; lo revisas y lo integras tú con los comandos de `resultado.md`.

### Ejemplo 3: investigar antes de decidir

**Situación:** antes de escribir el packet del ejemplo 2, quieres saber qué cubren los tests actuales.

```markdown
---
id: "R-20261029-inv-tests-duracion"
modo: investigar
baseline: <SHA completo>
investigador: claude
---
# OBJECTIVE
Describir qué casos cubren los tests de src/lib/duracion.test.ts y qué casos relevantes no están cubiertos.

# STOP_CONDITIONS
- Responder exige modificar archivos.

# OUTPUT_REQUIRED
- Lista de casos cubiertos y no cubiertos, con archivo y línea.
```

No toca nada. La respuesta (en `inv-investigador.md`) te sirve para escribir mejores SUCCESS_CRITERIA en el packet siguiente.

## 11. Una ejecución completa, de principio a fin

1. **Prepara** el packet en `handoffs\<id>\packet.md` dentro del proyecto. Comprueba que `handoffs/` está en `.git\info\exclude`.
2. **Valida** con `--dry-run`: debe terminar en "todo en orden".
3. **Calcula el hash** con `Get-FileHash` y **aprueba** escribiendo el id y los 8 primeros caracteres.
4. **Lanza** el mismo comando sin `--dry-run`. Puedes dejarlo trabajar; cada minuto avisa de que sigue vivo.
5. **Lee** `resultado.md` y la respuesta del revisor (`iterN-revisor.md` o `rev-revisor.md`).
6. **Decide:** integra (PASS), corrige el packet (FAIL) o resuelve lo que motivó la parada (STOP).
7. **Si quieres preguntar** a un agente por lo que hizo, `resultado.md` trae el comando para reabrir su sesión. Antes de escribir, pulsa Shift+Tab hasta ver **plan mode**.
8. **Limpia** el worktree; guarda la carpeta de resultados.

## 12. Qué deja en la carpeta de resultados

| Archivo | Contenido |
|---|---|
| `packet.md` | Copia exacta del packet usado |
| `log.md` | Qué pasó y cuándo, con los comandos lanzados |
| `iterN-*.prompt.md` | Lo que se envió a cada agente |
| `iterN-implementador.md`, `iterN-revisor.md` | Sus respuestas |
| `iterN-diff.patch`, `iterN-tests.txt` | Evidencia generada por el orquestador |
| `worktree-estado-final.txt` | Cómo quedó la copia de trabajo |
| `resultado.md` | Estado, motivo, huellas, sesiones y siguiente paso |
| `manifiesto.sha256` | Huella de todos los archivos anteriores |

## 13. Si algo falla

Mira primero [Limitaciones conocidas](../README.md#limitaciones-conocidas) en el README y, para validar tu instalación, [`pruebas/PRUEBA-WINDOWS.md`](../pruebas/PRUEBA-WINDOWS.md). Los casos más habituales:

- **"La rama handoff/<id> ya existe":** el id ya se usó. Pon otro id (por ejemplo, añade `-r`).
- **El revisor Codex para sin revisar:** créditos de Codex agotados. Pon `revisor: claude`.
- **`git push` rechazado con GH007:** el email del commit no es tu dirección noreply de GitHub.
- **Claude Code se niega a lanzar un script con `-ExecutionPolicy Bypass`:** lánzalo tú en tu terminal. No añadas permisos para saltarlo.

## 14. Glosario

- **Baseline:** el commit desde el que se parte. Siempre su SHA completo.
- **Candidato:** el resultado que se revisa (un commit o los cambios del worktree).
- **SHA:** el identificador único de un commit (40 caracteres).
- **Hash SHA-256:** huella digital de un archivo; si cambia un solo carácter, cambia la huella.
- **Worktree:** una segunda copia de trabajo del mismo repositorio, en otra carpeta.
- **Diff:** la lista de diferencias entre dos versiones.
- **CI:** las comprobaciones automáticas que GitHub ejecuta en cada PR.
- **Sandbox:** un entorno que limita lo que un programa puede hacer.
- **MCP:** conectores que dan herramientas extra a una IA (navegador, búsqueda…). El orquestador los desactiva.
