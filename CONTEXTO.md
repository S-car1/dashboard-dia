# Contexto — dashboard-dia

## Resumen
Dashboard de resultados DIA (Diagnóstico Integral de Aprendizajes) para SLEP Del
Pino: archivo único `index.html` (Chart.js + chartjs-plugin-datalabels + tom-select),
consume `datos.json` local. Login contra un endpoint de Apps Script compartido con
Proyecto_Asistencia.

## Último estado (actualizado: 2026-08-29)
Pase de auditoría/prototipo local sobre 7 puntos de calidad de front-end (encoding,
colisiones de variables globales, reset de leyendas, debounce, tabla ordenable,
tooltips/datalabels, recálculo de promedios). Este repo es la implementación de
referencia del patrón `debounce()` que otros repos (Proyecto_Asistencia, SLEP,
dashboard-dia-ep) están copiando.

Resultado del punto por punto:
- **Encoding/BOM/mojibake**: limpio, sin cambios (charset UTF-8 presente, sin BOM,
  sin patrones de mojibake).
- **Colisiones de variables globales**: ninguna encontrada (`top/name/location/...`),
  sin cambios.
- **Reset de leyendas con doble click**: NO existía. Se agregó `makeLegendOnClick()`
  (detecta doble click en <400ms sobre el mismo dataset de la leyenda y llama
  `setDatasetVisibility(i, true)` a todos + `chart.update()`), conectado a los 9
  gráficos vía `onClick` en `lineOpt`, `barPorAnioOpt` y `radarOpt`. El click simple
  se probó y sigue funcionando igual que antes (toggle normal, sin tocar otras series).
- **Debounce — auditoría**: el patrón `debounce(updateImmediate, 150)` YA estaba
  aplicado de forma consistente en TODOS los puntos de entrada que pueden dispararse
  rápido (los 6 `onChange` de tom-select, el botón limpiar filtros, los tags de
  filtros activos, las tarjetas de eje, el toggle Lectura/Matemática). No se encontró
  ningún handler que llamara a `updateImmediate()` directo saltándose el debounce.
  Verificado en vivo: 10 llamadas rápidas a `update()` → 0 renders a los 50ms → un
  solo ciclo de render (9 `chart.update()`, uno por gráfico) después de que se
  cumplen los 150ms.
- **Tabla ordenable**: la única tabla real (`var-table`, la de "Variación 2026 vs
  2025", reutilizada en 3 contenedores: `var-leng-evo`, `var-mate-evo`,
  `var-1basico-evo`) no tenía headers clickeables. Se agregó orden por columna
  (numérico para las columnas de variación, `localeCompare('es')` para la de
  texto), indicador visual ▲/▼, y persistencia del estado de orden
  (`tableSortState` + `lastVarTableData`, a nivel de módulo) para que sobreviva al
  re-render que dispara cada cambio de filtro. Nota: el `<thead>` se sigue
  reconstruyendo completo en cada render (mismo patrón que ya tenía la función vía
  `innerHTML`), así que los listeners se reasignan a nodos `<th>` nuevos cada vez —
  no hay acumulación de listeners porque los nodos viejos no siguen vivos.
- **Tooltips y datalabels**: ya estaban bien — tooltip no deshabilitado en ningún
  gráfico, `chartjs-plugin-datalabels` registrado globalmente y con `formatter`
  coherente con el tooltip (mismo % en ambos). Sin cambios.
- **Promedio/recompute**: ya estaba correcto — recalcula en cada `update()`,
  maneja el caso sin datos (`count === 0` → `'—'`, sin NaN). Sin cambios.

## Decisiones clave
- [2026-08-29] Reset de leyenda implementado como doble click sobre el ítem de
  leyenda (vía `legend.onClick` con detección de tiempo), en vez de un botón
  "Mostrar todo" aparte, para no sumar más UI al header de cada gráfico.
- [2026-08-29] La tabla usa `innerHTML` completo por render (no solo el `<tbody>`),
  igual que el código original — se mantuvo ese patrón y se resolvió la
  persistencia de orden con estado a nivel de módulo en vez de cambiar la
  estrategia de renderizado de la tabla.

