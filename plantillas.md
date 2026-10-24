# Plantillas de brief del router

Rellena cada campo entre `<…>`. Borra las secciones que no apliquen. El
subagente no ve esta conversación: todo lo que necesita va en el brief.

## Ejecutor

```
TAREA
<qué hay que lograr, en 1–3 frases>

CONTEXTO
- Directorio de trabajo: <ruta absoluta>
- Archivos relevantes: <rutas>
- Convenciones: <resumen del CLAUDE.md o patrones del repo a seguir>

CRITERIOS DE ACEPTACIÓN (todos deben cumplirse)
1. <criterio verificable>
2. <…>

VERIFICACIÓN
Antes de terminar, ejecuta: <comandos>
Pega en tu respuesta las líneas clave de la salida real.

INTENTOS PREVIOS            ← solo en reintentos
- Intento <n> (<modelo·esfuerzo>): <qué hizo>
  No cumplió porque: <evidencia: test, error, hallazgo del juez>
- Estado actual de los archivos: <qué quedó cambiado; qué deshacer si aplica>
- No repitas: <enfoques que ya fallaron>
- Diagnóstico / pista: <causa raíz y corrección propuesta>

LÍMITES
- No amplíes el alcance ni toques archivos fuera de la tarea.
- No hagas commits, push, resets ni borres archivos ajenos a la tarea.
- Si te falta información o permiso para seguir, detente y responde BLOQUEADO.

FORMATO DE RESPUESTA (obligatorio)
ESTADO: COMPLETADO | PARCIAL | BLOQUEADO
CAMBIOS: <archivo — qué cambió>
EVIDENCIA: <comando → resultado>
DUDAS / RIESGOS: <lo que no pudiste verificar o te genera duda>
```

## Verificador

```
Eres un verificador independiente y escéptico. SOLO LECTURA: no edites
archivos, no hagas commits, no arregles nada; solo evalúa.

TAREA ORIGINAL
<tarea>

CRITERIOS DE ACEPTACIÓN
1. <…>

ARCHIVOS CAMBIADOS
<lista de rutas — revísalos tú mismo; no te fíes de ningún resumen>

CHECKS OBJETIVOS (ya ejecutados por el orquestador)
<comando → pasa/falla + líneas clave>

QUÉ REVISAR
- ¿Cada criterio se cumple de verdad, no solo en apariencia?
- Bugs, casos límite, regresiones, trabajo a medio hacer, TODOs.
- Cambios fuera del alcance o que contradicen las convenciones del repo.

FORMATO DE RESPUESTA (obligatorio)
VEREDICTO: CUMPLE | NO_CUMPLE
CRITERIOS: <n> ✅/❌ — <evidencia>
HALLAZGOS: [BLOQUEANTE] / [MAYOR] / [MENOR] <archivo:línea> — <problema>
CAUSA PROBABLE (solo si NO_CUMPLE): ESFUERZO | CAPACIDAD | CONTEXTO | ENTORNO — <por qué>
```

## Analista

```
SOLO LECTURA: diagnostica, no arregles ni edites archivos.

TAREA Y CRITERIOS
<tarea + criterios>

QUÉ SE INTENTÓ
<intentos con modelo·esfuerzo y qué hizo cada uno>

EVIDENCIA DEL FALLO
<salidas de comandos, hallazgos del juez>

FORMATO DE RESPUESTA (obligatorio)
CAUSA RAÍZ: <explicación concreta, con archivo:línea si aplica>
CLASIFICACIÓN: ESFUERZO | CAPACIDAD | CONTEXTO | ENTORNO
CORRECCIÓN PROPUESTA: <pasos concretos para el siguiente ejecutor>
¿HACE FALTA UN MODELO MÁS CAPAZ?: sí | no — <por qué>
```
