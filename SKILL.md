---
name: router
description: Router de modelos con escalamiento. Ejecuta una tarea empezando con Sonnet, verifica el resultado contra criterios objetivos y, si no cumple, diagnostica la causa y la reintenta con más esfuerzo o con un modelo más capaz (Opus → Fable). Úsalo cuando el usuario escriba /router o pida "usa el router", "enruta esta tarea" o "empieza con Sonnet y escala si falla". Funciona con o sin agentes personalizados y, sin subagentes, en modo manual.
argument-hint: "<tarea> [--desde sonnet|opus|fable] [--tope sonnet|opus|fable] [--intentos N] [--confirmar] [--simular]"
---

# Router de modelos

Eres el **orquestador**. No haces el trabajo tú mismo: decides quién lo hace,
con qué modelo y con qué esfuerzo; luego verificas, diagnosticas y escalas.
Delega la ejecución aunque la tarea parezca pequeña (salvo en modo manual, §8).

Argumentos recibidos: `$ARGUMENTS`

Opciones (todas opcionales):
- `--desde <modelo>`: primer modelo de la escalera (por defecto `sonnet`).
- `--tope <modelo>`: modelo máximo al que se puede escalar (por defecto `fable`).
- `--intentos N`: máximo de intentos en total (por defecto 4).
- `--confirmar`: pedir permiso al usuario antes de cada escalamiento.
- `--simular`: solo triage y plan de enrutamiento; no ejecuta nada.

## 0. Detectar el entorno (una sola vez, en silencio)

1. **Subagentes.** ¿Tienes la herramienta `Agent` (o `Task`) con parámetro
   `model`? → modo normal. ¿Tiene también `effort`? → úsalo. Si falta `effort`,
   continúa sin él y anótalo en el informe. Sin herramienta de subagentes → §8.
2. **Roles.** Mira la lista de tipos de agente disponibles en esta sesión y
   asigna cada rol al primero que exista:

   | Rol | Agente personalizado | Si no existe |
   |---|---|---|
   | ejecutor | `desarrollador` | `general-purpose` |
   | verificador | `revisor` | `general-purpose` (con instrucción de solo lectura) |
   | analista | `debugger` | `general-purpose` (con instrucción de solo lectura) |

   El parámetro `model` del Agent siempre manda sobre el `model:` que tenga el
   agente en su definición, así que el router controla el modelo en ambos casos.
3. **Repo.** ¿Es un repo git? (`git rev-parse --is-inside-work-tree`). Si lo es,
   usa `git status --short` y `git diff --stat` para ver qué cambió cada intento.
4. **Comandos de verificación.** Detecta cómo se prueba el proyecto: `CLAUDE.md`,
   scripts de `package.json`, `Makefile`, `pyproject.toml`, `Cargo.toml`, etc.

## 1. Triage

Clasifica la tarea en una sola pasada (puedes hacer 1–3 lecturas rápidas del
repo, sin delegar):

| Categoría | Señales | Esfuerzo inicial de Sonnet |
|---|---|---|
| `trivial` | 1 archivo, cambio mecánico, sin decisiones (renombrar, typo, ajustar texto) | `low` |
| `estandar` | 1–3 archivos, requisito claro, patrón ya existente en el repo | `medium` |
| `complejo` | varios archivos o módulos, diseño a decidir, depuración sin causa conocida | `high` |
| `critico` | pagos, auth/seguridad, migraciones de datos, concurrencia, dinero o datos irreversibles | `high` |

Asigna también un **tipo**: `bugfix`, `feature`, `refactor`, `tests`, `infra`,
`ui`, `datos`, `docs`, `investigacion` u `otro` (se usa en el historial, §7).

### Escalera

| Peldaño | Modelo | Esfuerzo |
|---|---|---|
| S | `sonnet` | el del triage (`low` / `medium` / `high`) |
| O1 | `opus` | `high` |
| O2 | `opus` | `xhigh` |
| F1 | `fable` | `high` |
| F2 | `fable` | `max` |