## Pase de normalización + verificación (2026-09-03)
Se corrigieron dos inconsistencias detectadas al comparar con los otros
repos: (1) faltaba `.filter-value-tag:hover` (hover rojo+tachado en los tags
de filtros activos, que los repos hermanos sí tenían), agregado; (2) el
criterio de línea sólida/punteada por año estaba invertido en `createDS()`
(marcaba 2026 como punteada) y además contradecía a `ESTILO_POR_ANIO`, que
en el mismo archivo ya tenía el criterio correcto — se unificó a: 2026 sólida
(dato vigente), 2025 punteada (histórico). Commit `1c70ef1` (ya pusheado)
incluye ambos fixes + lo del pase anterior.

Un subagente verificó ese commit en vivo (bypass de login vía
`IS_ADMIN=true; startDashboard()`): confirmó `makeLegendOnClick()` no
comparte estado de doble-click entre gráficos distintos, confirmó
`chart.data.datasets.map(d => d.borderDash)` correcto (2025→`[5,5]`,
2026→`[]`) tras el fix, confirmó que la rama `isEjes=true` de
`renderVariationsTable` (la duda pendiente de abajo) nunca se invoca en la
práctica — no es un riesgo nuevo, es código ya muerto de antes. **No se
encontraron bugs.**

## Rediseño visual (piloto Stitch + Kit Digital) — 2026-09-21
Rama local `ui-stitch`, **SIN commit ni push** (decisión del usuario: primero validar
en local, luego ver cuándo subir). `main` y GitHub Pages siguen con la versión anterior.

Qué se hizo (CSS/markup + los cambios acotados de JS listados abajo; los cálculos existentes, los datos y el login NO se modificaron):
- Paleta sobria con azul marino `#25306B` como acento (variables `--brand`,
  `--brand-soft`, `--on-brand` en `:root` y en `body.dark-theme`); tema oscuro con la
  misma paleta. `--leng`/`--mate` se mantienen (colores de las series).
- Tipografía gobCL del Kit Digital, embebida desde `fonts/` (4 .otf, ~160 KB);
  se quitó Google Fonts (Archivo/JetBrains Mono).
- Logo blanco (`logo-blanco.png`, del kit) para el tema oscuro: antes el logo negro casi
  no se veía. `logo3.png` ya era idéntico al logo oficial a color.
- Filtros enmarcados (filtro con selección se marca en azul marino), "Filtros activos"
  en fila propia bajo los filtros, pestañas con íconos, KPI con barra lateral + ícono,
  control segmentado Lectura/Matemática, tabla de variación con banda de título y
  píldoras, línea fina de 6 colores del logo arriba (quitar la regla
  `.sticky-top-section::before` si no gusta).
- Responsive: se agregó breakpoint ≤1180px (toggles de tema/SLEP bajan a su propia fila;
  antes se solapaban con las pestañas entre 641 y ~1280px) y ≤480px (KPI en 1 columna).
- Cambios en JS (todos aditivos o de presentación; diff contra `main` revisado):
  · 4 strings de fuente `'Archivo'` → `'gobCL'` en las opciones de los gráficos.
  · **Delta en los KPI** (Promedio General, Lectura, Matemática): `getKpiDelta()` y
    `renderKpiDelta()` (nuevas, junto a `getVariationDiff`). Compara 2026 vs 2025 en la ÚLTIMA
    etapa con datos en ambos años (no retrocede a etapas anteriores) y se rotula con esa etapa
    (`↓ 0.8 pp · vs 2025 · Intermedio`). Muestra `—` si algún año tiene < `MIN_REGISTROS_DELTA`
    (=30) puntajes o no hay etapa comparable (p. ej. filtrando solo 2026). Promedio General =
    Lenguaje + Matemática, igual que `res.kpi`. Verificado contra un cálculo independiente sobre
    `DATA_RAW` (sin filtros, por comuna, solo 2026) y con 9 pruebas de borde sintéticas.
  · Rótulo `%` → `pp` (puntos porcentuales) en las insignias y el título de la tabla de
    variación (solo texto; el cálculo `getVariationDiff` no cambió).
  · Series de evolución Lectura/Matemática usan `cssVar('--leng'/'--mate')` según el tema
    (antes fijas `#38bdf8`/`#fb923c`, con poco contraste en modo claro); `toggleTheme()` llama
    `updateImmediate()` para repintarlas. Las demás series (ejes, radares) siguen con `COLORS`.

