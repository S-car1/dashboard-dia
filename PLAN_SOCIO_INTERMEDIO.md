# Plan: agregar Socioemocional DIA Intermedio 2026 al dashboard-dia

Fecha: 2026-09-22. Escrito en la conversación del proyecto `Informes DIA Intermedio
(Socioemocional)`, para ejecutar aquí (`dashboard-dia`). Nada de esto se ha aplicado todavía.

## Contexto y archivos fuente (ya generados, en otro proyecto)

Carpeta: `C:\Users\Sebastian Carcamo M\Documents\Proyectos\Informes DIA Intermedio (Socioemocional)\`

| Archivo | Qué es |
|---|---|
| `DIA_Socioemocional_Intermedio_2026.xlsx` | Hoja `datos`: **1.878 filas, 70 RBD**, 9 columnas en formato `datos.json`. Hoja `N_cuestionarios`: referencia (N por grado, no va al JSON) |
| `generar_base_socio.py` | Genera el Excel de arriba desde `DIA_MONITOREO_2026_DEL_PINO.xlsx` (re-ejecutable si cambia el mapeo) |
| `PLAN_integracion_dashboard.md` | Plan detallado original (datos, pipeline, riesgos, pruebas paso a paso) |
| `CONTEXTO.md` | Historial de decisiones de esta base de datos |

Verificado: el Excel coincide al decimal con los 511 gráficos de los 73 PDFs oficiales (OCR, 0
diferencias). Es la fuente de verdad, no hace falta scrapear PDFs.

## Estado de los repos del dashboard (revisar antes de tocar nada)

| Repo/carpeta | Estado |
|---|---|
| `GitHub\dashboard-dia\` — rama `main` | Lo publicado en vivo (`https://s-car1.github.io/dashboard-dia/`), commit `c63a616` |
| `GitHub\dashboard-dia\` — rama `ui-stitch` | Rediseño visual (Stitch), `index.html` modificado **sin commitear**. **EN HOLD**: pendiente de que jefatura apruebe vía `dashboard-dia-revision`. No tocar esta rama hasta terminar lo de abajo |
| `GitHub\dashboard-dia-revision\` | Repo público aparte, copia temporal de `ui-stitch` para revisión de jefatura. No se toca en esta tarea |
| `dashboard-dia-piloto\` | Sandbox sin git para probar sin arriesgar el real. **Desactualizado** (632k filas vs 711k reales) — resincronizar antes de usarlo |

**Decisión ya tomada:** estos cambios de datos van sobre `main` (no sobre `ui-stitch`), para no
mezclar cambio de datos con rediseño sin terminar. Cuando esto esté en producción, se retoma
`ui-stitch` adaptándolo a los mismos cambios.

**Decisión pendiente (resolver en esta conversación):** trabajar primero en
`dashboard-dia-piloto` (resincronizar y probar ahí) vs. crear una rama nueva en `dashboard-dia`
desde `main` (ej. `feature/socio-intermedio`). Recomendación: piloto primero (menor riesgo, sin
git de por medio), luego aplicar a una rama de `dashboard-dia` para el PR/merge final.

## 1. Mapeo de datos (ya generado, para referencia)

Formato de fila (igual a `datos.json`): `Agno=2026, Etapa="Intermedio", RBD_Establecimiento,
Nivel_fix, Curso="Socioemocional", Asignatura, Eje, Puntaje (0-1), Comuna`.

| Columna del consolidado | Asignatura | Eje | Filas |
|---|---|---|---|
| BIENESTAR_EXPERIENCIAS | Bienestar | Experiencias | 339 |
| BIENESTAR_CONTEXTO | Bienestar | Contexto escolar | 339 |
| ANSIEDAD_EXPERIENCIA | Ansiedad ante evaluaciones | Experiencia | 339 |
| ANSIEDAD_GESTION_APOYOS | Ansiedad ante evaluaciones | Gestión de apoyos | 339 |
| CONVIVENCIA_DESARROLLO (solo 7° a IV medio) | Convivencia digital | Desarrollo estudiantes | 170 |
| CONVIVENCIA_GESTION | Convivencia digital | Gestión establecimiento | 339 |
| VALORACION_EDUCACION (solo IV medio) | Valoración de la educación | Valoración | 13 |

Nombres de Asignatura **a propósito distintos** a los del Diagnóstico (que usa
`Desarrollo`/`Gestión` × `Personal/Comunitario/Ciudadano`) para que nunca se mezclen en el mismo
radar. Cobertura: 70 de 73 RBD (3 sin ningún grado socio: 10565, 10588, 25960).

Confirmado contra `datos.json` real (711.330 filas, formato `{cols, maps, rows}`): hoy
`Curso="Socioemocional"` solo existe para `2026/Diagnóstico` (3.750 filas, Asignatura
`Desarrollo`/`Gestión`); `Curso="Convivencia"` igual (2.303 filas). `Etapa` ya admite `Cierre`
como valor pero no se usa aún en esta categoría. `Año 2025` no tiene nada de esta categoría.

## 2. Fusión en `datos.json`

Script nuevo (ej. `DIA_Intermedio/agregar_socio_intermedio.py`), usando
`dataset.cargar_datos_json`/`guardar_datos_json` (formato `cols/maps/rows`, Puntaje a 4
decimales). Idempotente:
1. Cargar `datos.json` (piloto primero, luego real).
2. Borrar filas previas con `Agno=2026 & Etapa=Intermedio & Curso=Socioemocional` (si existen de
   una corrida anterior).
3. Agregar las 1.878 filas de la hoja `datos` del Excel.
4. Guardar.

**No usar el merge normal de `consolidar.py`** (dedupe por `Agno,Etapa,RBD,Curso,Asignatura` sin
Nivel/Eje — sirve para archivos de colegio, no para esto).

Validaciones: total pasa de 711.330 a 713.208; todo lo que no es Socioemocional-Intermedio queda
idéntico (comparar antes/después); solo se agregan 4 Asignaturas y 7 Ejes nuevos a `maps`
(RBD/Nivel/Comuna ya existen); ida y vuelta contra el Excel original = 0 diferencias.

## 3. Pipeline `DIA_Intermedio` (para no romper la rutina diaria)

- `auditar.py` → `cargar_real()` (l.116-127): hoy contaría las 1.878 filas nuevas como "SOBRA".
  Parche: ignorar `Curso in ('Socioemocional','Convivencia')` ahí.
- Nuevo `verificar_socio_intermedio.py`: compara `datos.json` vs el consolidado (misma ida y
  vuelta del punto 2).
- `consolidar.py`: confirmar con una corrida de prueba que preserva las filas (reescribe todo
  `datos.json`).
- Documentar en `DIA_Intermedio/CONTEXTO.md`.

## 4. Front-end `index.html` — diseño ya definido

Layout: **secciones apiladas** dentro de la pestaña existente "Convivencia y Socioemocional"
(mismo patrón visual que ya usa hoy, sin tabs/toggle nuevos — cada ronda es una sección con su
propio título, radar y aviso de "no comparable").

**HTML** (reemplaza la sección actual "Sección: Socioemocional"):
```html
<div class="section-title">Sección: Socioemocional — Ronda Diagnóstico</div>
<div class="charts-grid">
  <div class="chart-block full-width">
    <div class="chart-card">
      <button class="btn-download" onclick="descargarGrafico(this)" title="Descargar Gráfico">...</button>
      <div class="chart-header">
        <div><div class="chart-title">Socioemocional<span>Perfil Integrado (Radar 2026 · Diagnóstico)</span></div></div>
      </div>
      <div class="chart-wrap" style="min-height: 450px;"><canvas id="canvas-socio-super-radar"></canvas></div>
    </div>
  </div>