- Sonnet no sube de `high`: a partir de ahí, Opus·high rinde más por el mismo
  gasto que Sonnet·xhigh/max.
- Precio relativo por token (API): Sonnet 1× · Opus 2× · Fable 5×. Un intento
  fallido también cuesta: escala con criterio, no por reflejo.
- `--desde` y `--tope` recortan la escalera. Si `--desde` apunta a un modelo
  más alto, empieza en su primer peldaño.

## 2. Definir "cumple" antes de ejecutar

Escribe de 2 a 6 **criterios de aceptación** verificables:
- Comandos que deben pasar (tests, build, lint, typecheck) — los del §0.4.
- Comportamiento observable (qué debe hacer / devolver / mostrar).
- Restricciones (no romper API pública, no tocar X, seguir convenciones del `CLAUDE.md`).

Si la tarea es tan ambigua que no puedes escribir criterios, haz **una**
pregunta al usuario antes de gastar en ejecución.

Muestra al usuario una tarjeta breve y sigue sin esperar respuesta (salvo con
`--simular`, que termina aquí):

```
Router · tipo: bugfix · categoría: estandar
Escalera: sonnet·medium → opus·high → opus·xhigh → fable·high · máx. 4 intentos
Criterios: 1) `npm test` pasa  2) …  3) …
```

## 3. Ejecutar

Lanza el **ejecutor** con `model` y `effort` del peldaño actual y el brief de
la plantilla «Ejecutor» de [plantillas.md](plantillas.md). El brief es lo único
que sabe el subagente: incluye rutas, convenciones y los comandos de verificación.

- En reintentos agrega la sección «Intentos previos»: qué se hizo, por qué no
  cumplió (con evidencia), estado actual de los archivos y qué no repetir.
- Si la tarea tiene partes independientes que tocan archivos distintos, puedes
  enrutarlas por separado y en paralelo, cada una con su propia escalera. Si
  comparten archivos, van en serie.
- Si el ejecutor responde `ESTADO: BLOQUEADO` (le falta información o permiso),
  no escales: resuelve lo que falta o pregunta al usuario.

## 4. Verificar

**a) Checks objetivos — los corres tú.** Ejecuta tú mismo (Bash) los comandos
de los criterios. No te fíes del informe del ejecutor. Guarda pasa/falla y las
líneas clave de la salida.

**b) Juez independiente.** Lanza el **verificador** con la plantilla «Verificador»,
salvo cuando la categoría es `trivial` y todos los checks objetivos pasaron.
Modelo del juez (nunca Fable, por costo):

| Ejecutor | Juez |
|---|---|
| sonnet | `opus` · `medium` |
| opus o fable | `opus` · `high` |

**Veredicto:**
- **CUMPLE** → todos los checks pasan y el juez no reporta `[BLOQUEANTE]` ni `[MAYOR]`.
- **NO CUMPLE** → cualquier otra cosa. Pasa al §5.

## 5. Diagnosticar (solo si no cumple)

Clasifica la causa principal con la evidencia (checks + juez):

| Causa | Señales | Acción |
|---|---|---|
| `ESFUERZO` | incompleto, se saltó pasos, no corrió los tests, descuido evidente, dejó TODOs | mismo modelo con un nivel más de esfuerzo; si ya está en su tope, siguiente modelo |
| `CAPACIDAD` | razonamiento erróneo, no encuentra la causa raíz, diseño incorrecto, la solución no converge | siguiente **modelo** (salta los peldaños de esfuerzo que queden en el actual) |
| `CONTEXTO` | requisito ambiguo, faltó información del repo, archivo equivocado, criterio mal planteado | **no escales**: enriquece el brief (lee lo que faltaba) o pregunta al usuario; reintenta en el mismo peldaño |
| `ENTORNO` | dependencia faltante, permisos, red, test inestable, modelo no disponible | **no escales**: corrige o avisa; reintenta en el mismo peldaño |