Verificación (Playwright + Edge, servidor local `python -m http.server`): 0 errores JS;
prueba con clics reales de filtros, bandeja, toggles, orden de tabla, Promedio SLEP, tema,
"Ver detalle" y descarga de gráfico; 15 anchos de 1920 a 320 px sin desbordes ni solapes
(la versión original tenía desbordes/solapes entre 641 y 1280 px y KPI cortados en móvil).

Sobre Stitch: genera mockups por dispositivo (DESKTOP/MOBILE/TABLET) a una resolución
fija, HTML con Tailwind y gráficos SVG dibujados a mano; no sirve para pegar en el
dashboard ni resuelve responsive. Se usó como referencia de diseño. Se descartó lo
inventado (menú superior, "Descargar informe", metas, cobertura, notificaciones, etc.).
`get_screen` entrega HTML + captura; `list_screens` devolvió vacío (hay que sacar el ID
de la URL: parámetro `node-id`). Proyecto Stitch: `projects/1664661317738926419`
(privado); pantalla `b8e833bb21c64d25bf9ce38bb9aeb3fb`.

### Versión de revisión publicada (2026-09-21)
Para que la vea jefatura sin tocar la versión oficial: repo aparte **`S-car1/dashboard-dia-revision`**
(público, carpeta hermana `../dashboard-dia-revision`), servido en
`https://s-car1.github.io/dashboard-dia-revision/`. Es copia de la rama `ui-stitch` con título
`[REVISIÓN]` y `noindex`; no incluye `.git`, `CONTEXTO.md` ni el kit. Este repo (`dashboard-dia`) sigue
sin commits de la rama `ui-stitch` y su Pages (`main`) sigue con la versión anterior.
Verificado en la URL pública: carga, 711.330 filas, fuentes, 0 errores JS, deltas de KPI y el **login
responde desde el origen nuevo** (Apps Script devolvió 'Credenciales incorrectas' a una credencial falsa).
Primer despliegue de Pages tardó ~8 min (17 MB de `datos.json`).

Flujo tras el visto bueno: (1) commit de `ui-stitch` en `dashboard-dia` y merge/push a `main`;
(2) borrar el repo `dashboard-dia-revision`. Si jefatura pide cambios: editar en `ui-stitch`, copiar
`index.html` a `../dashboard-dia-revision`, reponer título `[REVISIÓN]` + `noindex` y hacer push.
## Plan escrito: Socioemocional DIA Intermedio 2026 (2026-09-22)
Ver **`PLAN_SOCIO_INTERMEDIO.md`** (en esta misma carpeta) — plan completo y autocontenido para
agregar los datos del Intermedio Socioemocional a `datos.json` y a esta pestaña del dashboard
(mapeo, script de fusión, parche a `auditar.py`, HTML/JS exactos con número de línea, pruebas,
publicación). Nada aplicado todavía. Decisión pendiente ahí mismo: partir en
`dashboard-dia-piloto` o en una rama nueva de este repo.

## Socioemocional Intermedio fusionado en ui-stitch (2026-09-22)
Sebastián aplicó los datos del DIA Monitoreo Intermedio 2026 — Socioemocional en `main` (rama
`feature/socio-intermedio`, commits `c393929` + `bd0cf66`, sin mergear a `main` todavía): `datos.json`
+1.878 filas (4 asignaturas nuevas: Bienestar, Ansiedad ante evaluaciones, Convivencia digital,
Valoración de la educación) y la pestaña Convivencia/Socioemocional pasó de radar a gráficos de
barras horizontales, reordenada en 2 secciones ("Socioemocional" con Diagnóstico + Intermedio side
by side, "Convivencia" solo Diagnóstico abajo), con overlay "Sin resultados en esta ronda" y filtro
Etapa deshabilitado en esa pestaña (`TABS_SIN_ETAPA`).