</div>

<div class="section-title">Sección: Socioemocional — Ronda Monitoreo Intermedio</div>
<p style="color: var(--text-muted); font-size: 0.85rem; margin: -0.5rem 0 1rem;">
  Las rondas Diagnóstico e Intermedio usan instrumentos distintos: no son comparables entre sí.
</p>
<div class="charts-grid">
  <div class="chart-block full-width">
    <div class="chart-card">
      <button class="btn-download" onclick="descargarGrafico(this)" title="Descargar Gráfico">...</button>
      <div class="chart-header">
        <div><div class="chart-title">Socioemocional<span>Perfil Integrado (Radar 2026 · Intermedio)</span></div></div>
      </div>
      <div class="chart-wrap" style="min-height: 450px;"><canvas id="canvas-socio-int-radar"></canvas></div>
    </div>
  </div>
  <div class="chart-block">
    <div class="chart-card">
      <div class="chart-header">
        <div><div class="chart-title">Valoración de la educación<span>Solo IV Medio · 13 RBD</span></div></div>
      </div>
      <div class="kpi" style="--kpi-c: var(--purple); margin-top: 1rem;">
        <div class="kpi-label">Promedio</div>
        <div class="kpi-val" id="kpi-valoracion-edu">—</div>
      </div>
    </div>
  </div>
