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
| Versión del documento | v1.4 |
| Última actualización | 2026-06-10 |
| Estado | v1.4 cerrada · P1, P4–P7 y R1 cerrados · Fases 3–8 desbloqueadas · siguiente: Fase 5 |

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
6. **(Datos existentes, no explotados todavía)** $ COP financiado para movilidad entrante en estudiantes, tipos de financiadores nacionales e internacionales. **(v1.4)** Cobertura auditada: máx. 6% (nacional) y 10% (internacional) por período; valores en múltiples monedas. Se explota como página exploratoria con salvaguardas obligatorias (§7.26, §8.10).
7. **(Agregada v1.4)** Caracterización demográfica (sexo, edad) de estudiantes en movilidad — presencial y virtual. La DRI confirmó interés en ambas modalidades. Los visuales de atributos de persona (sexo, nivel, programa) usan `[Personas]` (DISTINCTCOUNT); la distribución etaria usa `[Movilidades]` con semántica de participación (§7.20, §7.22).
8. **(Agregada v1.4)** ¿Cuál es la movilidad estudiantil nacional (destinos/orígenes dentro de Colombia) y cómo se comporta? Datos ya disponibles en la FCT.

### 3.2 Preguntas estratégicas faltantes (observación del arquitecto)

La DRI hoy usa el dashboard como **herramienta de consulta**, no de decisión estratégica. Las preguntas priorizadas son descriptivas, ninguna pregunta de causalidad, retorno o priorización. El rediseño debe responder lo que piden, pero deja espacio para insights estratégicos no solicitados todavía:

- ¿Cuál es la balanza neta entrante vs saliente y cómo evoluciona?
- ¿Quiénes son los estudiantes regulares extranjeros y cómo evoluciona la atracción?
- ¿Qué peso tiene la modalidad virtual frente a presencial?
- ¿Cuál es el perfil demográfico típico del estudiante en movilidad y varía por modalidad?
- ¿Cómo se compara la movilidad nacional con la internacional en volumen y tendencia?

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

#### 4.1.1 Campos demográficos disponibles en la FCT (agregado v1.4)

| Campo | Tipo | Ubicación | Nota |
|---|---|---|---|
| `NUM_EDAD_CAL_AJT` | Entero | `FCT_Movilidad_Estudiantil` | Edad al momento del evento. Calculada en PQ. Varía por fila — una persona puede tener edades distintas en eventos de distintos períodos. |

**Implicación de grano:** como la edad vive en la FCT (grano evento), un visual que agrupe por rango de edad podría asignar la misma persona a múltiples rangos si participó en eventos en diferentes edades. Resuelto en §7.22: se acepta la duplicación con semántica de participación, métrica `[Movilidades]`.

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
| Modalidad | `FCT_Movilidad_Estudiantil[Modalidad]` — columna materializada en PQ. "Virtual" si `STR_DETALLE_ACTIVIDAD_CAL_AJT` contiene "virtual", "Presencial" en otro caso. Confirmada v1.4. |
| Rango de Edad | Columna calculada en PQ derivada de `NUM_EDAD_CAL_AJT`, buckets de §7.23 + columna auxiliar de sort. **A implementar en Fase 7.** |

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

### 4.6 Campos demográficos disponibles (agregado v1.4)

| Campo | Ubicación | Tipo | Nota |
|---|---|---|---|
| Sexo | `DIM_PERSONA.STR_COD_GENERO_AJT` | Categórico | ~17 registros sin valor asignado detectados en mockup stakeholder (1.228 personas vs 1.211 con sexo). Nombre confirmado v1.4. |
| `NUM_EDAD_CAL_AJT` | `FCT_Movilidad_Estudiantil` | Entero | Edad al momento del evento. Ver §4.1.1 para implicación de grano. |

**Tratamiento de nulos en sexo:** los registros sin valor se muestran como categoría **"Sin dato"** en visuales demográficos. Justificación: ocultar registros sin sexo falsea el total de personas. Transparencia sobre completitud > limpieza cosmética. Ver §7.21.

### 4.7 Campos financieros disponibles (agregado v1.4)

| Campo | Ubicación | Tipo | Nota |
|---|---|---|---|
| `NUM_VALOR_FINAN_NACIONAL_CAL_AJT` | FCT | Numérico | Valor de financiación nacional del evento. Cobertura máx. 6% por período (R1 cerrado). |
| `STR_MONEDA_FINAN_NACIONAL_CAL_AJT` | FCT | Categórico | Moneda del valor nacional. `"NFD"` = sin dato de financiación. |
| `NUM_VALOR_FINAN_INTERNACIONAL_CAL_AJT` | FCT | Numérico | Valor de financiación internacional del evento. Cobertura máx. 10% por período (R1 cerrado). |
| `STR_MONEDA_FINAN_INTERNACIONAL_CAL_AJT` | FCT | Categórico | Moneda del valor internacional. `"NFD"` = sin dato de financiación. |

**Regla dura:** valores en monedas distintas **no se suman en ningún visual**. No existe tasa de cambio histórica confiable en el modelo; toda agregación monetaria se segmenta por moneda. Ver §7.26.

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

### 7.1 Decisión: 10 páginas separadas (no 4, no 8)

- **Tipo:** estructura del producto.
- **Por qué:** cada página tiene audiencia y propósito **mutuamente excluyentes**. No hay solapamiento funcional.
- **Trade-off aceptado:** más navegación, más mantenimiento. Mitigado con menú lateral persistente agrupado por bloques.
- **Alternativas descartadas:**
  - 3 páginas con tabs/bookmarks → mezcla niveles analíticos, viola principio 5.4.
  - Cortes adicionales sin pregunta propia → fragmenta la lectura ejecutiva.

> **Nota Fase 2 (2026-05-27):** el conteo subió a 7 con Export SNIES dividido en dos páginas. Ver §7.8 revisado.

> **Nota v1.4 (2026-06-09):** el conteo sube a 9. Se integran dos páginas no documentadas previamente: "Caracterización Estudiantes en Movilidad" (legado a conformar, §8.8) y "Movilidad Nacional" (nueva, §8.9).

> **Nota v1.4 — cierre (2026-06-10):** el conteo sube a 10 con la página "Financiación — Exploratoria" (§8.10, §7.26).

### 7.2 Decisión: Movilidad Virtual como página propia, no bookmark

- **Tipo:** estructura.
- **Por qué:** decisión del usuario priorizando facilidad de mantenimiento. Un bookmark compartiendo lienzo con Presencial complica versionado y debugging.
- **Trade-off aceptado:** duplicación visual entre Balance Presencial y Virtual.
- **Mitigación:** las dos páginas comparten la misma definición de visuales y las mismas medidas base. El único diferenciador es el filtro de alcance `Modalidad`.

### 7.3 Decisión: `Modalidad` como filtro de alcance en panel oculto

- **Decisión:** aplicar `FCT_Movilidad_Estudiantil[Modalidad] = "Presencial"` en panel de página oculto y bloqueado para la página Balance Presencial; `FCT_Movilidad_Estudiantil[Modalidad] = "Virtual"` para la página Virtual. Columna materializada en PQ, confirmada v1.4 (§4.3).
- **Tipo de filtro:** alcance.
- **Capa elegida:** panel de página.
- **Por qué acá y no en DAX:** la métrica "Movilidades" no cambia su definición. Cambia el universo. Crear `[Movilidades Presenciales]` y `[Movilidades Virtuales]` sería hardcodear el contexto en la medida — anti-patrón directo.
- **Trade-off aceptado:** el filtro es invisible para el usuario en el panel de filtros. Mitigación: cuadro de texto en la página documentando el alcance.
- **Test de portabilidad:** si mañana se pide un Balance mixto (Presencial + Virtual), basta con una nueva página que no aplique este filtro. Cero cambios en medidas.

### 7.4 Decisión: `STR_TABLA_ORIGEN_CAL` como filtro de alcance

- **Aplicación:**
  - Balance Presencial / Virtual → `STR_TABLA_ORIGEN_CAL IN {"FCT_MOVILIDAD_ESTUDIANTE", "FCT_VISITANTE_EXTRANJERO"}` (excluye matriculados regulares).
  - Internacionales en La Sabana → `STR_TABLA_ORIGEN_CAL = "FCT_MATRICULADOS"` + filtro adicional de país ≠ Colombia (validado — P1 cerrado: el campo país representa la nacionalidad del estudiante).
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

> **Nota:** §7.13 documenta el desvío final implementado en Fase 1 — un solo visual por top con Dirección como serie interna.

### 7.7 Decisión: dona "% bajo convenio" → tarjeta KPI

- **Tipo:** cambio de visualización.
- **Por qué:**
  - Convenio NO está entre las preguntas prioritarias de la DRI; aparece tangencialmente por SNIES.
  - Una dona de 2 categorías es un anti-patrón de visualización (un valor implica el otro).
  - Ocupa 1/6 del lienzo para una métrica binaria no prioritaria.
- **Solución:** tarjeta KPI pequeña ("57% bajo convenio") junto al resto de KPIs.

### 7.8 Decisión: Export SNIES con dos páginas independientes + navegación cruzada

> **Revisado en Fase 2 (2026-05-27).** Reemplaza decisión original de una página con bookmarks.

