# Cuadro de Mando TESEO

Skill para construir dashboards en cualquier microfrontend, siguiendo los patrones de `Global/src/pages/CuadroMandoIncidentes` y `Global/src/pages/CuadroMandoPuestoServicio`. Rutas relativas a `MicroFrontend/Aplicaciones/`.

## Archivos de Referencia

Leer antes de implementar:

| Capa | Archivo |
|---|---|
| Interfaces (ref 1) | `Global/src/interface/Global/ICuadroMandoIncidente.ts` |
| Interfaces (ref 2) | `Global/src/interface/Global/ICuadroMandoPuestoServicio.ts` |
| Servicio (ref 1) | `Global/src/services/global/CuadroMandoIncidenteService.ts` |
| Servicio (ref 2) | `Global/src/services/global/CuadroMandoPuestoServicioService.ts` |
| Hook (ref 1) | `Global/src/hooks/cuadroMando/useCuadroMando.ts` |
| Hook (ref 2) | `Global/src/hooks/cuadroMando/useCuadroMandoPuestoServicio.ts` |
| Pagina (ref 1) | `Global/src/pages/CuadroMandoIncidentes/CuadroMandoIncidentes.tsx` |
| Pagina (ref 2) | `Global/src/pages/CuadroMandoPuestoServicio/CuadroMandoPuestoServicio.tsx` |
| Charts | `components/ChartMensual.tsx`, `ChartHistorico.tsx`, `DonutChartIncidentes.tsx`, `ChartCumplimientoTurnos.tsx` |
| Cards | `components/KpiCard.tsx`, `KpiCardPS.tsx`, `VariationCard.tsx` |
| Colores | `Global/src/utils/cuadroMandoColors.ts` |
| Utilidades | `Global/src/utils/utils.ts` |
| Fecha/Hora | `Global/src/utils/dateTimeHelper.tsx` |
| Estado | `Global/src/common/app.constants.ts` → `IResponseState<T>` |
| HTTP | `Global/src/services/http/apiClient.ts` |
| Estilos compartidos | `Global/src/styles/cuadroMando.css` |
| Estilos por dashboard | `Global/src/pages/CuadroMando{Dominio}/styles.css` |

## Proceso

### Paso 1: Interfaces

Crear `src/interface/{Modulo}/ICuadroMando{Dominio}.ts` con DTOs tipados. Cada DTO debe modelar exactamente la respuesta del backend. Incluir `ICuadroMando{Dominio}Filtros` con `anio` obligatorio e IDs opcionales. Para KPIs con variacion, usar la estructura `IKpiValor` (valorActual, valorComparativo, variacionPorcentaje, periodoComparacion).

### Paso 2: Servicio

Crear `src/services/{modulo}/CuadroMando{Dominio}Service.ts`. Exportar como objeto con metodos async (no clase). Constante `BASE` para prefijo de endpoints. Usar la instancia axios correspondiente (`axiosGlobal`, `axiosAaa`, etc.) importada desde `services/http`. Pasar filtros con `{ params }` para GET. Tipar request y response con generics de axios (`axiosGlobal.get<TipoRespuesta>(...)`). Excel siempre con `responseType: 'blob'`.

### Paso 3: Hook

Crear `src/hooks/cuadroMando/useCuadroMando{Dominio}.ts`. Un `useState<IResponseState<T>>` por cada slice de datos, con factory `INITIAL_STATE<T>()` y helper `toApiError()`. Flag `loading` compuesto (OR de todos los individuales).

**`SetterMap` estabilizado con `useRef`**: Crear `useRef<SetterMap>` con el mapeo de keys a setters. Esto evita stale closures y satisface `exhaustive-deps` en el linter. NO pasar setters directamente como dependencias de `useCallback`.