</div>
```

**JS — `SUBJECTS` (l.466) y `asignaturasEspeciales` (l.739):** agregar a ambas listas:
`'Bienestar', 'Ansiedad ante evaluaciones', 'Convivencia digital', 'Valoración de la
educación'`. (Falta en `SUBJECTS` → `TypeError` al primer render. Falta en
`asignaturasEspeciales` → contamina el KPI "Promedio General".)

**JS — `updateImmediate()`, después de la llamada del radar de Diagnóstico (l.808):**
```js
updateSuperRadarCombinado(
    charts['socio-int-radar'],
    { Bienestar: res.ejes.Bienestar, 'Ansiedad ante evaluaciones': res.ejes['Ansiedad ante evaluaciones'], 'Convivencia digital': res.ejes['Convivencia digital'] },
    sData ? { Bienestar: slep.ejes.Bienestar, 'Ansiedad ante evaluaciones': slep.ejes['Ansiedad ante evaluaciones'], 'Convivencia digital': slep.ejes['Convivencia digital'] } : null,
    ['Bienestar', 'Ansiedad ante evaluaciones', 'Convivencia digital']
);

// Valoración de la educación: 1 solo eje, solo IV medio -> tarjeta KPI, no radar
const ve = res.ejes['Valoración de la educación']?.['Valoración']?.['2026'];
const veStage = ve ? ve[STAGES.indexOf('Intermedio')] : [0, 0];
document.getElementById('kpi-valoracion-edu').innerText = f(veStage[0], veStage[1]);
```
Nota: `updateSuperRadarCombinado` (función genérica existente, l.860) **no se modifica** — ya
elige sola "la última etapa 2026 con datos", y como Bienestar/Ansiedad/Convivencia digital solo
existirán en Intermedio, resuelve sola a esa etapa. No hace falta parámetro de etapa fija.

**JS — filtro Etapa dentro de esta pestaña:** hoy `filters.etapa` se aplica global (l.712) sobre
TODAS las filas — si el usuario filtra "Intermedio" o "Diagnóstico" en esta pestaña, vacía uno de
los dos radares. Igual que ya existe `TABS_SIN_CURSO` (l.604) para el filtro Curso: agregar
`id="filter-group-etapa"` al div del filtro (l.248, hoy sin id) y un `TABS_SIN_ETAPA =
['tab-convivencia-socio']` con el mismo bloque disable/enable (clear + disable +
`data-label="ETAPA (No aplica)"`) dentro de `showTab()`.

**JS — init de charts (l.1362):** agregar `'socio-int-radar'` a la lista que crea los radares
(mismas opciones que `socio-super-radar`).

**Estados vacíos:** RBD sin datos en esta ronda (10565, 10588, 25960, o filtro sin grados con
dato) → mensaje "Sin resultados en esta ronda" en vez de canvas/tarjeta en blanco.

## 5. Pruebas (antes de publicar)

Regresión: KPI Promedio General/Lectura/Matemática y radar Diagnóstico **idénticos**
antes/después. Intermedio sin filtros = promedio simple por Eje (comparar contra
`groupby(Asignatura,Eje).Puntaje.mean()` en Python). Un RBD conocido (ej. 9583) contra la hoja
`datos` del Excel. Casos límite: 10588 (sin datos), 9713 (un solo grado), 10686 (mezcla
básica/media). Filtro Etapa deshabilitado en esta pestaña y reactivado al salir. Consola sin
errores, descarga de gráficos, vista móvil.

## 6. Publicación

Confirmar con Sebastián antes de commit/push. `git pull` de `main` primero (por sesiones
paralelas). Backup de `datos.json` real antes de tocarlo. Commits separados: uno de datos, uno de
`index.html`. Verificar en vivo tras el despliegue de Pages.

## 7. Pendiente aparte (no bloquea esto)

Cuando esto esté en producción sobre `main`: retomar `ui-stitch` y adaptarlo a estos mismos
cambios de datos/HTML, y re-duplicar a `dashboard-dia-revision` para que jefatura siga
revisando. `DIA_Socioemocional.py` (script del Diagnóstico) tiene una API key de Gemini
expuesta — revocar, no reutilizar ese script.