- **Implementación:** dos páginas independientes ("Export SNIES Saliente", "Export SNIES Entrante") con tabla plana cada una. Botones de alternancia tipo tab (Page Navigation) en la parte superior para navegar entre ellas sin pasar por Home. Entrada única "Export SNIES" desde Home que aterriza en Saliente.
- **Filtros de alcance (panel oculto):** `STR_TABLA_ORIGEN_CAL IN {FCT_MOVILIDAD_ESTUDIANTE, FCT_VISITANTE_EXTRANJERO}`, `STR_CLASIFICACION_MOVILIDAD_CAL_AJT` fijado a la dirección de la página, Año y Semestre fijados al período a reportar.
- **Slicers visibles (dropdown en línea horizontal, arriba de la tabla):** Unidad Académica, Programa Académico, Tipo de Actividad, ID Estudiante. Panel lateral reducido respecto a páginas analíticas — solo filtros útiles para validación operativa.
- **Título dinámico:** medida `_Título Período SNIES` muestra el período configurado en panel oculto para que el coordinador sepa qué período valida sin interactuar con filtros.
- **Sincronización de slicers:** activa entre las dos páginas SNIES. Desactivada con páginas analíticas.
- **Por qué dos páginas y no bookmarks:** el caso de uso incluye validación por coordinadores, no solo extracción por analista SNIES. Dos páginas fijas eliminan el riesgo de desincronización de bookmarks para un escenario donde la claridad es crítica. Cada coordinador valida típicamente una sola dirección.
- **Trade-off aceptado:** una página adicional en el archivo. Duplicación mínima — son tablas planas.
- **Desvío de §7.8 original:** justificado. El supuesto original era flujo secuencial de un analista; la realidad incluye coordinadores validando una sola dirección.

### 7.9 Decisión: medidas base mínimas, no medidas por dirección/modalidad/tipo

Habrá UNA medida `[Movilidades]` y UNA medida `[Personas]`. **No** habrá `[Movilidades Entrantes]`, `[Movilidades Virtuales]`, `[Movilidades de Matriculados]`. El contexto (página, slicer, filtro de visual) hace el resto.

Excepciones legítimas: medidas que sí codifican definición intrínseca (no contexto), como `[% bajo convenio]`, `[Razón Entrante/Saliente]`, o las medidas de cobertura de financiación (§7.26). Cada una con su bloque de justificación al crearse.

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
- **Test de portabilidad:** mismo patrón replicable en Fase 5 (Movilidad Virtual) y Fase 8 (Movilidad Nacional).

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

### 7.16 Decisión: Export SNIES — slicers en lienzo horizontal (excepción a §7.15)

- **Decisión:** las páginas Export SNIES usan slicers dropdown en línea horizontal arriba de la tabla, no en panel lateral.
- **Tipo:** excepción justificada al estándar §7.15.
- **Por qué:** son páginas de tabla plana donde el espacio horizontal es crítico para las 16–20 columnas. Un panel lateral reduciría el ancho disponible para la tabla y forzaría scroll horizontal excesivo. Además, el set de slicers es reducido (4 vs 7 de páginas analíticas) y orientado a validación operativa, no a exploración analítica.
- **Trade-off aceptado:** inconsistencia visual con páginas analíticas. Aceptable porque las páginas SNIES son bloque OPERACIÓN con audiencia y propósito distintos.

### 7.17 Decisión: Año y Semestre fijados en panel oculto para páginas SNIES

- **Decisión:** en páginas Export SNIES, Año y Semestre se configuran como filtro de alcance en panel oculto (no como slicer visible). El título dinámico (`_Título Período SNIES`) muestra el período configurado.
- **Tipo de filtro:** alcance.
- **Capa elegida:** panel de página.
- **Por qué:** el coordinador valida un período específico definido por el administrador del reporte. Exponer Año/Semestre como slicers agrega ruido — el coordinador podría cambiar el período accidentalmente y validar datos incorrectos.
- **Trade-off aceptado:** el administrador del reporte debe actualizar manualmente el filtro de panel oculto cada período. Aceptable dado que el refresh es ~2x/semestre.
- **Test de portabilidad:** patrón reutilizable si se necesitan otras páginas con período fijo.

### 7.18 Decisión: país con nombre y código ISO en tabla Export SNIES

- **Decisión:** las tablas Export SNIES muestran tanto el nombre del país como el código ISO (`STR_COD_PAIS_AJT`) para País Extranjero y País Financiador.
- **Tipo:** UX operativa.
- **Por qué:** el coordinador valida visualmente por nombre (más legible), pero el flujo SNIES downstream necesita el código ISO. Incluir ambos evita que el coordinador tenga que hacer lookup mental código↔nombre.
- **Trade-off aceptado:** una columna adicional por cada campo de país. Ancho de tabla aumenta marginalmente.

### 7.19 Decisión: Caracterización demográfica como página única con Modalidad como slicer (v1.4)

- **Decisión:** una sola página "Caracterización Estudiantes en Movilidad" cubre ambas modalidades (Presencial + Virtual). `Modalidad` se trata como **slicer de exploración** en esta página, no como filtro de alcance en panel oculto.
- **Tipo de filtro:** exploración (Modalidad).
- **Capa elegida:** slicer.
- **Alternativas descartadas:**
  - *Opción B (dos páginas por modalidad, clon estructural):* consistente con Balance/Virtual, pero la pregunta demográfica es transversal a modalidad. Duplicar la página solo para cambiar el filtro de alcance viola §9 #6 (replicar lógica de alcance en N páginas cuando un slicer lo resuelve).
  - *Opción C (integrar demografía dentro de §8.3 Movilidad Virtual):* rompe el supuesto de clon estructural de Balance, contamina dos propósitos analíticos en una página (flujo E/S + demografía), viola §9 #4.
- **Por qué Opción A:** sexo y edad son atributos del individuo, no del evento. La misma persona puede aparecer en ambas modalidades. Una página única con Modalidad como slicer permite comparación directa y absorbe futuras modalidades sin nueva página.
- **Trade-off aceptado:** rompe el patrón "una página por modalidad" usado en Balance/Virtual. Aceptable porque la pregunta demográfica es conceptualmente distinta (¿quién se mueve?) vs la pregunta de Balance (¿cuánto se mueven y hacia dónde?).
- **Test de portabilidad:** `[Personas]` es portable. Si mañana se agrega una tercera modalidad, el slicer la absorbe sin nueva página ni nueva medida.

### 7.20 Decisión: métrica por tipo de atributo en visuales demográficos (v1.4, precisada en cierre)

- **Decisión:** los visuales de **atributos del individuo** (sexo, nivel académico, unidad/programa) usan `[Personas]` (DISTINCTCOUNT sobre `STR_PERSONA_ID_NK`). El visual de **distribución etaria** usa `[Movilidades]` — excepción documentada en §7.22, porque en este modelo la edad es atributo del evento (vive en la FCT) y la semántica elegida es participación.
- **Tipo:** definición intrínseca de visuales demográficos.
- **Por qué:** para atributos estables del individuo, usar `[Movilidades]` (COUNTROWS) inflaría la representación de personas con múltiples eventos — una persona con 3 movilidades contaría 3 veces en la distribución de sexo. Eso responde "¿cuántas movilidades por sexo?", no "¿cómo se distribuyen las personas por sexo?" (lo que la DRI pide). Para la edad, la relación se invierte: el atributo varía por evento, y la pregunta confirmada es "¿a qué edades se participa?" (§7.22).
- **Trade-off aceptado:** dos métricas conviven en la misma página. Mitigación: los títulos de visual hacen explícita la unidad ("Personas por..." vs "Participaciones por...").
- **Test de portabilidad:** ambas medidas son portables y funcionan en cualquier contexto de filtro (período, modalidad, dirección).

### 7.21 Decisión: nulos de sexo se muestran como "Sin dato" (v1.4)

- **Decisión:** registros sin valor de sexo en `DIM_PERSONA.STR_COD_GENERO_AJT` se muestran como categoría visible **"Sin dato"** en los visuales demográficos.
- **Tipo:** UX / integridad de datos.
- **Por qué:** ocultar ~17 registros sin sexo falsearía el total de personas. El directivo vería [Personas] = 1.228 en la tarjeta KPI pero solo 1.211 sumadas en el gráfico de sexo. La discrepancia genera desconfianza. Transparencia sobre completitud > limpieza cosmética.
- **Trade-off aceptado:** una categoría visual adicional ("Sin dato") en el gráfico. Impacto visual mínimo dado el volumen bajo.
- **Test de portabilidad:** tratamiento aplicable a cualquier campo demográfico con nulos.

### 7.22 Decisión: edad por evento — se acepta duplicación (v1.4, cerrada)

- **Decisión:** el visual de distribución por rango de edad usa `NUM_EDAD_CAL_AJT` del evento **sin desambiguar por persona**. Una persona que participó a distintas edades cuenta en cada rango correspondiente. Opción C confirmada.
- **Semántica elegida:** "¿a qué edades se participa en movilidad?" — no "¿qué edad tienen los participantes?".
- **Métrica del visual:** `[Movilidades]` (COUNTROWS), no `[Personas]`. El eje es rango de edad, la métrica es participaciones por rango.
- **Trade-off aceptado:** la suma del gráfico de edad puede ser mayor que la tarjeta KPI `[Personas]`. Mitigación: el título del visual dice **"Participaciones por rango de edad"**, no "Personas por rango de edad". Esto alinea la expectativa del directivo con la semántica real.
- **Alternativas descartadas:**
  - *Opción A (edad del último evento):* respondería "¿qué edad tienen hoy los participantes?" — pregunta válida pero no la que la DRI prioriza. Requiere columna PQ auxiliar.
  - *Opción B (edad del primer evento):* respondería "¿a qué edad empezaron?" — relevante pero no prioritario. Misma complejidad que A.