Patron central — `ejecutarCarga`: recibe array de `{ key: StateKey; promise: () => Promise<unknown> }`, accede a `setterMapRef.current`, marca loading en cada key, ejecuta con `Promise.allSettled` para aislamiento de errores, y actualiza cada setter segun fulfilled/rejected. Usar `useCallback` con dependencias vacias `[]` gracias al ref estable.

Exponer: `cargarDashboard` (carga inicial y recarga por anio — todos los endpoints) y funciones de recarga parcial por dimension (`recargarPorMes`, `recargarPor{Dimension}`) que solo recargan los slices afectados. **NO crear funciones duplicadas** — si `recargarPorAnio` hace lo mismo que `cargarDashboard`, no crearlo. Todas con `useCallback`.

### Paso 4: Componentes de Graficos

Crear `src/pages/CuadroMando{Dominio}/components/` (35-100 lineas cada uno):

- **ChartMensual** — `GroupedVerticalBarChart`: 12 meses x 2 series, `barWidth={11}`, `maxBarWidth={16}`, `roundCorners`, `height={220}`. `ResizeObserver` para ancho con cleanup (`observer.disconnect()`). `calloutProps` con `styles: { calloutContentRoot: 'cuadro-mando-callout-half' }` (tipo `Partial<ChartPopoverProps>`, NO usar `className` que no existe en este tipo).
- **ChartBarrasApiladas** — `VerticalStackedBarChart`: multiples series apiladas por mes, `barWidth={20}`, `roundCorners`. Soporta `lineData` para indicadores superpuestos (ej. % cumplimiento). Misma configuracion de `calloutProps` y `ResizeObserver`.
- **ChartHistorico** — `VerticalBarChart`: ultimo anio en RED, anteriores en gris (#EDEBE9). Labels con variacion (triangulos + %). `barWidth={46}`, `hideLegend`, `roundCorners`.
- **DonutChart** — `DonutChart`: `innerRadius={46}`, `valueInsideDonut={total}`, colores ciclicos de `CHART_SERIES_COLORS`, `width/height={200}`.
- **KpiCard** — Contador animado con `requestAnimationFrame` (800ms). **Cleanup obligatorio**: `return () => cancelAnimationFrame(rafId)` en el `useEffect` para evitar memory leaks al desmontar. Props: label, value (o `data: IKpiValor`), color, delay, icon. Borde superior coloreado, animacion `fadeUp` escalonada. Prop `invertVariation` para diferenciar KPIs donde subir es bueno (ej. % Cumplimiento → verde) vs donde subir es malo (ej. Ausentismo → rojo).
- **VariationCard** — Colores semanticos (verde=mejora, rojo=empeora). Iconos `ArrowTrendingRegular`/`ArrowTrendingDownRegular`. Diferencia absoluta + porcentaje.

### Paso 5: Pagina Principal

Crear `src/pages/CuadroMando{Dominio}/CuadroMando{Dominio}.tsx`. Layout: Header (`.cuadro-mando-header`: titulo, selector anio, Dropdown filtro, boton exportar) + Body (`.cuadro-mando-body`: filas de contenido con grid) + Footer (`.cuadro-mando-footer`: timestamp).

- **Filtros**: Anio con botones → `cargarDashboard` (recarga todos los endpoints). Mes/Tipo con Fluent `Dropdown` → funcion de recarga parcial correspondiente. Empresa con click en tabla (toggle).
- **Ref para closures**: Mantener `useRef(cargarDashboard)` actualizado y llamarlo desde `useEffect` inicial para evitar closures obsoletos.
- **Inicializacion de filtros**: Usar `getCurrentYear()` y `getCurrentMonth()` de `dateTimeHelper.tsx` en lugar de `new Date()` directo.
- **Calculos derivados con `useMemo`**: Envolver todos los calculos derivados (totales, maximos, porcentajes, agregaciones) en `useMemo` con las dependencias correctas. NO calcular dentro del render sin memoizar.
- **Keys en listas**: Usar identificadores del dominio (`puestoId`, `nombrePuesto`, `tipoDePuesto`, `mes`) como `key`. **NUNCA usar indices de array** (`key={i}`).
- **Empty states**: Mostrar `<td colSpan={N} className="cuadro-mando-empty-state">Sin datos para el periodo seleccionado</td>` cuando un array de datos esta vacio.
- **Excel**: Blob URL → createElement anchor → click → remove + revokeObjectURL. Estado `exportando` durante descarga.
- **Errores**: `useEffect` que observa `.error` de cada slice y llama `showError(error.message)` via `useToast()`.
- **Formateo de periodos**: Usar `formatPeriodoComparacion()` de `utils.ts` para mostrar labels legibles en KPIs (ej. `anio_anterior` → `ano anterior`).

### Paso 6: Estilos

**Dos niveles de CSS:**

1. **Estilos compartidos** (`src/styles/cuadroMando.css`): Contiene clases base reutilizables entre dashboards: `.cuadro-mando-header`, `.cuadro-mando-body`, `.cuadro-mando-footer`, `.cuadro-mando-card`, `.cuadro-mando-tabla`, `.cuadro-mando-anio-*`, `.cuadro-mando-mes-*`, animacion `fadeUp`, scrollbar custom, tooltips `.cuadro-mando-callout-half`. Importar desde el componente principal con `import '../../styles/cuadroMando.css'`.

2. **Estilos especificos del dashboard** (`src/pages/CuadroMando{Dominio}/styles.css`): Contiene clases prefijadas `cuadro-mando-{dominio}-*` para layout especifico (ej. `cuadro-mando-ps-kpi-grid`, `cuadro-mando-ps-row2`), overrides de compactacion para sidebar/row3, y clases utilitarias de celdas de tabla (`cuadro-mando-td`, `cuadro-mando-td--right`, `cuadro-mando-td--bold`, `cuadro-mando-td--total`, `cuadro-mando-td--detail`, `cuadro-mando-td--var`, `cuadro-mando-td--diff`). Importar con `import './styles.css'`.

**Reemplazar estilos inline repetitivos con clases CSS utilitarias.** Si un estilo se repite en multiples celdas/rows, crear una clase en el CSS del dashboard.

Colores siempre desde `cuadroMandoColors.ts`.

### Paso 7: Ruta

Importar desde el barrel (`index.ts`) y agregar `<Route path="cuadroMando{dominio}" element={<CuadroMando{Dominio} />} />` en `ContentRouter.tsx` del microfrontend correspondiente.

### Paso 8: Barrel Export

Crear `src/pages/CuadroMando{Dominio}/index.ts` exportando el componente principal.

## Libreria de Graficos

Usar EXCLUSIVAMENTE `@fluentui/react-charts` v9:

| Componente | Uso |
|---|---|
| `GroupedVerticalBarChart` | Comparacion mensual (actual vs anterior, barras agrupadas) |
| `VerticalStackedBarChart` | Series apiladas por periodo (ej. cubiertos + no cubiertos) |
| `VerticalBarChart` | Evolucion historica anual |
| `DonutChart` | Distribucion por categoria/tipo |

**Tooltips**: Usar `calloutProps` con tipo `Partial<ChartPopoverProps>` y `styles: { calloutContentRoot: 'cuadro-mando-callout-half' }`. El tipo `PopoverComponentStyles` acepta strings como nombres de clase CSS. **NO usar `className`** en `calloutProps` — esa propiedad no existe en `ChartPopoverProps`.

## Utilidades a Reutilizar

**`utils/cuadroMandoColors.ts`**: `CUADRO_MANDO_COLORS` (paleta: RED, TEAL, AMBER, PURPLE, GREEN, TEAL2, DARK, DARK_LIGHT, GRAY, GRAY_LIGHT, BORDER...), `CHART_SERIES_COLORS` (5 colores para series), `VARIATION_COLORS` (positivo/negativo), `ALERT_COLORS` (critical/high/medium/low/none), `MONTHS_FULL`/`MONTHS_SHORT` (espanol), `getAlertLevel(value, max)`, `getAlertColor(value, max, alpha?)`, `getAlertBgColor(value, max)`.

**`utils/utils.ts`**: `calculatePercentage(current, previous)`, `formatDate(date, locale='es-PE')`, `formatPeriodoComparacion(periodo)`.

**`utils/dateTimeHelper.tsx`**: `getCurrentYear()`, `getCurrentMonth()`.

**`common/app.constants.ts`**: `IResponseState<T>` → `{ data?: T | null; error: ApiError | null; loading: boolean }`.

## Reglas Estrictas

### PROHIBIDO
1. **NO usar `GUID_REGEX`** — No definir ni copiar validacion de GUIDs. Usar `normalizeEventoTipoId()` de `utils.ts` o dejar al backend.
2. **NO usar otras librerias de graficos** — No recharts, Chart.js, Victory, d3, nivo. Solo `@fluentui/react-charts`.
3. **NO usar Redux, Zustand, Context ni MobX** — Usar `useState` + hook personalizado.
4. **NO usar form libraries** para filtros — Usar Fluent UI `Dropdown` + controles directos.
5. **NO usar `any` explicito** — Tipar todo con interfaces. Usar `unknown` donde sea necesario.
6. **NO hardcodear colores** — Usar `CUADRO_MANDO_COLORS` y `CHART_SERIES_COLORS`.
7. **NO crear instancias axios directas** — Usar las de `services/http/apiClient.ts`.
8. **NO usar indices de array como `key`** (`key={i}`) — Usar identificadores del dominio (`puestoId`, `nombrePuesto`, etc.).
9. **NO crear funciones de recarga duplicadas** — Si `recargarPorAnio` es identico a `cargarDashboard`, no crearlo.
10. **NO omitir cleanup en `useEffect`** — `cancelAnimationFrame`, `observer.disconnect()`, etc. siempre deben tener su return cleanup.
11. **NO calcular valores derivados en el render sin `useMemo`** — Totales, maximos, porcentajes deben estar memoizados.
12. **NO usar estilos inline repetitivos** — Extraer a clases CSS utilitarias en el archivo de estilos del dashboard.
13. **NO usar `className` en `calloutProps` de charts** — Usar `styles: { calloutContentRoot: '...' }` que es el formato correcto de `PopoverComponentStyles`.

### OBLIGATORIO
1. Hook con `ejecutarCarga` + `Promise.allSettled` por dashboard
2. `IResponseState<T>` para cada slice de datos
3. `SetterMap` estabilizado con `useRef` (no como objeto literal en cada render)
4. Funciones de recarga parcial por dimension de filtro (solo los endpoints afectados)
5. `useCallback` en todas las funciones expuestas del hook
6. `ResizeObserver` para ancho responsive en cada grafico + cleanup con `observer.disconnect()`
7. `cancelAnimationFrame` en cleanup de animaciones de KPI cards
8. Patron ref para evitar closures obsoletos en useEffect inicial
9. `useMemo` para todos los calculos derivados (totales, maximos, porcentajes)
10. Keys con identificadores del dominio en todas las listas renderizadas
11. Animacion `fadeUp` con delays escalonados en KPI cards
12. Prop `invertVariation` en KPI cards donde subir puede ser bueno o malo segun contexto
13. Layout header/body/footer con clases `cuadro-mando-*`
14. Estilos compartidos en `src/styles/cuadroMando.css`, especificos en `styles.css` del dashboard
15. Clases CSS utilitarias para celdas de tabla (`cuadro-mando-td--*`)
16. Locale `es-PE` para numeros (`toLocaleString('es-PE')`) y fechas
17. Empty states en tablas cuando no hay datos
18. `getCurrentYear()` / `getCurrentMonth()` de `dateTimeHelper.tsx` para filtros iniciales
19. Exportacion Excel con patron blob URL + cleanup
