# Plan Maestro — Dashboard Institucional de Movilidad Estudiantil DRI

> **Documento de referencia única (Single Source of Truth) del proyecto.**
> Toda decisión de diseño, arquitectura, modelado o UX que no esté aquí, no se ha tomado oficialmente.
> Si tenés dudas sobre por qué algo se hizo de cierta forma, este es el documento al que volver.

| Campo | Valor |
|---|---|
| Proyecto | Rediseño dashboard Movilidad Estudiantil |
| Cliente interno | Dirección de Relaciones Internacionales (DRI) |
| Arquitecto BI | [tu nombre] |
| Plataforma | Power BI (formato `.pbip`) |
| Versionado | Git |
| Versión del documento | v1.2 |
| Última actualización | 2026-05-22 |
| Estado | Fase 1 cerrada · listo para Fase 2 |

---

## Tabla de Contenidos

1. [Contexto del proyecto](#1-contexto-del-proyecto)
2. [Audiencias y necesidades](#2-audiencias-y-necesidades)
3. [Preguntas institucionales priorizadas](#3-preguntas-institucionales-priorizadas)
4. [Modelo de datos](#4-modelo-de-datos)
5. [Principios de arquitectura](#5-principios-de-arquitectura)
6. [Marco de decisión de filtros](#6-marco-de-decisión-de-filtros)
7. [Decisiones arquitectónicas tomadas](#7-decisiones-arquitectónicas-tomadas)
8. [Arquitectura de páginas](#8-arquitectura-de-páginas)
9. [Anti-patrones a rechazar](#9-anti-patrones-a-rechazar)
10. [Pendientes y riesgos abiertos](#10-pendientes-y-riesgos-abiertos)
11. [Estándares operativos](#11-estándares-operativos)
12. [Plan de implementación por fases](#12-plan-de-implementación-por-fases)
13. [Convenciones de versionado git](#13-convenciones-de-versionado-git)
14. [Metodología de trabajo del plan](#14-metodología-de-trabajo-del-plan)
15. [Glosario](#15-glosario)

---

## 1. Contexto del proyecto

### 1.1 Punto de partida

Existe un dashboard de movilidad estudiantil con dos páginas:

- **Cobertura Internacional** — funciona, se mantiene sin cambios. Página de exploración geográfica.
- **Balance de Movilidad** — será reemplazada completamente. Hoy mezcla niveles analíticos (ejecutivo, operativo, institucional) en una sola página.

### 1.2 Objetivo del rediseño

El dashboard debe convertirse en la **fuente oficial de verdad institucional** para cuatro usos simultáneos:

1. Visualización ejecutiva (directivos DRI).
2. Análisis institucional (analistas).
3. Soporte operativo (coordinadores).
4. Reporte normativo SNIES.

### 1.3 Restricciones de diseño institucional (no negociables)

- Logo institucional fijo (Universidad de La Sabana).
- Panel lateral de filtros estandarizado.
- Botón "Borrar filtros" siempre visible.
- Barra inferior con fuente del dato.
- Navegación con botón "Home".
- Paleta corporativa azul.

### 1.4 Lo que el dashboard actual no responde

- Movilidad virtual: subexplotada.
- Estudiantes extranjeros matriculados regularmente: invisibles.
- Visitantes extranjeros: invisibles.
- Reporte SNIES: se construye manualmente fuera del dashboard.

---

## 2. Audiencias y necesidades

| Audiencia | Necesidad primaria | Frecuencia de uso | Tolerancia a complejidad |
|---|---|---|---|
| Directivos DRI | Lectura ejecutiva rápida del estado institucional | Mensual / por reunión | Baja |
| Analistas institucionales | Análisis exploratorio, comparativos, soporte a decisión | Semanal | Media-Alta |
| Analistas SNIES | Extracción con formato normativo | Semestral | Baja (formato fijo) |
| Operadores / coordinadores | Consulta puntual, verificación de casos | Diaria | Media |

**Implicación de diseño:** ninguna página debe servir simultáneamente a directivos y operadores con el mismo visual. La densidad informativa que tolera un operador satura a un directivo.

---

## 3. Preguntas institucionales priorizadas

### 3.1 Preguntas confirmadas por la DRI (Prioridad 1)

1. Número total de movilidades y personas por año y/o semestre — estudiantes entrantes o salientes, diferenciados vía filtros.
2. Top de países y universidades, origen o destino.
3. Top de tipos de movilidad entrantes y salientes (actividades puntuales).
4. Número de movilidades por destino u origen.
5. Número de movilidades por tipos de movilidad.
6. **(Datos existentes, no explotados todavía)** $ COP financiado para movilidad entrante en estudiantes, tipos de financiadores nacionales e internacionales.

### 3.2 Preguntas estratégicas faltantes (observación del arquitecto)

La DRI hoy usa el dashboard como **herramienta de consulta**, no de decisión estratégica. Las 6 preguntas son descriptivas, ninguna pregunta de causalidad, retorno o priorización. El rediseño debe responder lo que piden, pero deja espacio para insights estratégicos no solicitados todavía:

- ¿Cuál es la balanza neta entrante vs saliente y cómo evoluciona?
- ¿Quiénes son los estudiantes regulares extranjeros y cómo evoluciona la atracción?
- ¿Qué peso tiene la modalidad virtual frente a presencial?

---

## 4. Modelo de datos

### 4.1 Tabla de hechos

**`FCT_Movilidad_Estudiantil`** — almacenamiento Import Mode (.pbip).

Construida mediante `Table.Combine` (append) de tres fuentes, discriminadas por `STR_TABLA_ORIGEN_CAL`:

| Origen | Naturaleza | Tipo de Estudiante |
|---|---|---|
| `FCT_MOVILIDAD_ESTUDIANTE` | Eventos de movilidad bidireccional | Temporal |
| `FCT_VISITANTE_EXTRANJERO` | Visitantes extranjeros | Temporal |
| `FCT_MATRICULADOS` | Matrícula regular | Regular |

### 4.2 Grano

Una fila = combinación única de:

```
NUM_DIM_PERSONA_SK
+ STR_DETALLE_ACTIVIDAD_CAL_AJT
+ STR_NOMBRE_ENTIDAD_EXTERNA_CAL_AJT
+ NUM_DIM_PERIODO_ACADEMICO_SK
+ NUM_NUMERO_MOVILIDAD_CAL
```

**Consecuencia crítica:** una misma persona en un mismo período puede tener múltiples filas. Por lo tanto:

- `Movilidades = COUNTROWS(FCT_Movilidad_Estudiantil)` — una fila = un evento de movilidad por definición de grano.
- `Personas = DISTINCTCOUNT(STR_PERSONA_ID_NK)` — NK (documento de identidad) es más defensiva que SK: identifica al individuo aunque cambien surrogates.
- `Movilidades ≥ Personas` siempre. La diferencia es real (multi-actividad), no error.
- Los dos KPIs deben mostrarse **siempre juntos**. Por separado, mienten.
- `FCT_MATRICULADOS` inyecta filas con `NUM_NUMERO_MOVILIDAD_CAL = 0` que no son eventos de movilidad. Con `COUNTROWS`, esas filas cuentan como 1. Hoy no rompe porque el panel de página las excluye (§7.4). Reevaluar en Fase 4 cuando entren matriculados al alcance.

### 4.3 Reglas de negocio aplicadas en Power Query

| Regla | Implementación |
|---|---|
| Dirección | `STR_CLASIFICACION_MOVILIDAD_CAL_AJT` → "Saliente" (OUTBOUND) / "Entrante" (INBOUND) |
| Tipo de Estudiante | "Regular" si origen = `FCT_MATRICULADOS`, "Temporal" en otro caso |
| Modalidad | "Virtual" si `STR_DETALLE_ACTIVIDAD_CAL_AJT` contiene "virtual", "Presencial" en otro caso |

### 4.4 Volumen y actualización

- ~15.000 registros totales.
- Datos desde 2010; foco analítico: últimos 10 períodos.
- Refresh: ~2 veces por semestre.

### 4.5 Dimensiones relacionadas

| Dimensión | Llave en FCT |
|---|---|
| Persona | `NUM_DIM_PERSONA_SK`, `STR_PERSONA_ID_NK` |
| Período Académico | `NUM_DIM_PERIODO_ACADEMICO_SK` |
| Programa Académico | `NUM_DIM_PROGRAMA_ACADEMICO_SK_CAL` |
| Ubicación Geográfica | `NUM_DIM_UBICACION_GEOGRAFICA_SK` |
| Convenio | `NUM_DIM_CONVENIO_SK_CAL` |
| Ubicación Financiación | `NUM_DIM_UBICACION_GEOGRAFICA_FINAN_SK_CAL` |

---

## 5. Principios de arquitectura

> Estos principios provienen del documento de arquitectura interno y son **no negociables**.

### 5.1 Separación de capas

**Cada filtro pertenece a una sola capa, según su rol semántico. Nunca mezcles capas en un mismo objeto.**

Antes de implementar cualquier filtro, se clasifica en uno de tres tipos. La clasificación determina la capa correcta. Si vas a desviarte, lo declarás explícitamente y justificás.

### 5.2 Estándar de documentación de medidas DAX

La documentación tiene dos destinatarios y cada uno la recibe en su capa:

- **Usuario final del reporte** (no lee DAX) → **cuadro de texto visible** en la página con el alcance documentado. Esto es no negociable.
- **Mantenedor del modelo** (lee DAX) → **bloque de comentarios estándar al inicio de cada medida**, según el formato minimalista definido abajo.

**El DAX nunca sustituye la documentación visible al usuario.** Mover lógica de alcance al DAX para "dejarla documentada" es anti-patrón.

#### 5.2.1 Estructura obligatoria de toda medida

```dax
Nombre de la Medida =
/* -----------------------------------------------------------------------------
    [PROPÓSITO]:    {Opcional: solo si el nombre NO es auto-explicativo.}
    [CASO DE USO]:  {Obligatorio: qué decisiones de negocio apoya esta métrica.}
    [LÓGICA_SIMPLE]:{Opcional: solo si la lógica DAX es compleja o contraintuitiva.}
----------------------------------------------------------------------------- */
VAR VariableDescriptiva = ...
RETURN
    VariableDescriptiva
```

#### 5.2.2 Principios del estándar

**Encabezado de negocio (bloque `/* */`):**

- **Minimalismo:** sin ruido. Si el propósito es obvio, se omite.
- `[CASO DE USO]` es el único campo obligatorio. Enfoque en utilidad analítica.
- `[PROPÓSITO]` solo cuando el nombre no es auto-explicativo.
- `[LÓGICA_SIMPLE]` solo cuando el DAX es complejo o contraintuitivo.

**Código DAX:**

- **Formato impecable:** mejores prácticas tipo DAX Formatter (espaciado, mayúsculas en funciones, indentación).
- **Variables en español, PascalCase descriptivo.**
  - ✅ `PromedioPuntaje`, `TotalEstudiantesEvaluados`
  - ❌ `_promedio`, `var1`, `AverageScore`
- **Comentarios `//`** solo si explican algo técnico no obvio dentro del DAX. No documentar lo evidente.

**Refactorización:**

- Si se detectan oportunidades de mejora (rendimiento/legibilidad) → presentar **pros y contras primero**.
- Solo refactorizar tras aprobación explícita.

#### 5.2.3 Ejemplo de aplicación

**Input (medida sin documentar):**

```dax
Promedio Competencia = AVERAGE('FCT_SP_COMP_GENERICAS'[NUM_PUNT_COMPETENCIA])
```

**Output (medida bajo estándar):**

```dax
Promedio Competencia =
/* -----------------------------------------------------------------------------
    [CASO DE USO]: Indicador fundamental de desempeño para comparar el
                   rendimiento entre cohortes o programas.
----------------------------------------------------------------------------- */
VAR PromedioPuntaje =
    AVERAGE ( 'FCT_SP_COMP_GENERICAS'[NUM_PUNT_COMPETENCIA] )
RETURN
    PromedioPuntaje
```

#### 5.2.4 Sobre la propiedad `Description` del modelo

La propiedad `Description` del modelo Power BI **no se utiliza** en este proyecto. La documentación al mantenedor vive en el bloque DAX (más cerca del código, sin cambio de panel, versionable en git). El cuadro de texto en página sigue siendo la fuente para el usuario final.

### 5.3 Portabilidad de medidas

Una medida bien diseñada es **portable**: la misma definición sirve para múltiples alcances. Si para soportar otro alcance hay que crear otra medida, la primera está mal diseñada (a menos que el filtro sea definición intrínseca del KPI).

### 5.4 KISS sobre estética

Ningún visual existe por estética. Cada visual responde una pregunta concreta de la lista priorizada. Lo que no responde nada, se elimina.

---

## 6. Marco de decisión de filtros

### 6.1 Los tres tipos de filtros

**1. Definición de negocio** — el filtro *es* parte de qué mide la métrica.
- Test: si la página se replicara con otro alcance, esta lógica seguiría aplicando idéntica.
- Capa correcta: **DAX dentro de la medida**.

**2. Alcance del reporte/página** — el filtro define qué porción del universo mira esta vista.
- Test: si mañana hay que ver el mismo KPI con otro alcance, no debería requerir una medida nueva.
- Capa correcta: **panel de filtros de página** (bloqueado y oculto), con cuadro de texto visible.

**3. Exploración del usuario** — existe para que el usuario corte y compare.
- Capa correcta: **slicer** en el lienzo.

### 6.2 Test de portabilidad (obligatorio antes de crear una medida)

1. ¿Este filtro forma parte de la definición intrínseca del KPI, o solo del alcance de esta página?
2. Si replicara esta página con otro alcance, ¿la medida tendría que cambiar?
3. ¿Estoy creando una medida nueva porque la lógica es realmente distinta, o porque estoy hardcodeando el contexto de la página?

**Regla:** si la medida es portable, el filtro **NO** va en DAX. Va al panel.

### 6.3 Formato obligatorio de justificación

Para **cada** medida y **cada** filtro:

```
Decisión: [qué se hace]
Tipo de filtro: definición / alcance / exploración
Capa elegida: DAX / panel de página / slicer / Calculation Group
Por qué acá y no en otra capa: [razonamiento]
Trade-off aceptado: [desventaja tolerable]
Test de portabilidad: [qué pasa si el alcance cambia]
```

Si no se pueden llenar los cinco campos con honestidad, la decisión está mal.

### 6.4 Cierre obligatorio por componente

Ningún componente se declara terminado sin entregar:

1. Inventario de filtros (qué filtros, dónde).
2. Justificación por capa.
3. Test de portabilidad.
4. Confirmación de cuadro de texto visible al usuario.
5. Confirmación de que todas las medidas siguen el estándar de documentación §5.2.

---

## 7. Decisiones arquitectónicas tomadas

### 7.1 Decisión: 6 páginas separadas (no 4, no 8)

- **Tipo:** estructura del producto.
- **Por qué:** cada página tiene audiencia y propósito **mutuamente excluyentes**. No hay solapamiento funcional.
- **Trade-off aceptado:** más navegación, más mantenimiento. Mitigado con menú lateral persistente agrupado por bloques.
- **Alternativas descartadas:**
  - 3 páginas con tabs/bookmarks → mezcla niveles analíticos, viola principio 5.4.
  - 8+ páginas con cortes adicionales → fragmenta la lectura ejecutiva.

### 7.2 Decisión: Movilidad Virtual como página propia, no bookmark

- **Tipo:** estructura.
- **Por qué:** decisión del usuario priorizando facilidad de mantenimiento. Un bookmark compartiendo lienzo con Presencial complica versionado y debugging.
- **Trade-off aceptado:** duplicación visual entre Balance Presencial y Virtual.
- **Mitigación:** las dos páginas comparten la misma definición de visuales y las mismas medidas base. El único diferenciador es el filtro de alcance `Modalidad`.

### 7.3 Decisión: `Modalidad` como filtro de alcance en panel oculto

- **Decisión:** aplicar `Modalidad = "Presencial"` en panel de página oculto y bloqueado para la página Balance Presencial; `Modalidad = "Virtual"` para la página Virtual.
- **Tipo de filtro:** alcance.
- **Capa elegida:** panel de página.
- **Por qué acá y no en DAX:** la métrica "Movilidades" no cambia su definición. Cambia el universo. Crear `[Movilidades Presenciales]` y `[Movilidades Virtuales]` sería hardcodear el contexto en la medida — anti-patrón directo.
- **Trade-off aceptado:** el filtro es invisible para el usuario en el panel de filtros. Mitigación: cuadro de texto en la página documentando el alcance.
- **Test de portabilidad:** si mañana se pide un Balance mixto (Presencial + Virtual), basta con una nueva página que no aplique este filtro. Cero cambios en medidas.

### 7.4 Decisión: `STR_TABLA_ORIGEN_CAL` como filtro de alcance

- **Aplicación:**
  - Balance Presencial / Virtual → `STR_TABLA_ORIGEN_CAL IN {"FCT_MOVILIDAD_ESTUDIANTE", "FCT_VISITANTE_EXTRANJERO"}` (excluye matriculados regulares).
  - Internacionales en La Sabana → `STR_TABLA_ORIGEN_CAL = "FCT_MATRICULADOS"` + filtro adicional de país ≠ Colombia (pendiente validación, ver §10).
- **Tipo:** alcance.
- **Capa:** panel oculto.
- **Por qué:** mismas razones que `Modalidad`. La medida `Movilidades` sigue siendo la misma; cambia el universo.

### 7.5 Decisión: `Dirección` (Entrante/Saliente) como slicer de exploración

- **Tipo:** exploración.
- **Capa:** slicer en el lienzo.
- **Por qué:** la DRI lo pidió textualmente — "diferenciados via filtros". Confirma intención de exploración.
- **Trade-off aceptado:** ninguno relevante.

### 7.6 Decisión: KPIs filtrados por slicer; tops separados estructuralmente

En la página Balance, la dirección se trata distinto según el visual:

| Visual | Tratamiento | Justificación |
|---|---|---|
| KPIs (Movilidades, Personas) | Reactivo al slicer | DRI lo pidió así. |
| Evolución temporal | Dos series fijas (Entrante + Saliente) en mismo gráfico | "Balance" semánticamente implica comparación. Si el usuario filtra a uno, ve uno; con ambos seleccionados, ve la balanza. |
| Top países origen / destino | Dos visuales lado a lado | Universos distintos. País que envía no es necesariamente el que recibe. Separarlos elimina alternancia. |
| Top instituciones | Dos visuales lado a lado | Mismo razonamiento. |
| Top tipos de movilidad | Dos visuales lado a lado | Catálogos de actividades distintos entre entrante y saliente. |

**Importante:** los tops "separados" no se logran con medidas filtradas. Se logran con dos visuales que comparten la misma medida `Movilidades` y se diferencian por filtro de visual (`Dirección = "Entrante"` en uno, `Dirección = "Saliente"` en otro). El filtro a nivel de visual no contamina la medida.

### 7.7 Decisión: dona "% bajo convenio" → tarjeta KPI

- **Tipo:** cambio de visualización.
- **Por qué:**
  - Convenio NO está entre las preguntas prioritarias de la DRI; aparece tangencialmente por SNIES.
  - Una dona de 2 categorías es un anti-patrón de visualización (un valor implica el otro).
  - Ocupa 1/6 del lienzo para una métrica binaria no prioritaria.
- **Solución:** tarjeta KPI pequeña ("57% bajo convenio") junto al resto de KPIs.

### 7.8 Decisión: Export SNIES con dos sub-vistas (no dos páginas)

- **Implementación:** una sola página con bookmarks que alternan entre layout Entrante y Saliente. Las columnas mostradas son distintas porque SNIES exige columnas distintas.
- **Por qué una página y no dos:** el flujo del analista SNIES es secuencial (genera entrante, genera saliente). Una página con toggle mantiene el contexto.
- **Trade-off:** los bookmarks aumentan complejidad de mantenimiento. Aceptable porque las columnas SNIES son fijas por norma (no cambian con frecuencia).

### 7.9 Decisión: medidas base mínimas, no medidas por dirección/modalidad/tipo

Habrá UNA medida `[Movilidades]` y UNA medida `[Personas]`. **No** habrá `[Movilidades Entrantes]`, `[Movilidades Virtuales]`, `[Movilidades de Matriculados]`. El contexto (página, slicer, filtro de visual) hace el resto.

Excepciones legítimas: medidas que sí codifican definición intrínseca (no contexto), como `[% bajo convenio]`, `[Razón Entrante/Saliente]`. Cada una con su bloque de justificación al crearse.

### 7.10 Decisión: Movilidades = COUNTROWS, no SUM

- **Decisión:** `[Movilidades] = COUNTROWS(FCT_Movilidad_Estudiantil)`.
- **Tipo:** definición de medida base.
- **Por qué:** el grano de la FCT (§4.2) garantiza que una fila = un evento de movilidad. `COUNTROWS` refleja directamente esa semántica. `SUM(NUM_NUMERO_MOVILIDAD_CAL)` sería redundante: el campo vale 1 en todos los casos donde se cuenta movilidad real.
- **Trade-off aceptado:** `FCT_MATRICULADOS` inyecta filas con valor 0 que `COUNTROWS` cuenta como 1. El panel de página (§7.4) las excluye. Riesgo diferido a Fase 4.
- **Test de portabilidad:** funciona idéntico en Balance Presencial, Virtual, Detalle y SNIES. Revisar en Fase 4.

### 7.11 Decisión: medidas portables + panel de página oculto

- **Decisión:** eliminar `[Movilidades Presenciales Internacional]` y `[Personas Presenciales Internacional]`. Reemplazar por `[Movilidades]` y `[Personas]` portables. Los 4 filtros de alcance (origen, país, modalidad, periodo regular) viven en panel de página oculto y bloqueado.
- **Tipo:** refactoring arquitectónico de Fase 0.
- **Páginas afectadas:** Resumen Movilidad Presencial Internacional, Detalle Movilidades Presencial Internacional.
- **Por qué:** las medidas anteriores violaban §7.9 y §9 #1, #2, #6 (hardcodear alcance en DAX). El refactoring habilita reutilización en Fases 1, 4, 5 sin medidas nuevas.
- **Trade-off aceptado:** el panel de filtros es invisible al usuario. Mitigación: cuadro de texto de alcance visible bajo el subtítulo dinámico, en ambas páginas.
- **Test de portabilidad:** aprobado. Los mismos números se reproducen con medidas portables + panel.

### 7.12 Decisión: `[% Bajo Convenio]` con filtro intrínseco en DAX

- **Decisión:** `[% Bajo Convenio] = DIVIDE(CALCULATE([Movilidades], STR_DATOS_CONVENIO_CAL_AJT = "Si"), [Movilidades])`.
- **Tipo:** definición intrínseca de KPI.
- **Por qué:** "con convenio" es el numerador del KPI, no alcance de página. Ponerlo en panel obligaría a replicarlo en cada página que use el KPI — falla del test de portabilidad.
- **Trade-off aceptado:** `CALCULATE` sin `KEEPFILTERS` hace override si el usuario filtra por convenio. Aceptable: tautológico.
- **Test de portabilidad:** funciona en Resumen, Virtual (F5), Internacionales (F4), SNIES (F2). Sin cambios.

### 7.13 Decisión: tops con dimensión Dirección incorporada (no filtro de visual separado)

- **Decisión:** los 3 tops (países, tipos, instituciones) usan un solo visual con Dirección como dimensión interna (Legend/eje), no dos visuales separados con filtro de visual por Dirección.
- **Tipo:** desvío justificado de §7.6.
- **Por qué:** permite ver E/S simultáneamente en un vistazo. Con filtro de visual separado, el usuario necesitaba comparar dos visuales apilados — mayor carga cognitiva para directivos. La paleta consistente (`#2E5597` Entrante, `#BDD7EE` Saliente) preserva la lectura E/S sin necesidad de visuales separados.
- **Consecuencia:** la "decisión de implementación" Edit Interactions → None (planeada originalmente para evitar conflicto entre slicer y filtro de visual) deja de aplicar. No hay conflicto porque no existe filtro de visual por Dirección — el slicer pasa limpio a través del visual.
- **Trade-off aceptado:** visual más denso. Mitigado con Top N (§7.14).
- **Test de portabilidad:** mismo patrón replicable en Fase 5 (Movilidad Virtual).

### 7.14 Decisión: Top 5 con filtro nativo + tooltip pages diferidas

- **Decisión:** los 3 tops muestran Top 5 por [Movilidades] con filtro Top N nativo de Power BI, descendente. Tooltip pages con ranking completo diferidas a post-reunión stakeholder.
- **Tipo:** UX / densidad informativa.
- **Por qué:** Top 5 es suficiente para lectura ejecutiva. La profundidad adicional (tooltip pages con ranking extendido) se valida con stakeholder antes de construir para no sobreconstruir. Sin categoría "Otros" agrupada — diluye lectura ejecutiva.
- **Trade-off aceptado:** acceso al ranking completo no disponible desde esta página hasta validación con stakeholder.
- **Test de portabilidad:** patrón replicable en Fase 5.

### 7.15 Decisión: slicers de exploración en panel lateral (no en lienzo horizontal)

- **Decisión:** todos los slicers (incluyendo Año, Semestre, Dirección) van en panel lateral estándar heredado de plantilla §11.6. El wireframe de §8.2 los mostraba en lienzo horizontal — ese sketch era conceptual, no normativo.
- **Tipo:** estándar operativo.
- **Por qué:** consistente con §11.5.1. El panel lateral estandarizado es la convención institucional y mantiene sincronización entre páginas analíticas.
- **Trade-off aceptado:** ninguno relevante.

---

## 8. Arquitectura de páginas

### Vista general

```
┌─────────────────────────────────────────────────────────┐
│                    NAVEGACIÓN PERSISTENTE                │
├─────────────────────────────────────────────────────────┤
│  VISIÓN                                                  │
│  └─ 1. Cobertura Internacional (existente)              │
│                                                          │
│  ANÁLISIS                                                │
│  ├─ 2. Balance de Movilidad Presencial                  │
│  ├─ 3. Movilidad Virtual                                │
│  └─ 4. Internacionales en La Sabana                     │
│                                                          │
│  OPERACIÓN                                               │
│  ├─ 5. Detalle Movilidad                                │
│  └─ 6. Export SNIES                                     │
└─────────────────────────────────────────────────────────┘
```

### 8.1 Cobertura Internacional *(existente, sin cambios)*

- **Audiencia:** directivos.
- **Pregunta:** ¿Dónde estamos presentes en el mundo?
- **Estado:** producción, no se toca.

### 8.2 Balance de Movilidad Presencial *(rediseño completo)*

- **Audiencia:** directivos + DRI.
- **Pregunta:** ¿Cuál es el flujo entrante vs saliente y cómo evoluciona?
- **Alcance (panel oculto):** `Modalidad = "Presencial"` AND `STR_TABLA_ORIGEN_CAL IN {Movilidad, Visitantes}`.
- **Cuadro de texto visible:** "Esta página analiza movilidad presencial de estudiantes temporales (entrantes y salientes). Excluye estudiantes regulares matriculados y modalidad virtual."

**Estructura visual implementada (Fase 1 — cerrada 2026-05-22):**

Layout sobre canvas 1080×600 (área útil descontando panel lateral de slicers y barra inferior):

```
┌──────────────────────────────────────────────────────────┐
│  [Movilidades]       [Personas]       [% Convenio]       │ 65px
├──────────────────────────────────────────────────────────┤
│                                                          │
│            Evolución temporal (líneas E/S)                │ 240px
│                                                          │
├──────────────────┬──────────────────┬────────────────────┤
│  Top Países      │  Top Tipos Mov   │  Top Instituciones │ 280px
│  (E + S juntos)  │  (E + S juntos)  │  (E + S juntos)    │
└──────────────────┴──────────────────┴────────────────────┘
```

**Fila 1 — KPIs (65px):**
- 3 tarjetas individuales (new Card visual).
- Movilidades y Personas a ~400px, % Convenio a ~250px.
- Font: 36pt bold primarias, 28pt bold secundaria.
- Color valor: `#1F3864` primarias, `#595959` % Convenio.

**Fila 2 — Evolución temporal (240px):**
- Line Chart ancho completo.
- Eje X: período académico categórico (`YYYY-N`), 9pt rotación 45°.
- Eje Y: [Movilidades].
- Legend: `STR_CLASIFICACION_MOVILIDAD_CAL_AJT`.
- Series: Entrante `#2E5597`, Saliente `#BDD7EE`.
- Tooltip: [Personas]. Etiquetas de datos: Off.
- Gridlines horizontales activas en `#F2F2F2`.

**Fila 3 — Tops (280px):**
- 3 Clustered Bar Charts horizontales (no 6 visuales — ver §7.13).
- Dimensión Dirección incorporada como serie interna del visual.
- Top 5 por [Movilidades], descendente, filtro Top N nativo (ver §7.14).
- Títulos explícitos: "Top Países", "Top Tipos de Movilidad", "Top Instituciones".
- Etiquetas al final de barra, 10pt.
- Eje X desactivado. Gridlines desactivadas.
- Sin categoría "Otros" agrupada.

**Elementos transversales:**
- Paleta E/S consistente: `#2E5597` Entrante, `#BDD7EE` Saliente.
- Subtítulo dinámico heredado de plantilla.
- Cuadro de texto de alcance visible bajo subtítulo.
- Barra inferior: "Nota: No se incluyen interacciones virtuales" + fecha de corte.
- Slicers Año, Semestre, Dirección, Nivel, Unidad, Programa, País en panel lateral (ver §7.15).

### 8.3 Movilidad Virtual *(nueva)*

- **Audiencia:** DRI.
- **Pregunta:** ¿Qué peso tiene la modalidad virtual y cómo se comporta?
- **Alcance (panel oculto):** `Modalidad = "Virtual"` AND `STR_TABLA_ORIGEN_CAL IN {Movilidad, Visitantes}`.
- **Estructura:** clon estructural de §8.2. Mismas medidas, mismo layout. Solo cambia el filtro de alcance.

### 8.4 Internacionales en La Sabana *(nueva)*

- **Audiencia:** directivos + DRI.
- **Pregunta:** ¿Quiénes nos eligen para estudiar (regular o temporalmente)?
- **Alcance (panel oculto):** país ≠ Colombia (todos los orígenes).
- **Visuales clave:**
  - KPIs: total internacionales, desglose Regular vs Temporal.
  - Mapa de países origen.
  - Distribución por programa / nivel.
  - Evolución temporal de atracción.
  - Top instituciones de procedencia (solo Temporales).
- **Pendiente bloqueante:** ver §10.

### 8.5 Detalle Movilidad *(ajuste menor sobre lo existente)*

- **Audiencia:** directivos (consulta puntual).
- **Estado actual:** página de imagen 3, dos visuales (consulta + descargue).
- **Ajuste a realizar:** unificar en una tabla, con selector Entrante/Saliente que cambia el set de columnas visibles (entrante y saliente tienen columnas distintas según naturaleza del registro).

### 8.6 Export SNIES *(nueva)*

- **Audiencia:** analistas SNIES.
- **Propósito:** tabla plana exportable con layout SNIES.
- **Filtros visibles:** Año + Semestre + Dirección.
- **Layouts (por bookmarks):**

**SNIES Saliente:**
```
AÑO · SEMESTRE · ID_TIPO_DOCUMENTO · NUM_DOCUMENTO
· ID_PAIS_EXTRANJERO · INSTITUCION_EXTRANJERA
· ID_TIPO_MOV_EST_EXTERIOR · NUM_DIAS_MOVILIDAD
· MOVILIDAD_POR_CONVENIO · CODIGO_CONVENIO
· ID_FUENTE_NACIONAL_INVESTIG · VALOR_FINANCIACION_NACIONAL
· ID_FUENTE_INTERNACIONAL · ID_PAIS_FINANCIADOR
· VALOR_FINANCIACION_INTERNAC
```

**SNIES Entrante:**
```
AÑO · SEMESTRE · ID_TIPO_DOCUMENTO · NUM_DOCUMENTO
· PRIMER_NOMBRE · SEGUNDO_NOMBRE · PRIMER_APELLIDO · SEGUNDO_APELLIDO
· ID_PAIS_EXTRANJERO · INSTITUCION_EXTRANJERA
· ID_TIPO_MOV_EST_EXTRANJ · NUM_DIAS_MOVILIDAD
· MOVILIDAD_POR_CONVENIO · CODIGO_CONVENIO
· ID_FUENTE_NACIONAL_INVESTIG · VALOR_FINANCIACION_NACIONAL
· ID_FUENTE_INTERNACIONAL · ID_PAIS_FINANCIADOR
· VALOR_FINANCIACION_INTERNAC
```

**Mapeo modelo → SNIES:** se documenta en Fase 2 como anexo aparte (requiere tabla de homologación de `ID_TIPO_MOV_EST_*` confirmada disponible).

---

## 9. Anti-patrones a rechazar

| # | Anti-patrón | Detección |
|---|---|---|
| 1 | Crear `[Movilidades Entrantes]`, `[Movilidades Salientes]` cuando `[Movilidades]` + slicer resuelven | Medidas que solo difieren por un filtro de contexto. |
| 2 | Mover filtros de alcance a DAX para "dejarlos documentados" | Documentación va en bloque de comentarios estándar de la medida (§5.2) + cuadro de texto, no en `CALCULATE`. |
| 3 | Mezclar alcance + definición de negocio en una misma medida | Una medida hace una cosa. |
| 4 | Dashboard saturado por página | Más de ~7 visuales por página o mezcla de niveles analíticos. |
| 5 | Estética sobre claridad ejecutiva | Donas binarias, gradientes innecesarios, animaciones decorativas. |
| 6 | Replicar lógica de alcance en N medidas | Aplicar el alcance una vez en la página, no N veces en DAX. |
| 7 | "El equipo lee DAX, no necesita documentación" | La documentación al usuario va en cuadros de texto. DAX es para mantenedores. |
| 8 | Dos métricas redundantes en mismo eje | Ej. Movilidades + Personas en misma línea — están correlacionadas, una en tooltip basta. |

---

## 10. Pendientes y riesgos abiertos

### 10.1 Pendientes bloqueantes

| # | Pendiente | Bloquea fase | Responsable |
|---|---|---|---|
| P1 | Validar semántica del campo país en `FCT_Movilidad_Estudiantil` para registros de origen `FCT_MATRICULADOS`. ¿Representa nacionalidad del matriculado o algo distinto? Determina si filtrar por país ≠ Colombia es válido para identificar internacionales regulares. | Fase 4 | Equipo modelado |
| P2 | Tabla de homologación `ID_TIPO_MOV_EST_EXTERIOR` / `ID_TIPO_MOV_EST_EXTRANJ` (código SNIES ↔ descripción interna). Confirmada existencia, falta integrar al modelo si no lo está. | Fase 2 | Equipo modelado |
| ~~P3~~ | ✅ **Cerrado en Fase 0 (2026-05-15).** Resuelto trivialmente: `STR_DATOS_CONVENIO_CAL_AJT` ya es binario "Si"/"No" en el origen. No requiere columna calculada ni binarización. | — | — |

### 10.2 Riesgos abiertos

| # | Riesgo | Mitigación |
|---|---|---|
| R1 | Datos financieros (`NUM_VALOR_FINAN_*`) existen en el modelo pero no se han explotado en ningún visual hasta hoy. Sin validación previa de calidad/completitud por período. | Auditoría de cobertura por período antes de Fase 6. Identificar períodos con dato vs sin dato para mostrar mensaje explícito en visual. |
| R2 | Nombres internos del modelo (`STR_NOMBRE_ENTIDAD_EXTERNA_CAL_AJT`) vs nombres normativos SNIES (`INSTITUCION_EXTRANJERA`). | Mapeo en capa de visual (renombrado en columna del visual), no en modelo. No rompe DAX existente. |
| R3 | 15k registros es bajo volumen; refresh 2x/semestre. Riesgo mínimo de performance, pero validar al final. | Test de performance en Fase 0. |
| R4 | Cobertura Internacional (página existente) podría usar medidas que se modifiquen en Fase 0. | Auditar dependencias antes de tocar medidas existentes. |

### 10.3 Decisiones diferidas

- **Drill-through entre páginas:** se decide en Fase 1, una vez establecida la página Balance. **Diferido a Fase 3** — drill-through hacia Detalle Movilidad se construye cuando Detalle esté rediseñado, para evitar dependencia frágil sobre página que va a cambiar.
- **Sincronización de slicers entre páginas:** se decide en Fase 0, junto con la convención de slicers.
- **Tooltip pages ranking completo (tops Fase 1):** diferido a post-reunión stakeholder. Validar si Top 5 satisface requerimiento o si se necesita profundidad adicional. Si se requiere → construir 3 tooltip pages ocultas (países, tipos, instituciones) antes de cierre de Fase 5.

---

## 11. Estándares operativos

> Estándares adoptados por criterio de industria (Microsoft / SQLBI / patrones comunes en BI corporativo). Sujetos a ajuste en marcha si la realidad del proyecto exige otra cosa.

### 11.1 Convenciones de naming

#### 11.1.1 Medidas

- **Formato:** PascalCase con espacios.
- ✅ `Movilidades`, `Personas`, `% Bajo Convenio`, `Movilidades Año Anterior`
- ❌ `mov_count`, `MovTotal`, `m_movilidades`

#### 11.1.2 Medidas auxiliares (ocultas)

Medidas intermedias que otras consumen y no deben aparecer al usuario final:

- **Prefijo `_`** + propiedad `IsHidden = true`.
- Aparecen ordenadas al inicio del panel del desarrollador.
- ✅ `_Movilidades Base`, `_Filtro Año Anterior`

#### 11.1.3 Tabla de medidas

- Tabla vacía llamada **`_Medidas`** (underscore al inicio para ordenamiento alfabético).
- Patrón estándar SQLBI: la tabla no contiene datos, solo aloja medidas.
- Subcarpetas por dominio dentro de `_Medidas`:
  - `Movilidad/`
  - `Personas/`
  - `Convenio/`
  - `Financiación/`
  - `Auxiliares/`

#### 11.1.4 Variables DAX dentro de medidas

- Ya definido en §5.2: **PascalCase descriptivo en español**.
- ✅ `PromedioPuntaje`, `TotalEstudiantesEvaluados`, `MovilidadesFiltradas`
- ❌ `_promedio`, `var1`, `result`, `AverageScore`

#### 11.1.5 Columnas calculadas

- Solo crear si **estrictamente necesario** (siempre preferir medidas).
- Si se crea: **PascalCase con espacios** + propiedad `Description` (esta sí va a nivel de columna, no se reemplaza por comentario DAX porque no hay bloque DAX equivalente).

### 11.2 Tabla de fechas y time intelligence

#### 11.2.1 Decisión

- **No se crea `DIM_FECHA` diaria.** Se usa `DIM_PERIODO_ACADEMICO` existente con grano semestral.
- **No se marca como Date Table** en Power BI. Las funciones de time intelligence DAX estándar (`SAMEPERIODLASTYEAR`, `DATEADD`, `TOTALYTD`) **no se usan**.

#### 11.2.2 Implicación

Comparativos temporales se construyen **manualmente** con patrones DAX explícitos. Ejemplo:

```dax
Movilidades Semestre Anterior =
/* -----------------------------------------------------------------------------
    [CASO DE USO]: Comparativo de movilidades del mismo período del semestre
                   anterior. Soporta análisis de evolución sin time intelligence
                   nativa (grano semestral, no diario).
----------------------------------------------------------------------------- */
VAR PeriodoActual =
    SELECTEDVALUE ( DIM_PERIODO_ACADEMICO[NUM_PERIODO_SK] )
VAR PeriodoAnterior =
    PeriodoActual - 1
VAR MovilidadesAnterior =
    CALCULATE (
        [Movilidades],
        FILTER (
            ALL ( DIM_PERIODO_ACADEMICO ),
            DIM_PERIODO_ACADEMICO[NUM_PERIODO_SK] = PeriodoAnterior
        )
    )
RETURN
    MovilidadesAnterior
```

#### 11.2.3 Criterio de necesidad

Los comparativos temporales **no están en la lista priorizada de la DRI** pero son estándar mínimo de BI ejecutivo. Se construyen bajo demanda, no se crean preventivamente para evitar inflar el catálogo de medidas.

### 11.3 Sistema visual

#### 11.3.1 Paleta corporativa

Códigos hex propuestos basados en las páginas existentes. Sujetos a confirmación contra manual de marca institucional si aparece.

| Rol | Hex | Uso |
|---|---|---|
| Azul institucional oscuro | `#1F3864` | Header, KPIs principales, tipografía destacada |
| Azul medio | `#2E5597` | Serie primaria en visuales, fondos de tarjetas |
| Azul claro | `#BDD7EE` | Serie secundaria, acentos |
| Gris claro fondo | `#F2F2F2` | Fondo de paneles, separadores suaves |
| Gris texto secundario | `#595959` | Etiquetas, ejes, anotaciones |
| Texto principal | `#262626` | Cuerpo de texto, datos en tablas |
| Rojo alerta | `#C00000` | Botón "Borrar filtros", indicadores negativos |
| Verde positivo | `#548235` | Indicadores favorables (uso restringido) |
| Naranja advertencia | `#ED7D31` | Llamados de atención (uso restringido) |

#### 11.3.2 Tamaño de página estándar

- **1920×1080 (16:9 grande).**
- Razón: rango óptimo para dashboards corporativos modernos. Más espacio para layouts ejecutivos sin sacrificar legibilidad en pantallas estándar (escalado automático en monitores menores).

#### 11.3.3 Tipografía

- **Segoe UI** (fuente nativa de Power BI).
- Tamaños base sugeridos:
  - Título de página: 20-24 pt, semibold
  - Título de visual: 14 pt, semibold
  - KPIs grandes: 32-40 pt, bold
  - Cuerpo / etiquetas: 10-11 pt, regular
  - Notas al pie: 9 pt, regular

#### 11.3.4 Elementos institucionales fijos

Heredados del diseño existente (no se rediseñan):

- Logo Universidad de La Sabana — esquina superior izquierda
- Botón "Borrar filtros" — esquina superior derecha (rojo `#C00000`)
- Botón "Home" — esquina inferior izquierda
- Barra inferior con fuente del dato — pie de página

### 11.4 Formateo de datos

#### 11.4.1 Regional

- **Formato `es-CO`.**
- Separador de miles: punto (`.`)
- Separador decimal: coma (`,`)
- Ejemplo: `10.223,50`

#### 11.4.2 Decimales por tipo de métrica

| Tipo | Decimales | Ejemplo |
|---|---|---|
| Conteos enteros (Movilidades, Personas) | 0 | `10.223` |
| Porcentajes | 1 | `57,3%` |
| Monedas COP | 0 | `$ 1.250.000` |
| Monedas USD / EUR | 2 | `$ 1.250,50` |
| Ratios / índices | 2 | `1,42` |
| Días (NUM_DURACION_DIAS) | 0 | `120` |

#### 11.4.3 Manejo de medidas vacías

- **`BLANK()` por defecto.**
- Razón: cero engaña (sugiere "no pasó nada" cuando en realidad "no hay dato"). Cadena `"Sin datos"` satura visualmente y rompe ordenamiento numérico.
- Excepción: tarjetas KPI principales donde una celda vacía pueda confundir al directivo. En esos casos puntuales se justifica con bloque de decisión.

#### 11.4.4 Formato de fechas en visuales

- **Períodos académicos:** `YYYY-N` (ej. `2026-1`, `2026-2`). Compacto y ordenable.
- **Fechas calendario** (cuando aparezcan): `DD/MM/YYYY` para consumo humano, `YYYY-MM-DD` para exportaciones SNIES.

### 11.5 Política de slicers
### 11.5.1 Tabla de slicers estándar (confirmada Fase 0)

Esta tabla es la referencia normativa. Las páginas analíticas nuevas copian el panel completo de Resumen Movilidad Presencial Internacional.

| Slicer | Tipo de visual | Orden | Pre-selección |
|---|---|---|---|
| Movilidad (Dirección) | Chiclet horizontal | Entrante, Saliente | Sin pre-selección (ambos activos) |
| Año | Dropdown | Descendente | Sin pre-selección |
| Semestre | Chiclet horizontal | 1, 2 | Sin pre-selección |
| Nivel Académico | Dropdown | Jerarquía institucional | Sin pre-selección |
| Unidad Académica | Dropdown | Alfabético ascendente | Sin pre-selección |
| Programa Académico | Dropdown | Alfabético ascendente | Sin pre-selección |
| País | Dropdown | Alfabético ascendente | Sin pre-selección |

**Regla operativa:** cada página analítica nueva se construye copiando el panel de slicers de Resumen tal cual. Si una página necesita un slicer adicional, se agrega al final sin reordenar los existentes.

**Sincronización:** activa entre páginas analíticas (Balance Presencial, Virtual, Internacionales). Desactivada con Detalle y SNIES (§11.5.3).

### 11.6 Plantilla de página

**Convención:** no se mantiene un activo "plantilla" separado. Cada página nueva se construye copiando Resumen Movilidad Presencial Internacional y vaciando los visuales, preservando: logo, panel de slicers, botón borrar filtros, botón home, barra inferior de fuente, subtítulo dinámico, cuadro de texto de alcance (vacío para llenar).

### 11.7 Performance baseline (Fase 0)

Medición: Performance Analyzer en Power BI Desktop. Dataset: ~15.000 filas en FCT_Movilidad_Estudiantil. Página: Resumen Movilidad Presencial Internacional. Fecha: 2026-05-15.

**Metodología:** wall clock por refresh = max(visual end) − min(visual start). La suma serial de duraciones no aplica porque los visuales renderizan en paralelo.

| Escenario | Wall clock | Visual más lento | DAX más lenta |
|---|---|---|---|
| Cold cache (primer refresh tras abrir .pbip) | 657 ms | Tarjetas KPIs (630 ms, render de cardVisual) | Mapa Azure (28 ms) |
| Warm cache (último refresh con filtros aplicados) | 515 ms | Tendencia mov vs personas (507 ms, render de lineChart) | SubTítulo (24 ms) |

**Conclusiones:**
- DAX queries todas <25 ms. Motor VertiPaq sin estrés.
- Cuello de botella en render de visuales custom (cardVisual, azureMap) y lineChart con múltiples series. No en cálculo.
- Modelo en estrella sano para el volumen actual.
- Cualquier degradación futura se evaluará contra este baseline. Sospechar primero render/visuales nuevos, después DAX.

---

## 12. Plan de implementación por fases

Cada fase es **un entregable autocontenido** versionable en git. Una fase no se cierra hasta cumplir su Definition of Done.

### Definition of Done global (aplica a toda fase)

- ✅ Código DAX comiteado (formato `.pbip`).
- ✅ Inventario de filtros documentado en este `.md` (sección 7 actualizada).
- ✅ Justificación 5-puntos para cada medida nueva y cada filtro nuevo.
- ✅ Documentación según estándar §5.2 (encabezado de negocio + variables PascalCase + formato impecable) en todas las medidas creadas o modificadas.
- ✅ Cuadro de texto de alcance visible en cada página entregada.
- ✅ Test de portabilidad documentado.
- ✅ Pull Request revisado y merged a `main`.

---

### **Fase 0 — Fundación** | branch: `phase-0-foundation`

**Objetivo:** dejar lista la capa base sobre la que se construyen todas las páginas. Sin esta fase, las demás no pueden empezar.

**Entregables:**

1. **Catálogo de medidas DAX base** (tabla `_Medidas`):
   - `[Movilidades]`
   - `[Personas]`
   - `[% bajo Convenio]`
   - `[Razón Entrante/Saliente]` *(opcional, evaluar en fase)*
2. **Tabla de medidas** organizada por carpetas (`Movilidad/`, `Convenio/`, `Auxiliares/`).
3. **Documentación de cada medida según estándar §5.2** (encabezado `[CASO DE USO]` + variables PascalCase + formato impecable).
4. **Convenciones documentadas:**
   - Nombre de medidas (PascalCase con espacios).
   - Nombre de slicers compartidos.
   - Política de sincronización de slicers.
5. **Plantilla de página** con elementos institucionales fijos (logo, panel filtros, botón borrar, barra inferior, navegación home).
6. **Cuadro de texto reutilizable** para documentación de alcance (formato y posición estándar).
7. **Test de performance baseline** con dataset actual.

**Definition of Done específico:**

- Las 4 medidas funcionan en una página de pruebas.
- Cada medida tiene `[CASO DE USO]` claro según estándar §5.2.
- Plantilla replicable lista para Fase 1.

**Dependencias:** ninguna. Es la fase base.

**Pendientes que se resuelven aquí:** P3 (binarización de convenio).

**✅ Fase 0 cerrada — 2026-05-15.**

Entregables completados:
- Tabla `_Medidas` con displayFolders (`Movilidad/`, `Personas/`, `Convenio/`, `Auxiliares/`, `Notas/`).
- Medidas portables `[Movilidades]`, `[Personas]`, `[% Bajo Convenio]` documentadas según §5.2.
- Eliminadas medidas hardcodeadas (`Total Movilidades`, `Total Personas`, `Movilidades Presenciales Internacional`, `Personas Presenciales Internacional`).
- Filtros de alcance migrados a panel oculto en Resumen y Detalle.
- Cuadro de texto de alcance visible en ambas páginas.
- P3 cerrado.
- Convenciones de slicers (§11.5.1), plantilla (§11.6) y performance baseline (§11.7) documentados.
- `[Razón Entrante/Saliente]` diferida hasta que la DRI la solicite.

---

### **Fase 1 — Balance de Movilidad Presencial** | branch: `phase-1-balance-presencial`

**Objetivo:** página de mayor valor inmediato. Responde 5 de 6 preguntas DRI.

**Entregables:**

1. Página "Balance de Movilidad Presencial" implementada según §8.2.
2. Filtros de alcance configurados en panel oculto:
   - `Modalidad = "Presencial"`
   - `STR_TABLA_ORIGEN_CAL IN {Movilidad, Visitantes}`
3. Slicers de exploración (Año, Semestre, Dirección, Nivel, Unidad, Programa, País).
4. Cuadro de texto visible con el alcance.
5. Visuales:
   - 3 KPIs (Movilidades, Personas, % Convenio).
   - Evolución temporal con 2 series (Entrante + Saliente).
   - Top país origen + Top país destino (lado a lado).
   - Top tipo movilidad entrante + saliente (lado a lado).
   - Top instituciones origen + destino (lado a lado).
6. Inventario completo de filtros en este `.md`.

**Definition of Done específico:**

- Las 5 preguntas DRI #1 a #5 son respondibles desde esta página, solo presencial.
- Filtro de alcance NO modifica las medidas base.
- Slicer Dirección con ambos valores seleccionados por defecto.

**Dependencias:** Fase 0.

**Pendientes que se resuelven aquí:** ninguno, depende de P3 ya resuelto.

**✅ Fase 1 cerrada — 2026-05-22.**

Entregables completados:
- Página "Balance de Movilidad Presencial" implementada según §8.2 (estructura final).
- Filtros de alcance en panel oculto: `Modalidad = "Presencial"` AND `STR_TABLA_ORIGEN_CAL IN {Movilidad, Visitantes}`.
- 3 KPIs (Movilidades, Personas, % Bajo Convenio) sin medidas nuevas — reutilización completa de medidas portables de Fase 0.
- Evolución temporal con 2 series E/S, etiquetas off, tooltip con [Personas], eje X categórico rotado 45°.
- 3 tops con dimensión Dirección incorporada como serie interna del visual (desvío justificado de §7.6, ver §7.13).
- Top 5 con filtro nativo de Power BI, sin categoría "Otros" (ver §7.14).
- Slicers en panel lateral estándar, no en lienzo horizontal (ver §7.15).
- Cuadro de texto de alcance visible bajo subtítulo dinámico.
- Cero medidas nuevas — medidas base de Fase 0 cubren toda la página.
- Drill-through diferido a Fase 3.
- Tooltip pages con ranking completo diferidas a validación post-stakeholder (§10.3).
- Las 5 preguntas DRI #1–#5 son respondibles desde la página.

---

### **Fase 2 — Export SNIES** | branch: `phase-2-export-snies`

**Objetivo:** resolver dolor operativo recurrente (reconstrucción manual de reporte SNIES).

**Entregables:**

1. Página "Export SNIES" con dos bookmarks (Entrante / Saliente).
2. Mapeo columna modelo → columna SNIES documentado como anexo.
3. Filtros visibles: Año, Semestre, Dirección.
4. Tabla plana sin estética analítica (formato exportación).
5. Botón "Exportar a Excel/CSV" o instrucciones de exportación nativa.
6. Cuadro de texto con instrucciones de uso para el analista SNIES.

**Definition of Done específico:**

- Layout entrante y saliente coinciden 1:1 con norma SNIES.
- Tabla de homologación de tipos de movilidad integrada (P2).
- Exportación produce archivo directamente cargable a SNIES (o lo más cercano posible).

**Dependencias:** Fase 0, P2 resuelto.

**Pendientes que se resuelven aquí:** P2.

---

### **Fase 3 — Detalle Movilidad** | branch: `phase-3-detalle-movilidad`

**Objetivo:** ajuste menor sobre página existente. Unificación de consulta y descargue.

**Entregables:**

1. Tabla unificada con selector Entrante/Saliente.
2. Columnas distintas según dirección (entrante incluye más datos personales, saliente menos).
3. Filtros estándar.
4. Botón "Volver al informe" (ya existe).

**Definition of Done específico:**

- Una sola tabla en lugar de dos.
- Cambio de dirección actualiza columnas visibles sin refrescar página.

**Dependencias:** Fase 0.

---

### **Fase 4 — Internacionales en La Sabana** | branch: `phase-4-internacionales`

**Objetivo:** página nueva, cubre vacío estratégico.

**Entregables:**

1. Página implementada según §8.4.
2. Filtro de alcance `país ≠ Colombia`.
3. KPIs con desglose Regular vs Temporal.
4. Mapa, evolución, distribución por programa, top instituciones.

**Definition of Done específico:**

- Pregunta "¿quiénes nos eligen?" respondible desde la página.
- Distinción visual clara entre matriculados regulares y movilidad temporal.

**Dependencias:** Fase 0, **P1 resuelto** (semántica del país en matriculados).

**⚠️ Bloqueante:** no iniciar hasta confirmar P1.

---

### **Fase 5 — Movilidad Virtual** | branch: `phase-5-virtual`

**Objetivo:** clonar Balance con alcance virtual.

**Entregables:**

1. Página implementada según §8.3.
2. Filtro de alcance `Modalidad = "Virtual"`.
3. Misma estructura visual que Balance Presencial.

**Definition of Done específico:**

- Cero medidas nuevas. Solo reutilización de Fase 1.
- Estructura idéntica a Balance Presencial.

**Dependencias:** Fase 1 (es clon estructural).

---

### **Fase 6 — Financiación** | branch: `phase-6-financiacion`

**Objetivo:** sumar la 6ª pregunta DRI (financiación). Los datos existen en el modelo pero no se han explotado.

**Entregables:**

1. Auditoría previa de cobertura por período (qué % de filas tienen valor financiero).
2. Visuales financieros en Balance (Presencial + Virtual).
3. Desglose por financiador nacional / internacional.
4. Manejo de monedas múltiples (`STR_MONEDA_FINAN_*`).
5. Indicación visual de períodos sin datos cuando aplique.

**Definition of Done específico:**

- Auditoría de calidad de datos validada con DRI.
- Conversión de monedas validada con DRI (¿tasa fija? ¿histórica? ¿se reporta en moneda original?).
- Visuales no muestran $0 engañosos donde realmente no hay dato — usan formato "Sin datos" o equivalente.

**Dependencias:** R1 auditado.

---

### Orden recomendado de ejecución

```
Fase 0 ──> Fase 1 ──┬──> Fase 5
                    ├──> Fase 2
                    └──> Fase 3

Fase 0 ──> Fase 4 (cuando P1 esté resuelto)

Fase 0 + auditoría R1 ──> Fase 6
```

**Camino crítico:** Fase 0 → Fase 1 → entrega mínima viable que responde preguntas DRI presenciales.

---

## 13. Convenciones de versionado git

### 13.1 Formato del proyecto

- Power BI en formato `.pbip` (proyecto, no `.pbix`).
- Genera carpeta con archivos legibles por git (JSON, TMDL).

### 13.2 Estructura de branches

```
main                    ← producción
├── phase-0-foundation
├── phase-1-balance-presencial
├── phase-2-export-snies
├── phase-3-detalle-movilidad
├── phase-4-internacionales
├── phase-5-virtual
└── phase-6-financiacion
```

Una fase = un branch = un PR a `main`.

### 13.3 Convención de commits

Formato: `tipo(fase): descripción`

| Tipo | Uso |
|---|---|
| `feat` | Nueva página, nueva medida, nuevo visual. |
| `fix` | Corrección de bug en medida o visual. |
| `refactor` | Cambio de implementación sin cambio funcional. |
| `docs` | Cambio en este `.md` o `Description` de medidas. |
| `chore` | Reorganización del modelo, renombres, limpieza. |

**Ejemplos:**

```
feat(phase-1): agrega evolución temporal con dos series E/S
fix(phase-0): corrige DISTINCTCOUNT en medida Personas
docs(phase-0): documenta justificación de medida [% bajo Convenio]
refactor(phase-2): mueve filtro de Dirección de DAX a slicer
```

### 13.4 Pull Request — checklist obligatorio

Cada PR debe cumplir:

- [ ] Definition of Done global (sección 12).
- [ ] Definition of Done específico de la fase.
- [ ] Este `.md` actualizado en las secciones afectadas.
- [ ] Sin medidas duplicadas por alcance (verificar §7.9).
- [ ] Sin anti-patrones de §9.
- [ ] Cuadro de texto de alcance presente en cada página de la fase.
- [ ] Test visual: capturas antes/después si aplica.

### 13.5 Política de hotfix

Si producción rompe entre fases:

- Branch `hotfix-<descripción>` desde `main`.
- PR directo a `main` + cherry-pick a la fase activa si aplica.
- Documentar en este `.md` bajo §10.

---

## 14. Metodología de trabajo del plan

> Cómo ejecutar el plan paso a paso de forma sostenible. Esta sección define el ritmo, la estructura de sesiones y el flujo de información entre fases.

### 14.1 Principio rector

**Una fase = una conversación = un branch git = un PR.**

Cada fase del plan se trabaja en una conversación dedicada con el asistente. El documento maestro (`PLAN_MAESTRO_DASHBOARD_MOVILIDAD.md`) es el artefacto que viaja entre conversaciones. Es la memoria persistente del proyecto.

### 14.2 Ciclo por fase

```
┌─────────────────────────────────────────────────────────┐
│  1. Abrir conversación nueva                            │
│  2. Adjuntar plan maestro (.md actualizado)             │
│  3. Mensaje de apertura: "Fase N — empezamos"           │
│  4. Trabajar contra Definition of Done de la fase       │
│  5. Validar entregable                                  │
│  6. Actualizar plan maestro con decisiones nuevas       │
│  7. Commit + PR + merge a main                          │
│  8. Cerrar conversación                                 │
└─────────────────────────────────────────────────────────┘
```

### 14.3 Por qué una conversación por fase y no un chat continuo

| Aspecto | Chat único continuo | Chat por fase |
|---|---|---|
| Contexto | Crece sin límite, se degrada | Fresco, enfocado |
| Búsqueda histórica | Difícil ubicar decisiones | Cada chat = una fase identificable |
| Costo / performance | Aumenta progresivamente | Estable |
| Mapeo a git | Confuso | 1:1 con branch |
| Riesgo de inconsistencia | Alto (decisiones tácitas se diluyen) | Bajo (cada fase cierra con .md actualizado) |

**Conclusión:** chat por fase es el patrón recomendado.

### 14.4 Mensaje de apertura estándar para una fase nueva

Plantilla copy-paste para iniciar la conversación de una fase:

```
Estoy iniciando [Fase N — Nombre] del plan maestro adjunto.

Estado del proyecto:
- Fases completadas: [lista]
- Última actualización del plan: [fecha]
- Pendientes abiertos relevantes: [P1, P2, etc. si aplica]

Objetivo de esta sesión: [qué quieres lograr hoy específicamente]

Tu rol: continuar como Arquitecto Principal BI según el plan.
Empezamos por [punto específico].
```

### 14.5 Sub-fases dentro de una fase grande

Algunas fases pueden requerir varias sesiones (Fase 1 — Balance es candidata). En ese caso:

**Opción A — Una sola conversación larga (preferida si la fase cabe en ~30 mensajes):**
- Mantener la conversación abierta hasta cerrar DoD.
- Cierre limpio al final.

**Opción B — Múltiples sesiones (si la fase es muy grande):**
- Sub-sesiones temáticas dentro de la fase: ej. "Fase 1.A — Medidas DAX", "Fase 1.B — Layout visual", "Fase 1.C — Validación".
- Cada sub-sesión termina con commit, sin PR.
- PR se abre al cierre de la fase completa.

### 14.6 Qué entregar al cerrar cada fase

Antes de cerrar la conversación de una fase, el plan maestro debe quedar actualizado con:

1. **Sección 7** — nuevas decisiones arquitectónicas tomadas durante la fase, con bloque 5-puntos completo.
2. **Sección 8** — refinamientos a la página específica trabajada.
3. **Sección 10** — pendientes resueltos marcados, nuevos pendientes detectados.
4. **Sección 11** — estándares operativos ajustados si surgió necesidad.
5. **Sección 12** — DoD de la fase tildado punto por punto.

El `.md` se versiona en el mismo PR que el `.pbip`.

### 14.7 Cuándo volver al chat maestro (este)

La conversación actual (donde se construyó el plan) se conserva como **chat maestro de decisiones estratégicas**. Volver aquí solo para:

- Cambios estructurales al plan (agregar/quitar fases).
- Decisiones que afectan múltiples fases simultáneamente.
- Conflictos entre fases que requieren rearbitraje.
- Revisiones de cierre cuando el proyecto está terminado.

**No volver aquí para implementación de fases.** Eso va en su chat específico.

### 14.8 Manejo de errores y reinicios

Si una fase se descarrila (decisiones malas, código que no funciona, requerimientos cambiantes):

1. Documentar en §10.3 qué pasó y por qué.
2. Cerrar PR sin merge si el branch es inviable.
3. Crear nueva conversación con mensaje de apertura que referencie el intento fallido.
4. No borrar el chat anterior — sirve como registro de aprendizaje.

### 14.9 Versionado del propio plan maestro

El `.md` también evoluciona. Convención:

- Cambios al plan = commit con tipo `docs(plan): descripción`.
- Cambios mayores (nuevas secciones, reestructuración) → versionado semántico opcional en metadatos del documento:
  - v1.0 — plan inicial aprobado (estado actual)
  - v1.1 — ajustes menores entre fases
  - v2.0 — reestructuración mayor

### 14.10 Orden recomendado de ejecución de las fases

1. **Fase 0** (foundation) — cerrar primero, sin excepciones.
2. **Fase 1** (Balance Presencial) — entrega de mayor valor inmediato.
3. **Fase 2** (Export SNIES) — resuelve dolor operativo concreto. Paralelizable con Fase 5 si hay capacidad.
4. **Fase 5** (Movilidad Virtual) — clon estructural de Fase 1, bajo esfuerzo.
5. **Fase 3** (Detalle) — ajuste menor, en cualquier momento después de Fase 0.
6. **Fase 4** (Internacionales) — desbloqueada al resolver P1.
7. **Fase 6** (Financiación) — auditoría de datos primero, luego implementación.

---

## 15. Glosario

| Término | Definición |
|---|---|
| **Alcance** | Porción del universo de datos que una página analiza. Se controla con filtros de página, no con DAX. |
| **Definición intrínseca** | Filtro que es parte permanente de qué mide un KPI. Va en DAX. |
| **DRI** | Dirección de Relaciones Internacionales. Cliente interno del dashboard. |
| **Entrante (Inbound)** | Movilidad de extranjeros hacia La Sabana. `STR_CLASIFICACION_MOVILIDAD_CAL_AJT = "Entrante"`. |
| **Estudiante Regular** | Matriculado en programa académico formal de La Sabana. Origen `FCT_MATRICULADOS`. |
| **Estudiante Temporal** | Movilidad o visita corta. Origen `FCT_MOVILIDAD_ESTUDIANTE` o `FCT_VISITANTE_EXTRANJERO`. |
| **Grano** | Nivel de detalle de una fila en la FCT. Ver §4.2. |
| **Modalidad** | Presencial o Virtual. Derivada de `STR_DETALLE_ACTIVIDAD_CAL_AJT`. |
| **Movilidad** | Evento de intercambio académico. Métrica: `SUM(NUM_NUMERO_MOVILIDAD_CAL)`. |
| **Persona** | Individuo único. Métrica: `DISTINCTCOUNT(NUM_DIM_PERSONA_SK)`. |
| **Portabilidad** | Propiedad de una medida que funciona en múltiples alcances sin modificación. |
| **Saliente (Outbound)** | Movilidad de estudiantes de La Sabana hacia el exterior. |
| **SNIES** | Sistema Nacional de Información de la Educación Superior. Reporte normativo obligatorio en Colombia. |
| **Test de portabilidad** | Procedimiento de §6.2 para validar si un filtro pertenece a DAX o a panel. |

---

## Apéndice A — Plantilla de bloque de justificación

Copiar y completar al crear cada medida o aplicar cada filtro:

```markdown
### Componente: [nombre]

**Decisión:**
**Tipo de filtro:** definición / alcance / exploración
**Capa elegida:** DAX / panel de página / slicer / Calculation Group
**Por qué acá y no en otra capa:**
**Trade-off aceptado:**
**Test de portabilidad:**
```

## Apéndice B — Plantilla de Definition of Done por fase

Copiar al cerrar cada fase:

```markdown
### Cierre Fase [N] — [nombre]

- [ ] Inventario de filtros documentado
- [ ] Justificación 5-puntos por cada medida nueva
- [ ] Justificación 5-puntos por cada filtro nuevo
- [ ] Documentación según estándar §5.2 en todas las medidas
- [ ] Cuadro de texto de alcance visible en cada página
- [ ] Test de portabilidad documentado
- [ ] Anti-patrones de §9 ausentes
- [ ] PR aprobado y merged
- [ ] Sección 7 de este .md actualizada
```

---

**Fin del documento.**

> Cualquier desviación de este plan se documenta aquí, con fecha y justificación. No hay decisiones tácitas.