- **Impacto técnico:** cero complejidad adicional. No requiere columna PQ auxiliar ni medida DAX nueva. Solo el campo `NUM_EDAD_CAL_AJT` agrupado por rango.
- ~~**Pendiente asociado:** P4 (§10.1).~~ **P4 cerrado.**

### 7.23 Decisión: rangos de edad — buckets confirmados (v1.4, cerrada)

- **Decisión:** los rangos de edad para el visual de distribución son los siguientes, confirmados por la DRI:

| Bucket | Rango | Nota |
|---|---|---|
| 15–17 | `NUM_EDAD_CAL_AJT` entre 15 y 17 | Menores de edad en movilidad. |
| 18–25 | entre 18 y 25 | Pregrado típico. Grupo dominante esperado. |
| 26–30 | entre 26 y 30 | Posgrado joven / final de pregrado tardío. |
| 31–40 | entre 31 y 40 | Posgrado / educación continua. |
| 41–50 | entre 41 y 50 | Educación continua / ejecutiva. |
| Mayor 50 | 51+ | — |
| Revisar | Todo lo demás | Red de seguridad: nulos, ceros, negativos, valores atípicos (>120, <15). Señal de calidad de datos. |

- **Tipo:** regla de negocio.
- **Implementación:** columna calculada en PQ sobre la FCT, derivada de `NUM_EDAD_CAL_AJT`. Lógica `if/else` que asigna el bucket como texto. El orden visual en el gráfico debe respetar el orden lógico de la tabla (no alfabético).
- **Ordenamiento:** columna auxiliar numérica de sort (`1` a `7`) para forzar orden correcto en el eje del visual (Sort By Column). Sin esta columna, Power BI ordenaría alfabéticamente — la columna de sort lo hace explícito y robusto.
- ~~**Pendiente asociado:** P6 (§10.1).~~ **P6 cerrado.**

### 7.24 Decisión: Movilidad Nacional como clon de Balance con swap geográfico→programa (v1.4, cerrada)

- **Decisión:** se agrega una página "Movilidad Nacional" al bloque ANÁLISIS. Es **clon estructural de Balance** (§8.2) con un único cambio: el top geográfico (países) se reemplaza por **Top Programas Académicos** (Unidad Académica → Programa, con drilldown). Filtro de alcance: `DIM_UBICACION_GEOGRAFICA_MOVILIDAD[STR_PAIS_AJT] = "Colombia"`.
- **Tipo:** estructura del producto.
- **Alcance (panel oculto):** `DIM_UBICACION_GEOGRAFICA_MOVILIDAD[STR_PAIS_AJT] = "Colombia"` AND `STR_TABLA_ORIGEN_CAL IN {FCT_MOVILIDAD_ESTUDIANTE, FCT_VISITANTE_EXTRANJERO}`.
- **Por qué clon + swap:** la movilidad nacional no tiene granularidad geográfica debajo de país (departamento mal diligenciado — restricción de calidad de datos documentada). Un top de países mostraría solo "Colombia". En cambio, el programa académico de La Sabana responde una pregunta que ninguna otra página cubre: "¿de qué programas salen y a qué programas llegan en movilidad nacional?".
- **Programas:** solo de La Sabana (no de la institución externa). El campo usa la misma jerarquía Unidad Académica → Programa que Caracterización (§8.8), pero con métrica `[Movilidades]`.
- **Trade-off aceptado:** la página deja de ser clon puro (tiene un visual distinto), pero la diferencia es mínima: un solo visual cambia de eje, no de métrica ni de tipo de gráfico.
- **Restricción de calidad documentada:** `DIM_UBICACION_GEOGRAFICA_MOVILIDAD` no tiene departamento/ciudad con diligenciamiento confiable. Si se corrige en el futuro, se puede agregar un top geográfico sin romper nada.
- **Filtros (slicers):** configuración a validar durante implementación (Fase 8) para evitar redundancia con Balance y otras páginas. Ver §10.3.
- **Test de portabilidad:** `[Movilidades]` y `[Personas]` funcionan idénticas con el nuevo filtro de alcance. Cero medidas nuevas.

### 7.25 Decisión: página de Caracterización existente es legado a conformar (v1.4)

- **Decisión:** la página "Caracterización Estudiantes en Movilidad" que existe actualmente en el dashboard es legado previo al rediseño. Se conforma a los estándares del plan (cuadro de texto de alcance, documentación de filtros, medidas portables, naming) y se extiende para cubrir ambas modalidades (§7.19) durante Fase 7.
- **Tipo:** conformación + extensión.
- **Estado actual:** funcional pero no documentada en el plan. Alcance actual: temporales presenciales. Construida fuera de las convenciones del plan.
- **Acción en Fase 7:** auditar visuales y medidas contra estándares §5.2, §9, §11. Reemplazar medidas no portables si existen. Agregar Modalidad como slicer. Agregar cuadro de texto de alcance.

### 7.26 Decisión: Financiación como página exploratoria multi-moneda (v1.4, cerrada)

- **Decisión:** la financiación se explota en **página propia marcada como "Exploratoria"** (§8.10), estructurada en dos capas: conteo de movilidades financiadas (confiable) y valores segmentados por moneda (exploratorio). **Nunca se suman valores entre monedas.**
- **Tipo:** estructura del producto + integridad de datos.
- **Por qué:** cobertura máxima auditada de 6% (nacional) y 10% (internacional) por período (R1 cerrado). Mostrar montos sin salvaguarda induciría decisiones sobre datos que representan una fracción mínima del universo. Además los valores viven en múltiples monedas — un KPI agregado entre monedas sería matemáticamente inválido.
- **Medidas de cobertura (definición intrínseca, van en DAX):** `[Movilidades Financiadas Nacional]`, `[% Cobertura Financiación Nacional]`, `[Movilidades Financiadas Internacional]`, `[% Cobertura Financiación Internacional]`. El filtro `STR_MONEDA_FINAN_* <> "NFD"` **define** qué es "financiada" — es el numerador del KPI, no alcance de página. Pasa el test de portabilidad (§6.2): las medidas funcionan idénticas en cualquier página o contexto de filtro. Carpeta `_Medidas/Financiación/`.
- **Salvaguardas obligatorias:**
  1. Cuadro de texto de alcance reforzado con advertencia explícita (texto en §8.10).
  2. Todo valor monetario visible va acompañado del % de cobertura del contexto seleccionado.
  3. Título de página con marca "Exploratoria".
  4. Prohibido todo visual que agregue valores de monedas distintas.
- **Alternativas descartadas:**
  - *No construir la página:* la DRI la solicita y los datos, aunque parciales, existen. Negar acceso es peor que entregar con salvaguardas.
  - *Convertir a moneda única:* no hay tasa de cambio histórica confiable en el modelo. Introducir conversiones inventadas es peor que segmentar por moneda.
  - *Integrar visuales financieros en Balance:* contaminaría una página con datos completos con métricas de cobertura 6–10% — el contraste de confiabilidad dentro de una misma página confunde. Página separada aísla el riesgo.