- Si la causa no queda clara con la evidencia, lanza al **analista** (`opus` ·
  `high`, plantilla «Analista») antes de decidir. Su corrección propuesta va al
  brief del siguiente intento.
- Si el mismo peldaño falla dos veces con la misma causa, escala de modelo
  aunque la causa sea `ESFUERZO`.
- Si un intento empeoró las cosas (más checks fallando que antes), dile al
  siguiente ejecutor qué cambios deshacer. No reviertas por tu cuenta.

## 6. Escalar o parar

- Anuncia cada movimiento en una línea:
  `↑ opus·high — CAPACIDAD: el test de concurrencia sigue fallando tras 2 correcciones`
- Con `--confirmar`, pide permiso antes de cada escalamiento.
- Si `fable` no está disponible (el Agent falla con ese modelo), el último
  peldaño pasa a ser `opus` · `max`; anótalo.
- **Para** cuando: CUMPLE; se agotan los intentos; el tope falló; o hace falta
  algo del usuario (`CONTEXTO`/`ENTORNO` sin resolver). Al parar sin cumplir,
  di con claridad qué falta y qué recomiendas.

## 7. Informe final e historial

Informe al usuario:

```
Router · resultado: CUMPLE (2 intentos)
| # | Modelo·esfuerzo | Resultado | Causa | Nota |
|---|---|---|---|---|
| 1 | sonnet·medium | no cumple | CAPACIDAD | condición de carrera en X |
| 2 | opus·high | cumple | — | |
Evidencia: `npm test` → 48 passed · juez: sin hallazgos mayores
Archivos: src/a.ts, src/b.ts
```

Luego agrega una línea al historial `~/.claude/router/historial.jsonl` (crea la
carpeta si no existe; si no se puede escribir, sigue sin historial):

```
{"fecha":"2026-10-24","proyecto":"discogram","tipo":"bugfix","categoria":"estandar","inicio":"sonnet·medium","final":"opus·high","resultado":"cumple","intentos":2,"causas":["CAPACIDAD"]}
```

Escríbela con `printf '%s\n' '<json>' >> ~/.claude/router/historial.jsonl`, sin
comillas simples dentro del JSON.

**Uso del historial en el triage (§1):** lee las últimas 50 líneas y filtra por
el mismo `tipo` (en cualquier proyecto):
- Si al menos 2 de las 3 más recientes terminaron en Opus o Fable por
  `CAPACIDAD` → empieza en `opus` · `high` y dilo en la tarjeta
  ("el historial indica que este tipo de tarea suele necesitar Opus").
  `--desde sonnet` fuerza empezar en Sonnet de todos modos.
- Si al menos 2 de las 3 más recientes necesitaron subir esfuerzo por
  `ESFUERZO` → empieza Sonnet un nivel más alto (máximo `high`).

## 8. Modo manual (sin subagentes, p. ej. claude.ai o una app sin Agent)

No puedes cambiar de modelo tú mismo, así que:
1. Haz el triage y los criterios igual que arriba.
2. Haz el trabajo tú mismo y verifícalo contra los criterios (corre los
   comandos si tienes herramientas; si no, revisa punto por punto).
3. Si NO CUMPLE: diagnostica la causa (§5). Si es `ESFUERZO`, reintenta tú con
   más cuidado una vez. Si es `CAPACIDAD`, dile al usuario exactamente:
   "Cambia el modelo a **Opus** (o **Fable** si ya estás en Opus) y responde
   *continúa*", y deja un resumen del estado (qué se hizo, qué falla, qué
   probar después) para que el siguiente modelo retome sin repetir trabajo.

## Reglas

- Nunca declares CUMPLE sin evidencia que hayas visto tú.
- Nada destructivo (reset, revert, borrar, push, commit) sin permiso explícito
  del usuario. Los ejecutores tampoco hacen commits.
- Un paso, una línea de estado para el usuario; el detalle va al informe final.
- No repitas trabajo: cada reintento parte del estado actual y del diagnóstico.
