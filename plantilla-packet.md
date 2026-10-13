---
id: "001"                 # único por handoff; da nombre a la rama handoff/<id> y a la carpeta de resultados
modo: implementar         # implementar | revisar | investigar
baseline: PEGAR_SHA_40    # SHA completo (git rev-parse main). Referencias como main o HEAD solo se admiten en --dry-run
candidato:                # solo en modo revisar: SHA completo del commit a revisar
implementador: claude     # claude | codex
revisor: codex            # claude | codex  (cruzado = más independiente)
investigador:             # solo en modo investigar: claude | codex
max_iteraciones: 2        # ciclos implementar→revisar antes de FAIL
commit_policy: none       # none (candidato sin commit; lo haces tú) | orchestrator (commit por iteración en la rama handoff/<id>)
verdict_map:              # vacío = PASS=pass, FAIL_RETRYABLE=retry, FAIL_STOP=stop, INCONCLUSIVE=stop
                          # ejemplo con vocabulario propio: SATISFIED=pass, NOT_SATISFIED=retry, AUTHORITY_GAP=stop, INCONCLUSIVE=stop
---

# PROJECT
Nombre del proyecto

# AUTHORITY
Qué decisión humana autoriza esto (fecha, dónde se adoptó) y documentos con autoridad.

# OBJECTIVE
Qué debe conseguirse, en una o dos frases.

# SCOPE
- Qué está autorizado tocar (archivos, módulos, comportamientos).

# OUT_OF_SCOPE
- Lo que no entra aunque parezca relacionado.

# PROHIBITED
- Cambiar dependencias, migraciones de base de datos, configuración de producción…
- Desactivar o borrar tests.

# INPUTS
- Archivos o documentos que el agente debe leer primero (los listados aquí tienen autoridad).

# SUCCESS_CRITERIA
- Criterios verificables uno a uno (tests que deben pasar, comportamiento observable).

# STOP_CONDITIONS
- Falta un archivo requerido o el código no coincide con lo descrito.
- Cumplir el objetivo exige salir del SCOPE.
- Aparece una decisión de diseño o de negocio no resuelta aquí.
- Un test existente deja de pasar y arreglarlo no está en el SCOPE.

# OUTPUT_REQUIRED
- Resumen de cambios, archivos tocados, comprobaciones hechas, dudas abiertas.