Se fusionó `feature/socio-intermedio` en `ui-stitch` (`git merge --no-ff`, commit `1bb5eed`):
automático, sin conflictos (las áreas que tocaba cada rama no se solapaban). Único ajuste manual:
la función nueva `barrasOpt()` traía 2 referencias de fuente `'Archivo'` en vez de `'gobCL'`
(no existía en `ui-stitch` al momento de la conversión general de fuentes) — corregido.

Verificado en local (servidor `python -m http.server`, bypass de login `IS_ADMIN=true;
startDashboard()`): 0 errores de consola, las 12 asignaturas cargan (`DATA_RAW.maps[5].length`),
el gráfico de barras "Socioemocional · Intermedio" renderiza con el divisor antes de "Valoración
de la educación" tal como en el plan. Comparación de responsive contra `main` (versión oficial) al
mismo ancho: a 1084px la oficial solapa "Tema Claro"/"Promedio SLEP" con el texto de las pestañas
(bug real, rango 641-1280px); este rediseño no, los baja a su propia fila.

Subido: `ui-stitch` pusheado a GitHub (commit `1bb5eed`) y copiado a `dashboard-dia-revision`
(commit `7965739`, con título `[REVISIÓN]` + `noindex` repuestos) para que jefatura vea la versión
actualizada. Pendiente su visto bueno para decidir si se adopta como oficial.

Nota de troubleshooting: al probar en local con Claude in Chrome, dos servidores `http.server`
compitiendo en el mismo puerto (uno viejo de otra sesión) sirvieron contenido desactualizado sin
avisar error — y por separado, el navegador cacheó `datos.json` (17 MB, sin `Cache-Control`) entre
navegaciones al mismo puerto, mostrando datos viejos pese a que el archivo en disco ya estaba
actualizado. Si algo similar vuelve a pasar: verificar `netstat -ano | grep <puerto>` por servidores
duplicados, y forzar hard-reload (Ctrl+Shift+R) o `fetch(..., {cache:'no-store'})` para descartar caché
del navegador antes de sospechar de un bug real en el código.

## Pendientes
- [ ] Revisar el rediseño en local (`python -m http.server` en esta carpeta, rama
      `ui-stitch`) con login real contra Apps Script (las pruebas saltaron el login).
- [ ] Decidir si se sube a producción (commit + push/merge a `main`), y replicar en
      `dashboard-dia-ep`, `Proyecto_Asistencia` y `SLEP`.
- [ ] Probar en un teléfono/tablet físico y con Safari/Firefox (pruebas hechas solo en Edge).
- [ ] La rama `isEjes = true` de `renderVariationsTable` (tabla con múltiples
      filas/ejes) no tiene ningún caller real hoy en `updateImmediate()` — el
      ordenamiento se probó con datos sintéticos porque las 3 llamadas actuales
      solo generan 1 fila. Si en algún momento se usa esa rama con datos reales,
      confirmar que el orden se vea bien con más filas.
- [ ] `MIN_REGISTROS_DELTA = 30` cuenta filas de puntaje (una por eje/curso), no estudiantes;
      confirmar que el umbral es razonable. Ningún colegio de la base actual cae bajo 30.
- [ ] Al filtrar solo 2026 (o un año) los deltas muestran `—`: no hay base 2025 en ese filtro.
      Igual comportamiento que la tabla de variación.
- [ ] Sin filtro de año el KPI mezcla 2025 y 2026 y no lo dice; evaluar rotularlo.
- [ ] Esperar comentarios / visto bueno de jefatura sobre la versión de revisión actualizada
      (ya incluye Socioemocional Intermedio) y aplicar cambios si piden algo.
- [ ] Al aprobarse: mergear `ui-stitch` → `main` en este repo (esto también trae los datos de
      `feature/socio-intermedio`, que aún no está en `main`) y borrar `dashboard-dia-revision`.
- [ ] Replicar el rediseño en `dashboard-dia-ep`, `Proyecto_Asistencia` y `SLEP` una vez aprobado.
- [ ] La jefatura necesita usuario/clave válidos para el login (mismo Apps Script).
