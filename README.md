# Router de modelos (skill para Claude Code)

Ejecuta una tarea empezando con **Sonnet**, la verifica contra criterios objetivos
y, si no cumple, **diagnostica por qué** antes de decidir: subir el esfuerzo,
pasar a un modelo más capaz (**Opus → Fable**) o pedirte lo que falta.

```
triage → criterios → ejecutor (sonnet·low/medium/high)
                       ↓
             checks objetivos + juez (opus)
                       ↓
         ¿cumple? ── sí → informe + historial
             │ no
             ↓
   diagnóstico: ESFUERZO → mismo modelo, más esfuerzo
                CAPACIDAD → siguiente modelo
                CONTEXTO / ENTORNO → no escala; corrige o pregunta
```

Escalera por defecto: `sonnet·(según triage)` → `opus·high` → `opus·xhigh` →
`fable·high` → `fable·max`, con un máximo de 4 intentos.

## Uso

```
/router corrige el bug del paginador en la vista de discos
/router --tope opus agrega validación al formulario de registro
/router --simular migra la tabla de usuarios a la nueva estructura
```

| Opción | Efecto |
|---|---|
| `--desde sonnet\|opus\|fable` | modelo con el que empieza (por defecto `sonnet`) |
| `--tope sonnet\|opus\|fable` | modelo máximo al que puede escalar (por defecto `fable`) |
| `--intentos N` | máximo de intentos (por defecto 4) |
| `--confirmar` | te pide permiso antes de cada escalamiento |
| `--simular` | solo muestra triage, criterios y escalera; no ejecuta |

También se activa si le dices "usa el router" o "enruta esta tarea".

## Funciona con o sin agentes

- **Con tus agentes** (`desarrollador`, `revisor`, `debugger`): los usa como
  ejecutor, juez y analista, y les fija el modelo y el esfuerzo de cada intento.
- **Sin ellos**: usa el agente integrado `general-purpose` con las plantillas de
  `plantillas.md`.
- **Sin subagentes** (claude.ai, apps sin herramienta Agent): modo manual; hace
  el trabajo, lo verifica y, si hace falta un modelo mayor, te dice que lo cambies
  y deja el estado resumido para continuar.

## Historial

Cada ejecución agrega una línea a `~/.claude/router/historial.jsonl`. En el triage
lo lee: si un tipo de tarea (bugfix, feature, ui…) viene necesitando Opus, el
router empieza ahí directamente; si viene necesitando más esfuerzo, sube el
esfuerzo inicial de Sonnet. El historial es local de cada equipo.

## Instalar en otro dispositivo

La skill es solo esta carpeta (`SKILL.md`, `plantillas.md`, `README.md`).

**Copiando la carpeta** (Linux / macOS):

```bash
scp -r ~/.claude/skills/router usuario@otro-equipo:~/.claude/skills/
```

En Windows la ruta es `%USERPROFILE%\.claude\skills\router`.

**Con git** (recomendado si la vas a editar): crea un repo privado con esta
carpeta y en cada equipo:

```bash
git clone <url-del-repo> ~/.claude/skills/router
```

Para actualizar después: `git -C ~/.claude/skills/router pull`.

**En claude.ai**: comprime la carpeta en un .zip y súbela en
Configuración → Capacidades → Skills. Ahí corre en modo manual.

Requisitos: una versión de Claude Code cuya herramienta Agent acepte `model`
(con `fable`) y `effort`. Si falta `effort`, el router sigue sin él; si `fable`
no está disponible, el último peldaño pasa a ser `opus·max`.

## Personalizar

Todo está en `SKILL.md`:
- §1: categorías del triage y la escalera (modelos y esfuerzos).
- §4: modelo del juez.
- §5: tabla de causas y qué acción toma cada una.
- §7: reglas de uso del historial.