- **Trade-off aceptado:** página con 8 visuales (uno sobre el límite §9 #4), justificado porque 4 son tarjetas KPI en una sola fila y la dualidad nacional/internacional duplica necesariamente los visuales de valor.
- **Test de portabilidad:** medidas de cobertura portables. Si mañana se audita y mejora la captura del dato financiero, la página no cambia — solo mejoran sus números.
- ~~**Riesgo asociado:** R1 (§10.2).~~ **R1 cerrado.**

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
│  ├─ 4. Internacionales en La Sabana                     │
│  ├─ 5. Caracterización Estudiantes en Movilidad         │
│  ├─ 6. Movilidad Nacional                               │
│  └─ 7. Financiación (Exploratoria)                      │
│                                                          │
│  OPERACIÓN                                               │
│  ├─ 8. Detalle Movilidad                                │
│  ├─ 9. Export SNIES Saliente                            │
│  └─ 10. Export SNIES Entrante                           │
└─────────────────────────────────────────────────────────┘
```

> **Nota:** desde Home, Export SNIES se muestra como entrada única que aterriza en Saliente. La navegación interna Saliente↔Entrante se hace con botones tipo tab dentro de las páginas SNIES.

> **Nota v1.4:** Caracterización (5) es legado a conformar + extender (Fase 7). Movilidad Nacional (6) es página nueva (Fase 8). Financiación (7) es página nueva exploratoria (Fase 6). El conteo final es 10 páginas. Ver §7.1 revisado.

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
- **Alcance (panel oculto):** país ≠ Colombia, todos los orígenes. **(v1.4)** Validado: el campo país representa la nacionalidad del estudiante (P1 cerrado, §10.1) — el filtro identifica correctamente a los extranjeros.
- **Visuales clave:**
  - KPIs: total internacionales, desglose Regular vs Temporal.
  - Mapa de países origen.
  - Distribución por programa / nivel.
  - Evolución temporal de atracción.
  - Top instituciones de procedencia (solo Temporales).

### 8.5 Detalle Movilidad *(ajuste menor sobre lo existente)*

- **Audiencia:** directivos (consulta puntual).
- **Estado actual:** página de imagen 3, dos visuales (consulta + descargue).
- **Ajuste a realizar:** unificar en una tabla, con selector Entrante/Saliente que cambia el set de columnas visibles (entrante y saliente tienen columnas distintas según naturaleza del registro).

### 8.6 Export SNIES Saliente *(nueva — Fase 2)*

- **Audiencia:** coordinadores (validación) + analistas SNIES (extracción).
- **Propósito:** tabla plana con datos esenciales para flujo de homologación SNIES. No es el reporte SNIES final — es el insumo desde Power BI.
- **Alcance (panel oculto):** `STR_TABLA_ORIGEN_CAL IN {FCT_MOVILIDAD_ESTUDIANTE, FCT_VISITANTE_EXTRANJERO}` AND `STR_CLASIFICACION_MOVILIDAD_CAL_AJT = "Saliente"` AND Año/Semestre fijados al período a reportar.
- **Título dinámico:** medida `_Título Período SNIES` muestra período configurado.
- **Cuadro de texto visible:** "Datos de movilidad estudiantil saliente para reporte SNIES. Período fijado por el administrador del reporte. Los datos mostrados son insumo para el flujo de homologación SNIES — no constituyen el reporte final."

**Navegación:** botones tipo tab (Page Navigation). Saliente activo (`#2E5597`, texto blanco), Entrante inactivo (`#F2F2F2`, texto `#595959`).

**Slicers visibles (dropdown en línea horizontal):**

| Slicer | Campo | Origen |
|---|---|---|
| Unidad Académica | campo unidad | `DIM_PROGRAMA_ACADEMICO` |
| Programa Académico | campo programa | `DIM_PROGRAMA_ACADEMICO` |
| Tipo de Actividad | `STR_DETALLE_ACTIVIDAD_CAL_AJT` | FCT |
| ID Estudiante | `STR_PERSONA_ID_NK` | FCT |

**Columnas de tabla (orden SNIES):**

| # | Columna visible | Campo modelo | Origen |
|---|---|---|---|
| 1 | ID Estudiante | `STR_PERSONA_ID_NK` | FCT |
| 2 | Tipo Documento | `STR_COD_TIPO_ID_CAL_AJT` | FCT |
| 3 | Número Documento | `STR_NUM_ID_AJT` | `DIM_PERSONA` |
| 4 | País Extranjero (nombre) | campo nombre país | `DIM_UBICACION_GEOGRAFICA` |
| 5 | País Extranjero (código) | `STR_COD_PAIS_AJT` | `DIM_UBICACION_GEOGRAFICA` |
| 6 | Institución Extranjera | `STR_NOMBRE_ENTIDAD_EXTERNA_CAL_AJT` | FCT |
| 7 | Tipo Movilidad | `STR_DETALLE_ACTIVIDAD_CAL_AJT` | FCT |
| 8 | Días Movilidad | `NUM_DURACION_DIAS_CAL_AJT` | FCT |
| 9 | Movilidad por Convenio | `STR_DATOS_CONVENIO_CAL_AJT` | FCT |
| 10 | Código Convenio | `NUM_CODIGO_CONVENIO_NK` | `DIM_CONVENIO` |
| 11 | Fuente Nacional | `STR_DESC_FUENTE_NACIONAL_INVESTIG_CAL_AJT` | FCT |
| 12 | Valor Financiación Nacional | `NUM_VALOR_FINAN_NACIONAL_CAL_AJT` | FCT |
| 13 | Moneda Nacional | `STR_MONEDA_FINAN_NACIONAL_CAL_AJT` | FCT |
| 14 | Fuente Internacional | `STR_DESC_FUENTE_INTERNACIONAL_CAL_AJT` | FCT |
| 15 | País Financiador (nombre) | campo nombre país | `DIM_UBICACION_GEOGRAFICA_FINANCIADOR` |
| 16 | País Financiador (código) | `STR_COD_PAIS_AJT` | `DIM_UBICACION_GEOGRAFICA_FINANCIADOR` |
| 17 | Valor Financiación Internacional | `NUM_VALOR_FINAN_INTERNACIONAL_CAL_AJT` | FCT |
| 18 | Moneda Internacional | `STR_MONEDA_FINAN_INTERNACIONAL_CAL_AJT` | FCT |

**Configuración tabla visual:**
- Tipo: Table (no Matrix). Sin totales, sin subtotales.
- Tamaño fuente: 10pt cuerpo, 11pt encabezados.
- Alternating rows: `#F2F2F2` / blanco.
- Grid lines verticales activas.
- Formato numérico: sin formato especial — datos crudos.
- Exportación: nativa de Power BI (clic derecho → Export data).

### 8.7 Export SNIES Entrante *(nueva — Fase 2)*

- **Audiencia, propósito, alcance:** idénticos a §8.6 excepto `STR_CLASIFICACION_MOVILIDAD_CAL_AJT = "Entrante"`.
- **Navegación:** botones invertidos — Entrante activo, Saliente inactivo.
- **Cuadro de texto visible:** "Datos de movilidad estudiantil entrante para reporte SNIES. Período fijado por el administrador del reporte. Los datos mostrados son insumo para el flujo de homologación SNIES — no constituyen el reporte final."

**Columnas de tabla (orden SNIES — agrega nombres entre Número Documento y País):**

| # | Columna visible | Campo modelo | Origen |
|---|---|---|---|
| 1 | ID Estudiante | `STR_PERSONA_ID_NK` | FCT |
| 2 | Tipo Documento | `STR_COD_TIPO_ID_CAL_AJT` | FCT |
| 3 | Número Documento | `STR_NUM_ID_AJT` | `DIM_PERSONA` |
| 4 | Primer Nombre | `STR_PRIMER_NOMBRE_AJT` | `DIM_PERSONA` |
| 5 | Segundo Nombre | `STR_SEGUNDO_NOMBRE_AJT` | `DIM_PERSONA` |
| 6 | Primer Apellido | `STR_PRIMER_APELLIDO_AJT` | `DIM_PERSONA` |
| 7 | Segundo Apellido | `STR_SEGUNDO_APELLIDO_AJT` | `DIM_PERSONA` |
| 8 | País Extranjero (nombre) | campo nombre país | `DIM_UBICACION_GEOGRAFICA` |
| 9 | País Extranjero (código) | `STR_COD_PAIS_AJT` | `DIM_UBICACION_GEOGRAFICA` |
| 10 | Institución Extranjera | `STR_NOMBRE_ENTIDAD_EXTERNA_CAL_AJT` | FCT |
| 11 | Tipo Movilidad | `STR_DETALLE_ACTIVIDAD_CAL_AJT` | FCT |
| 12 | Días Movilidad | `NUM_DURACION_DIAS_CAL_AJT` | FCT |
| 13 | Movilidad por Convenio | `STR_DATOS_CONVENIO_CAL_AJT` | FCT |
| 14 | Código Convenio | `NUM_CODIGO_CONVENIO_NK` | `DIM_CONVENIO` |
| 15 | Fuente Nacional | `STR_DESC_FUENTE_NACIONAL_INVESTIG_CAL_AJT` | FCT |
| 16 | Valor Financiación Nacional | `NUM_VALOR_FINAN_NACIONAL_CAL_AJT` | FCT |
| 17 | Moneda Nacional | `STR_MONEDA_FINAN_NACIONAL_CAL_AJT` | FCT |
| 18 | Fuente Internacional | `STR_DESC_FUENTE_INTERNACIONAL_CAL_AJT` | FCT |
| 19 | País Financiador (nombre) | campo nombre país | `DIM_UBICACION_GEOGRAFICA_FINANCIADOR` |
| 20 | País Financiador (código) | `STR_COD_PAIS_AJT` | `DIM_UBICACION_GEOGRAFICA_FINANCIADOR` |
| 21 | Valor Financiación Internacional | `NUM_VALOR_FINAN_INTERNACIONAL_CAL_AJT` | FCT |
| 22 | Moneda Internacional | `STR_MONEDA_FINAN_INTERNACIONAL_CAL_AJT` | FCT |

Misma configuración de tabla visual que §8.6.

### 8.8 Caracterización Estudiantes en Movilidad *(legado a conformar + extender — v1.4)*

- **Audiencia:** directivos + DRI.
- **Pregunta:** ¿Cuál es el perfil demográfico (sexo, edad, nivel académico) de los estudiantes en movilidad? (§3.1 P7)
- **Alcance (panel oculto):** `STR_TABLA_ORIGEN_CAL IN {FCT_MOVILIDAD_ESTUDIANTE, FCT_VISITANTE_EXTRANJERO}` (excluye matriculados regulares). **Sin filtro de Modalidad en panel** — Modalidad es slicer de exploración (§7.19).
- **Cuadro de texto visible:** "Esta página analiza el perfil demográfico de estudiantes temporales en movilidad (entrantes y salientes). Incluye modalidad presencial y virtual. Excluye estudiantes regulares matriculados. Use el filtro Modalidad para comparar."
- **Estado actual:** página legado existente en el dashboard. Funcional pero construida fuera de las convenciones del plan. Alcance actual limitado a temporales presenciales. No tiene cuadro de texto de alcance.

**Estructura visual confirmada (v1.4):**

Layout sobre canvas 1920×1080 (área útil descontando panel lateral de slicers y barra inferior):

```
┌──────────────────────────────────────────────────────────┐
│  [Personas]        [Movilidades]    [Prom. Duración]     │ 65px
├──────────────────────────────────────────────────────────┤
│  Distribución por sexo    │  Participaciones por rango   │ 260px
│  (barras horizontales)    │  de edad (barras horiz.)     │
├──────────────────────────────────────────────────────────┤
│  Nivel académico          │  Unidad Académica → Programa │ 260px
│  (barras horizontales)    │  (barras horiz. + drilldown) │
└──────────────────────────────────────────────────────────┘
```

**Fila 1 — KPIs (65px):**

| KPI | Medida | Nota |
|---|---|---|
| Personas | `[Personas]` | Métrica central de la página. |
| Movilidades | `[Movilidades]` | Siempre junto a Personas (§4.2). Permite lectura de ratio movilidades/persona. |
| Promedio duración (días) | `AVERAGE(NUM_DURACION_DIAS_CAL_AJT)` | Medida nueva `[Promedio Duración Días]` a documentar según §5.2 en Fase 7. Caracteriza la experiencia, no solo al individuo. |

**Fila 2 — Perfil demográfico (260px):**

| Visual | Métrica | Tipo | Nota |
|---|---|---|---|
| Distribución por sexo | `[Personas]` por `DIM_PERSONA[STR_COD_GENERO_AJT]` | Clustered Bar horizontal | Incluye "Sin dato" para nulos (§7.21). No dona — §9 #5. |
| Participaciones por rango de edad | `[Movilidades]` por rango de `NUM_EDAD_CAL_AJT` | Clustered Bar horizontal | Título dice "Participaciones", no "Personas" (§7.22). Rangos confirmados (§7.23). |

**Fila 3 — Contexto académico (260px):**

| Visual | Métrica | Tipo | Nota |
|---|---|---|---|
| Nivel académico | `[Personas]` por nivel (Pregrado/Posgrado) | Clustered Bar horizontal | Máximo 2–3 categorías. |
| Unidad Académica → Programa | `[Personas]` por Unidad Académica (drilldown a Programa Académico) | Clustered Bar horizontal con jerarquía nativa | Descendente por [Personas]. Drilldown nativo de PBI, sin medida nueva. |

**Total: 7 visuales (3 KPIs + 4 gráficos).** Dentro del límite §9 #4.

**Visuales del legado descartados:**

| Visual legado | Veredicto | Razón |
|---|---|---|
| Países involucrados (KPI) | ❌ | Cobertura geográfica, no demografía. Vive en §8.1. |
| Tipos de actividad (lista) | ❌ | Ya respondido en Balance (Top Tipos, §8.2 Fila 3). Duplicaría información. |
| Evolución temporal | ❌ | Ya en Balance (§8.2), Virtual (§8.3), Nacional (§8.9). No aporta pregunta nueva acá. |

**Elementos transversales:**
- Paleta E/S consistente: `#2E5597` Entrante, `#BDD7EE` Saliente.
- Subtítulo dinámico heredado de plantilla.
- Cuadro de texto de alcance visible bajo subtítulo.
- Slicers: panel lateral estándar (§11.5.1) + **Modalidad** como slicer adicional al final del panel.

**Acciones de conformación (Fase 7):**
1. Auditar medidas existentes contra §5.2, §9, §11.
2. Reemplazar medidas no portables si existen.
3. Agregar Modalidad como slicer de exploración.
4. Agregar cuadro de texto de alcance.
5. Remover filtro de Modalidad del panel oculto (si existe) y delegar al slicer.
6. ~~Definir rangos de edad con DRI (P6).~~ Cerrado: 7 buckets confirmados (§7.23). Implementar columna PQ + sort.
7. ~~Resolver desambiguación edad-persona (§7.22, P4).~~ Cerrado: Opción C, se acepta duplicación.

### 8.9 Movilidad Nacional *(nueva — v1.4, clon de Balance con swap geográfico→programa)*

- **Audiencia:** directivos + DRI.
- **Pregunta:** ¿Cuál es la movilidad estudiantil nacional (dentro de Colombia) y cómo se comporta? (§3.1 P8)
- **Alcance (panel oculto):** `DIM_UBICACION_GEOGRAFICA_MOVILIDAD[STR_PAIS_AJT] = "Colombia"` AND `STR_TABLA_ORIGEN_CAL IN {FCT_MOVILIDAD_ESTUDIANTE, FCT_VISITANTE_EXTRANJERO}`.
- **Cuadro de texto visible:** "Esta página analiza movilidad estudiantil temporal con destino/origen dentro de Colombia. Excluye estudiantes regulares matriculados y movilidad internacional."

**Estructura visual confirmada — clon de §8.2 con un swap (layout idéntico a Balance):**

```
┌──────────────────────────────────────────────────────────┐
│  [Movilidades]       [Personas]       [% Convenio]       │ 65px
├──────────────────────────────────────────────────────────┤
│                                                          │
│            Evolución temporal (líneas E/S)                │ 240px
│                                                          │
├──────────────────┬──────────────────┬────────────────────┤
│  Top Programas   │  Top Tipos Mov   │  Top Instituciones │ 280px
│  (UA→Programa    │  (E + S juntos)  │  (E + S juntos)    │
│   drilldown)     │                  │                    │
└──────────────────┴──────────────────┴────────────────────┘
```

| Fila | Visual | Métrica | Tipo | Nota |
|---|---|---|---|---|
| 1 — KPIs | Movilidades | `[Movilidades]` | Tarjeta | Portable, cero cambios. |
| 1 — KPIs | Personas | `[Personas]` | Tarjeta | Portable, cero cambios. |
| 1 — KPIs | % Bajo Convenio | `[% Bajo Convenio]` | Tarjeta | Portable, cero cambios. |
| 2 — Evolución | Evolución E/S | `[Movilidades]` por período | Línea con series E/S | Eje categórico `YYYY-N` (§11.4.4). Configuración idéntica a §8.2. |
| 3 — Tops | **Top Programas Académicos** | `[Movilidades]` por Unidad Académica → Programa | Clustered Bar horizontal + drilldown | **SWAP**: reemplaza Top Países de Balance. Solo programas La Sabana. Descendente por [Movilidades]. Dirección como serie interna (§7.13); interacción Top N + jerarquía a validar en Fase 8. |
| 3 — Tops | Top Tipos de Movilidad | `[Movilidades]` por tipo | Clustered Bar horizontal | Idéntico a Balance (§7.13, §7.14). |
| 3 — Tops | Top Instituciones | `[Movilidades]` por institución | Clustered Bar horizontal | Instituciones colombianas. Idéntico a Balance. |

**Total: 7 visuales (3 KPIs + 4 gráficos).** Idéntico a Balance excepto el swap en Fila 3.

**Medidas nuevas requeridas:** cero. Reutilización total de medidas portables.

**Restricción de calidad documentada:** `DIM_UBICACION_GEOGRAFICA_MOVILIDAD` no tiene departamento/ciudad con diligenciamiento confiable para movilidad nacional. Si se corrige en el futuro, se puede agregar un top geográfico subnacional sin romper la página.

**Slicers:** panel lateral estándar (§11.5.1). Configuración final de filtros a validar durante implementación (Fase 8) para evitar redundancia con Balance y otras páginas — en particular el slicer País, que es tautológico en esta página.

**Nota sobre solapamiento con Balance (§7.24, §10.3):** Balance Presencial y Virtual no filtran por país — incluyen eventos nacionales. Esta página sí filtra explícitamente a Colombia. No se modifica el alcance de páginas cerradas.

### 8.10 Financiación de Movilidad — Exploratoria *(nueva — v1.4)*

- **Audiencia:** DRI (uso interno; no respalda decisiones presupuestales).
- **Pregunta:** ¿Cuántas movilidades reciben financiación, de qué tipo, y qué montos están registrados? (§3.1 P6)
- **Alcance (panel oculto):** `STR_TABLA_ORIGEN_CAL IN {FCT_MOVILIDAD_ESTUDIANTE, FCT_VISITANTE_EXTRANJERO}`.
- **Cuadro de texto reforzado (obligatorio, §7.26):** "⚠️ Página exploratoria — datos financieros con cobertura limitada: hasta 6% en financiación nacional y hasta 10% en internacional por período. Las cifras representan registros diligenciados, no el total real. Valores en múltiples monedas — no se suman entre sí. No usar como base para decisiones presupuestales."
- **Título de página:** incluye la marca "Exploratoria".

**Estructura visual confirmada (v1.4) — dos capas: conteo (confiable) y valor (exploratorio):**

```
┌──────────────────────────────────────────────────────────┐
│ [Mov. Fin. Nal] [% Cob. Nal] [Mov. Fin. Intl] [% Cob. Intl] │ 65px
├──────────────────────────────────────────────────────────┤
│  Financiación nacional     │  Financiación internacional │ 240px
│  por moneda (tabla)        │  por moneda (tabla)         │
├──────────────────────────────────────────────────────────┤
│  Mov. financiadas por      │  Mov. financiadas por       │ 240px
│  UA → Programa (barras)    │  dirección E/S (barras)     │
└──────────────────────────────────────────────────────────┘
```

**Fila 1 — KPIs de cobertura (lo confiable):**

| KPI | Medida | Nota |
|---|---|---|
| Movilidades con financiación nacional | `[Movilidades Financiadas Nacional]` = COUNTROWS con `STR_MONEDA_FINAN_NACIONAL_CAL_AJT <> "NFD"` | Medida nueva, definición intrínseca (§7.26). |
| % Cobertura nacional | `[% Cobertura Financiación Nacional]` = financiadas nal. / `[Movilidades]` | Transparencia de completitud. |
| Movilidades con financiación internacional | `[Movilidades Financiadas Internacional]` = COUNTROWS con `STR_MONEDA_FINAN_INTERNACIONAL_CAL_AJT <> "NFD"` | Medida nueva, definición intrínseca. |
| % Cobertura internacional | `[% Cobertura Financiación Internacional]` = financiadas intl. / `[Movilidades]` | — |

**Fila 2 — Valores por moneda (lo exploratorio):**

| Visual | Métrica | Tipo | Nota |
|---|---|---|---|
| Financiación nacional por moneda | SUM(`NUM_VALOR_FINAN_NACIONAL_CAL_AJT`) por `STR_MONEDA_FINAN_NACIONAL_CAL_AJT` | Table/Matrix | Excluye "NFD". Una fila por moneda. **Sin fila de total general.** |
| Financiación internacional por moneda | SUM(`NUM_VALOR_FINAN_INTERNACIONAL_CAL_AJT`) por `STR_MONEDA_FINAN_INTERNACIONAL_CAL_AJT` | Table/Matrix | Ídem. |

**Fila 3 — Contexto por conteo (sin problema de moneda):**

| Visual | Métrica | Tipo | Nota |
|---|---|---|---|
| Movilidades financiadas por Unidad Académica → Programa | Conteo de financiadas (nal. o intl.) | Clustered Bar horizontal + drilldown | Responde "¿qué programas reciben más apoyo?" con conteo, no valor. |
| Movilidades financiadas por dirección E/S | Conteo de financiadas por dirección | Clustered Bar horizontal | ¿Se financia más la saliente o la entrante? |

**Total: 8 visuales (4 KPIs + 2 tablas + 2 gráficos).** Excepción justificada al límite §9 #4 (ver §7.26).

**Prohibiciones (§7.26):** ningún visual suma valores entre monedas; ningún KPI de "total financiado" agregado; ningún valor monetario sin su % de cobertura visible en el mismo contexto.

**Medidas nuevas:** 4 de cobertura (carpeta `_Medidas/Financiación/`), documentadas según §5.2 con bloque de justificación 5-puntos. Detalle fino de layout y formatos se ajusta en Fase 6.

**Slicers:** panel lateral estándar (§11.5.1). Configuración final a validar en Fase 6.

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
| 9 | Agregar valores monetarios de monedas distintas | Todo visual financiero se segmenta por moneda (§7.26). Un "total" inter-moneda es matemáticamente inválido. |

---

## 10. Pendientes y riesgos abiertos

### 10.1 Pendientes bloqueantes

| # | Pendiente | Bloquea fase | Responsable |
|---|---|---|---|
| ~~P1~~ | ✅ **Cerrado (v1.4).** El campo país en `DIM_UBICACION_GEOGRAFICA_MOVILIDAD` para registros de `FCT_MATRICULADOS` representa la **nacionalidad** del estudiante. Filtrar por `STR_PAIS_AJT ≠ "Colombia"` es válido para identificar estudiantes regulares extranjeros en Fase 4 (Internacionales en La Sabana). | — | — |
| ~~P2~~ | ✅ **Cerrado en Fase 2 (2026-05-27).** Redefinido: la página Export SNIES expone campos esenciales crudos (`STR_DETALLE_ACTIVIDAD_CAL_AJT`, `STR_COD_PAIS_AJT`, etc.) como insumo para un flujo de homologación externo. No se requiere tabla de homologación dentro del modelo Power BI. El mapeo modelo→SNIES queda documentado en §8.6 y §8.7. | — | — |
| ~~P3~~ | ✅ **Cerrado en Fase 0 (2026-05-15).** Resuelto trivialmente: `STR_DATOS_CONVENIO_CAL_AJT` ya es binario "Si"/"No" en el origen. No requiere columna calculada ni binarización. | — | — |
| ~~P4~~ | ✅ **Cerrado (v1.4).** Opción C: se acepta duplicación. La semántica es "¿a qué edades se participa?". Visual usa `[Movilidades]` por rango de edad, título dice "Participaciones". Ver §7.22. | — | — |
| ~~P5~~ | ✅ **Cerrado (v1.4).** Columna confirmada: `DIM_PERSONA.STR_COD_GENERO_AJT`. | — | — |
| ~~P6~~ | ✅ **Cerrado (v1.4).** Rangos confirmados: 15–17, 18–25, 26–30, 31–40, 41–50, Mayor 50, Revisar. Ver §7.23. | — | — |
| ~~P7~~ | ✅ **Cerrado (v1.4).** Filtro: `DIM_UBICACION_GEOGRAFICA_MOVILIDAD[STR_PAIS_AJT] = "Colombia"`. No hay departamento/ciudad confiable — restricción de calidad documentada. Top geográfico reemplazado por Top Programas Académicos (La Sabana). Ver §7.24. | — | — |

> **Estado v1.4 (cierre):** no quedan pendientes bloqueantes abiertos.

### 10.2 Riesgos abiertos

| # | Riesgo | Mitigación |
|---|---|---|
| ~~R1~~ | ✅ **Cerrado (v1.4).** Auditoría de cobertura realizada: máximo 6% (financiación nacional) y 10% (internacional) por período. Multi-moneda confirmada en `STR_MONEDA_FINAN_*` (`"NFD"` = sin dato). | Página Financiación — Exploratoria con salvaguardas obligatorias: disclaimer reforzado, % de cobertura junto a todo valor, segmentación estricta por moneda, marca "Exploratoria" (§7.26, §8.10). |
| R2 | Nombres internos del modelo (`STR_NOMBRE_ENTIDAD_EXTERNA_CAL_AJT`) vs nombres normativos SNIES (`INSTITUCION_EXTRANJERA`). | Mapeo en capa de visual (renombrado en columna del visual), no en modelo. No rompe DAX existente. |
| R3 | 15k registros es bajo volumen; refresh 2x/semestre. Riesgo mínimo de performance, pero validar al final. | Test de performance en Fase 0 (baseline §11.7). Re-medir si una página nueva degrada percepción. |
| R4 | Cobertura Internacional (página existente) podría usar medidas que se modifiquen en fases futuras. | Auditar dependencias antes de tocar medidas existentes. |

### 10.3 Decisiones diferidas

- **Drill-through entre páginas:** se decide en Fase 1, una vez establecida la página Balance. **Diferido a Fase 3** — drill-through hacia Detalle Movilidad se construye cuando Detalle esté rediseñado, para evitar dependencia frágil sobre página que va a cambiar.
- **Sincronización de slicers entre páginas:** se decide en Fase 0, junto con la convención de slicers.
- **Tooltip pages ranking completo (tops Fase 1):** diferido a post-reunión stakeholder. Validar si Top 5 satisface requerimiento o si se necesita profundidad adicional. Si se requiere → construir 3 tooltip pages ocultas (países, tipos, instituciones) antes de cierre de Fase 5.
- **Menú Home — entrada Export SNIES:** pendiente. Se conecta cuando se desarrolle la página Home. Debe apuntar a Export SNIES Saliente como aterrizaje por defecto.
- ~~**(v1.4) Exclusión de eventos nacionales en Balance Presencial/Virtual:**~~ **Cerrada (v1.4).** Balance no se toca. Las páginas Balance Presencial y Virtual siguen mostrando eventos nacionales e internacionales mezclados. Razón: reabrir fases cerradas modifica números ya validados por la DRI. El volumen nacional es presumiblemente bajo. Si en el futuro distorsiona tops internacionales, se agrega filtro `País ≠ Colombia` al panel oculto con cambio mínimo.
- **(v1.4) Medida `Promedio Duración Días`:** la página de Caracterización muestra un KPI de promedio de duración. No existe como medida documentada en `_Medidas`. Se crea y documenta según §5.2 durante Fase 7.
- **(v1.4) Menú Home — entradas Caracterización, Movilidad Nacional y Financiación:** agregar cuando se desarrolle Home.
- **(v1.4) Configuración de slicers en Movilidad Nacional:** validar durante Fase 8 qué filtros se mantienen y cuáles se ajustan para evitar redundancia (en particular el slicer País, tautológico en esa página).
- **(v1.4) Configuración de slicers en Financiación:** validar durante Fase 6.

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
- ✅ `_Movilidades Base`, `_Filtro Año Anterior`, `_Título Período SNIES`

#### 11.1.3 Tabla de medidas

- Tabla vacía llamada **`_Medidas`** (underscore al inicio para ordenamiento alfabético).
- Patrón estándar SQLBI: la tabla no contiene datos, solo aloja medidas.
- Subcarpetas por dominio dentro de `_Medidas`:
  - `Movilidad/`
  - `Personas/`
  - `Convenio/`
  - `Financiación/`
  - `Auxiliares/`
  - `Notas/`

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
| Azul medio | `#2E5597` | Serie primaria en visuales, fondos de tarjetas, botón activo SNIES |
| Azul claro | `#BDD7EE` | Serie secundaria, acentos |
| Gris claro fondo | `#F2F2F2` | Fondo de paneles, separadores suaves, botón inactivo SNIES |
| Gris texto secundario | `#595959` | Etiquetas, ejes, anotaciones, texto botón inactivo SNIES |
| Texto principal | `#262626` | Cuerpo de texto, datos en tablas |
| Rojo alerta | `#C00000` | Botón "Borrar filtros", indicadores negativos |
| Verde positivo | `#548235` | Indicadores favorables (uso restringido) |
| Naranja advertencia | `#ED7D31` | Llamados de atención (uso restringido), advertencia de página Exploratoria |

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

#### 11.5.1 Tabla de slicers estándar — páginas analíticas (confirmada Fase 0)

Esta tabla es la referencia normativa para páginas del bloque ANÁLISIS. Las páginas analíticas nuevas copian el panel completo de Resumen Movilidad Presencial Internacional.

| Slicer | Tipo de visual | Orden | Pre-selección |
|---|---|---|---|
| Movilidad (Dirección) | Chiclet horizontal | Entrante, Saliente | Sin pre-selección (ambos activos) |
| Año | Dropdown | Descendente | Sin pre-selección |
| Semestre | Chiclet horizontal | 1, 2 | Sin pre-selección |
| Nivel Académico | Dropdown | Jerarquía institucional | Sin pre-selección |
| Unidad Académica | Dropdown | Alfabético ascendente | Sin pre-selección |
| Programa Académico | Dropdown | Alfabético ascendente | Sin pre-selección |
| País | Dropdown | Alfabético ascendente | Sin pre-selección |

**Regla operativa:** cada página analítica nueva se construye copiando el panel de slicers de Resumen tal cual. Si una página necesita un slicer adicional, se agrega al final sin reordenar los existentes (ej. Modalidad en Caracterización, §8.8).

**Sincronización:** activa entre páginas analíticas. Desactivada con Detalle y SNIES (§11.5.3).

#### 11.5.2 Tabla de slicers — páginas Export SNIES (confirmada Fase 2)

Panel reducido, slicers dropdown en línea horizontal arriba de la tabla (no en panel lateral). Orientados a validación operativa.

| Slicer | Campo | Origen | Pre-selección |
|---|---|---|---|
| Unidad Académica | campo unidad | `DIM_PROGRAMA_ACADEMICO` | Sin pre-selección |
| Programa Académico | campo programa | `DIM_PROGRAMA_ACADEMICO` | Sin pre-selección |
| Tipo de Actividad | `STR_DETALLE_ACTIVIDAD_CAL_AJT` | FCT | Sin pre-selección |
| ID Estudiante | `STR_PERSONA_ID_NK` | FCT | Sin pre-selección |

**Sincronización:** activa entre las dos páginas SNIES. Desactivada con páginas analíticas y Detalle.

#### 11.5.3 Grupos de sincronización

| Grupo | Páginas | Sincronizado |
|---|---|---|
| Analíticas | Balance Presencial, Virtual, Internacionales, Caracterización, Movilidad Nacional, Financiación | ✅ Entre sí |
| SNIES | Export SNIES Saliente, Export SNIES Entrante | ✅ Entre sí |
| Detalle | Detalle Movilidad | ❌ Independiente |

> **Nota v1.4:** Caracterización, Movilidad Nacional y Financiación se integran al grupo Analíticas. Modalidad es slicer (no panel oculto) en Caracterización — la sincronización de Modalidad como slicer entre páginas analíticas requiere validación en implementación (Fase 7). La configuración fina de cada página nueva se valida en su fase (Fases 6, 7 y 8).

### 11.6 Plantilla de página

**Convención:** no se mantiene un activo "plantilla" separado. Cada página nueva se construye copiando Resumen Movilidad Presencial Internacional y vaciando los visuales, preservando: logo, panel de slicers (o reemplazándolo por slicers horizontales en caso SNIES), botón borrar filtros, botón home, barra inferior de fuente, subtítulo dinámico, cuadro de texto de alcance (vacío para llenar).

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

**Dependencias:** Fase 0.

---

### **Fase 2 — Export SNIES** | branch: `phase-2-export-snies`

**Objetivo:** resolver dolor operativo recurrente (reconstrucción manual de reporte SNIES). Proveer datos esenciales crudos para flujo de homologación externo.

**✅ Fase 2 cerrada — 2026-05-27.**

Entregables completados:
- Dos páginas independientes: "Export SNIES Saliente" (18 columnas) y "Export SNIES Entrante" (22 columnas). Desvío justificado de §7.8 original (ver §7.8 revisado).
- Botones de alternancia tipo tab (Page Navigation) entre las dos páginas.
- Filtros de alcance en panel oculto: tabla origen (Movilidad + Visitantes), dirección fija por página, año y semestre fijados al período a reportar (ver §7.17).
- Título dinámico con medida `_Título Período SNIES` (ver §7.17).
- 4 slicers dropdown en línea horizontal: Unidad Académica, Programa Académico, Tipo de Actividad, ID Estudiante (ver §7.16, §11.5.2).
- Sincronización de slicers entre ambas páginas SNIES, desactivada con analíticas.
- País mostrado con nombre y código ISO en ambas tablas (ver §7.18).
- Columnas en orden SNIES. Mapeo modelo→SNIES documentado en §8.6 y §8.7.
- Cuadro de texto de alcance visible en ambas páginas.
- P2 cerrado con nueva definición (flujo externo de homologación).
- 1 medida nueva: `_Título Período SNIES` en `_Medidas/Auxiliares/`, documentada según §5.2.
- Exportación nativa funcional (clic derecho → Export data).
- Pendiente: entrada "Export SNIES" en menú Home (se conecta cuando Home se desarrolle, ver §10.3).

**Dependencias:** Fase 0, P2 resuelto.

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
2. Filtro de alcance `país ≠ Colombia` (validado — campo país = nacionalidad, P1 cerrado).
3. KPIs con desglose Regular vs Temporal.
4. Mapa, evolución, distribución por programa, top instituciones.
5. Revalidación de `[Movilidades]` con matriculados en alcance: las filas de `FCT_MATRICULADOS` tienen `NUM_NUMERO_MOVILIDAD_CAL = 0` (§4.2, §7.10) — definir en fase si el KPI correcto para regulares es `[Personas]` exclusivamente.

**Definition of Done específico:**

- Pregunta "¿quiénes nos eligen?" respondible desde la página.
- Distinción visual clara entre matriculados regulares y movilidad temporal.
- Riesgo de COUNTROWS sobre matriculados (§7.10) resuelto y documentado.

**Dependencias:** Fase 0. ~~**P1 resuelto**~~ Cerrado: campo país = nacionalidad.

~~**⚠️ Bloqueante:** no iniciar hasta confirmar P1.~~ **Desbloqueada (v1.4).**

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

**Objetivo:** página "Financiación de Movilidad — Exploratoria" según §8.10. Responde pregunta DRI #6 (§3.1) con salvaguardas de integridad. R1 cerrado: cobertura 6%/10% confirmada, multi-moneda documentada (§4.7, §7.26).

**Entregables:**

1. ~~Auditoría previa de cobertura por período.~~ **Cerrada (R1):** máx. 6% nacional, 10% internacional.
2. **Página implementada según §8.10:** 4 KPIs de cobertura + 2 tablas de valor por moneda + 2 gráficos de contexto por conteo.
3. **4 medidas nuevas de cobertura** (`[Movilidades Financiadas Nacional]`, `[% Cobertura Financiación Nacional]`, `[Movilidades Financiadas Internacional]`, `[% Cobertura Financiación Internacional]`) en `_Medidas/Financiación/`, documentadas según §5.2 con justificación 5-puntos.
4. **Cuadro de texto de alcance reforzado** con advertencia obligatoria (§8.10).
5. **Marca "Exploratoria"** en el título de página.
6. **Validación de configuración de slicers** (§10.3).

**Definition of Done específico:**

- Estructura §8.10 implementada.
- **Cero visuales con agregación inter-moneda** (verificación explícita en PR).
- Todo valor monetario visible acompañado del % de cobertura del contexto.
- Valores "NFD" excluidos de visuales de valor e incluidos en denominadores de cobertura.
- Medidas nuevas documentadas según §5.2.
- R1 cerrado y referenciado.

**Dependencias:** Fase 0. ~~Auditoría R1.~~ **R1 cerrado — Fase 6 desbloqueada.**

---

### **Fase 7 — Caracterización Estudiantes en Movilidad** | branch: `phase-7-caracterizacion` *(nueva v1.4)*

**Objetivo:** conformar la página legado de caracterización demográfica a los estándares del plan y extender su alcance para cubrir ambas modalidades (Presencial + Virtual). Responde pregunta DRI #7 (§3.1).

**Entregables:**

1. **Auditoría de página legado:** inventario de medidas, filtros y visuales existentes. Identificar desviaciones contra §5.2, §9, §11.
2. **Conformación:** reemplazar medidas no portables, agregar cuadro de texto de alcance, documentar filtros con bloque 5-puntos.
3. **Extensión a ambas modalidades:** agregar Modalidad como slicer de exploración (§7.19). Remover filtro de Modalidad del panel oculto si existe.
4. ~~**Resolución de P4:**~~ Cerrado. Opción C: se acepta duplicación, semántica "¿a qué edades se participa?" (§7.22).
5. ~~**Resolución de P5:**~~ Cerrado. Columna confirmada: `DIM_PERSONA.STR_COD_GENERO_AJT`.
6. ~~**Resolución de P6:**~~ Cerrado. Rangos confirmados: 15–17, 18–25, 26–30, 31–40, 41–50, Mayor 50, Revisar (§7.23). **Implementar columna calculada en PQ + columna de sort** (§4.3).
7. **Tratamiento de nulos en sexo:** categoría "Sin dato" visible (§7.21).
8. **Medida nueva:** `[Promedio Duración Días]` documentada según §5.2.

**Definition of Done específico:**

- Página conformada a estándares §5.2, §9, §11.
- Modalidad funciona como slicer: usuario puede ver Presencial, Virtual, o ambos.
- Visuales de atributos de persona usan `[Personas]` (§7.20); distribución etaria usa `[Movilidades]` con título "Participaciones por rango de edad" (§7.22).
- Rangos de edad implementados con columna PQ + sort (§7.23).
- Nulos de sexo visibles como "Sin dato".
- Cuadro de texto de alcance presente.
- Toda medida nueva documentada con bloque `[CASO DE USO]`.
- ~~P4~~, ~~P5~~, ~~P6~~ cerrados. Sin pendientes bloqueantes.

**Dependencias:** Fase 0 (medidas base). ~~P4~~, ~~P5~~, ~~P6~~ cerrados — **Fase 7 desbloqueada.**

---

### **Fase 8 — Movilidad Nacional** | branch: `phase-8-movilidad-nacional` *(nueva v1.4)*

**Objetivo:** página clon de Balance (§8.2) con alcance Colombia y swap geográfico→programa. Responde pregunta DRI #8 (§3.1).

**Entregables:**

1. ~~**Resolución de P7:**~~ Cerrado. Filtro confirmado: `DIM_UBICACION_GEOGRAFICA_MOVILIDAD[STR_PAIS_AJT] = "Colombia"`.
2. **Página implementada según §8.9.** Clon de Balance con Top Programas Académicos en lugar de Top Países.
3. **Filtros de alcance en panel oculto:** `STR_PAIS_AJT = "Colombia"` + `STR_TABLA_ORIGEN_CAL IN {Movilidad, Visitantes}`.
4. **Cuadro de texto de alcance visible.**
5. **Validación de configuración de slicers:** confirmar qué filtros se dejan para evitar redundancia (slicer País tautológico).
6. **Validación de interacción Top N + jerarquía** en el visual Top Programas (drilldown).

**Definition of Done específico:**

- Pregunta "¿cuál es la movilidad nacional?" respondible desde la página.
- Cero medidas nuevas (reutilización total de medidas portables).
- Estructura idéntica a Balance excepto swap Top Países → Top Programas.
- ~~P7 cerrado.~~ Cerrado.
- Cuadro de texto de alcance presente.
- Configuración de slicers validada (sin redundancia).

**Dependencias:** Fase 0 (medidas base). ~~P7~~ cerrado — **Fase 8 desbloqueada.**

---

### Orden recomendado de ejecución

```
Fase 0 ──> Fase 1 ──┬──> Fase 2 (cerrada)
                    ├──> Fase 5 (siguiente)
                    ├──> Fase 3
                    └──> Fase 7 (desbloqueada — P4/P5/P6 cerrados)

Fase 0 ──> Fase 4 (desbloqueada — P1 cerrado)

Fase 0 ──> Fase 8 (desbloqueada — P7 cerrado)

Fase 0 ──> Fase 6 (desbloqueada — R1 cerrado)
```

**Camino crítico:** Fase 0 → Fase 1 → entrega mínima viable que responde preguntas DRI presenciales.

**Siguiente fase recomendada:** Fase 5 (Movilidad Virtual) — clon estructural de Fase 1, mínimo esfuerzo, máximo retorno. DoD inalterado: cero medidas nuevas.

**Estado v1.4 (cierre):** no quedan fases bloqueadas. Fases 3, 4, 5, 6, 7 y 8 pueden ejecutarse; 5, 7 y 8 son paralelizables entre sí y de bajo riesgo (reutilización de medidas portables).

---

## 13. Convenciones de versionado git

### 13.1 Formato del proyecto

- Power BI en formato `.pbip` (proyecto, no `.pbix`).
- Genera carpeta con archivos legibles por git (JSON, TMDL).

### 13.2 Estructura de branches

```
main                    ← producción
├── phase-0-foundation          ✅ merged
├── phase-1-balance-presencial  ✅ merged
├── phase-2-export-snies        ✅ merged
├── phase-3-detalle-movilidad
├── phase-4-internacionales
├── phase-5-virtual
├── phase-6-financiacion
├── phase-7-caracterizacion
└── phase-8-movilidad-nacional
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
feat(phase-2): agrega página Export SNIES Saliente con tabla plana 18 columnas
feat(phase-2): agrega página Export SNIES Entrante con columnas nombre
feat(phase-2): agrega medida _Título Período SNIES
docs(phase-2): documenta §7.8 revisado y §8.6/§8.7 en plan maestro
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
  - v1.0 — plan inicial aprobado
  - v1.1 — ajustes Fase 0
  - v1.2 — ajustes Fase 1
  - v1.3 — Fase 2 cerrada, §7.8 revisado, §8.6/§8.7 nuevos, §11.5.2 nuevo
  - v1.4 — Ajuste pre-Fase 5: hallazgos 1–7 integrados (§3.1 P7/P8, §4.1.1/§4.3/§4.6 campos demográficos, §7.19–§7.25, §8.8/§8.9, Fases 7–8). **Cierre 2026-06-10:** P1, P4–P7 y R1 cerrados; §4.7 campos financieros; §7.20 precisado (excepción etaria §7.22); §7.26 y §8.10 nuevos (Financiación — Exploratoria); §9 #9 nuevo; conteo final 10 páginas; Fases 3–8 todas desbloqueadas.

### 14.10 Orden recomendado de ejecución de las fases

1. **Fase 0** (foundation) — ✅ cerrada.
2. **Fase 1** (Balance Presencial) — ✅ cerrada.
3. **Fase 2** (Export SNIES) — ✅ cerrada.
4. **Fase 5** (Movilidad Virtual) — clon estructural de Fase 1, bajo esfuerzo. **Siguiente.**
5. **Fase 7** (Caracterización) — conformar legado + extender a ambas modalidades. P4/P5/P6 cerrados — **desbloqueada.** Paralelizable con Fase 5.
6. **Fase 8** (Movilidad Nacional) — clon de Balance con swap programa. P7 cerrado — **desbloqueada.** Paralelizable.
7. **Fase 3** (Detalle) — ajuste menor, en cualquier momento después de Fase 0.
8. **Fase 4** (Internacionales) — P1 cerrado — **desbloqueada.**
9. **Fase 6** (Financiación) — R1 cerrado — **desbloqueada.** Página exploratoria según §8.10.

---

## 15. Glosario

| Término | Definición |
|---|---|
| **Alcance** | Porción del universo de datos que una página analiza. Se controla con filtros de página, no con DAX. |
| **Caracterización** | Análisis del perfil demográfico (sexo, edad, nivel) de los individuos en movilidad. Métricas: `[Personas]` para atributos de individuo; `[Movilidades]` para distribución etaria (§7.22). |
| **Conformación** | Proceso de auditar y ajustar una página legado para cumplir los estándares del plan (§5.2, §9, §11). |
| **Definición intrínseca** | Filtro que es parte permanente de qué mide un KPI. Va en DAX. |
| **DRI** | Dirección de Relaciones Internacionales. Cliente interno del dashboard. |
| **Entrante (Inbound)** | Movilidad de extranjeros hacia La Sabana. `STR_CLASIFICACION_MOVILIDAD_CAL_AJT = "Entrante"`. |
| **Estudiante Regular** | Matriculado en programa académico formal de La Sabana. Origen `FCT_MATRICULADOS`. |
| **Estudiante Temporal** | Movilidad o visita corta. Origen `FCT_MOVILIDAD_ESTUDIANTE` o `FCT_VISITANTE_EXTRANJERO`. |
| **Grano** | Nivel de detalle de una fila en la FCT. Ver §4.2. |
| **Modalidad** | Presencial o Virtual. Derivada de `STR_DETALLE_ACTIVIDAD_CAL_AJT`. Columna materializada: `FCT_Movilidad_Estudiantil[Modalidad]`. |
| **Movilidad** | Evento de intercambio académico. Métrica: `COUNTROWS(FCT_Movilidad_Estudiantil)`. |
| **Movilidad Nacional** | Movilidad estudiantil con destino/origen dentro de Colombia. Filtro: `DIM_UBICACION_GEOGRAFICA_MOVILIDAD[STR_PAIS_AJT] = "Colombia"`. |
| **NFD** | Valor en `STR_MONEDA_FINAN_*` que indica "sin dato de financiación". Se excluye de visuales de valor; define el denominador inverso de las medidas de cobertura (§7.26). |
| **Página Exploratoria** | Página construida sobre datos con cobertura parcial confirmada. Lleva advertencia obligatoria visible y no respalda decisiones operativas ni presupuestales (§7.26, §8.10). |
| **Persona** | Individuo único. Métrica: `DISTINCTCOUNT(STR_PERSONA_ID_NK)`. |
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