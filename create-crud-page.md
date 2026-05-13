# Skill: create-crud-page

Genera páginas CRUD completas para microfrontends del monorepo. Crea listado con tabla, drawers crear/editar/eliminar, service, interface y validaciones. Detecta automáticamente el sabor de componentes de la app destino.

## Uso

```bash
/create-crud-page <App>: <Entidad> - <campos> - endpoint: <Endpoint> - axios: <axiosInstance>
```

### Ejemplo

```bash
/create-crud-page Configuracion: Vehiculo - nombre:string:Nombre, placa:string:Placa, estado:boolean:Estado - endpoint: Vehiculo - axios: axiosAdministracion
```

### Parámetros

| Parámetro | Descripción | Ejemplo |
|-----------|-------------|---------|
| **App** | Nombre de la app destino en `MicroFrontend/Aplicaciones/` | `Configuracion`, `GestionDocumentos`, `SoporteTicket` |
| **Entidad** | Nombre en PascalCase de la entidad | `Vehiculo`, `Concepto`, `Documento` |
| **Campos** | Lista `campo:tipo:label` separados por coma | `nombre:string:Nombre, codigo:number:Código` |
| **Endpoint** | Base del endpoint backend | `Vehiculo` → genera `/Vehiculo/Crear`, `/Vehiculo/Editar`, etc. |
| **Axios** | Instancia axios o enum de servicio | `axiosAdministracion` (Sabor A) o `Services.Aaa` (Sabor B) |

### Tipos de campo soportados

| Tipo | Componente Fluent UI | Default |
|------|---------------------|---------|
| `string` | `<Input />` | `null` |
| `number` | `<Input type="number" />` | `null` |
| `boolean` | `<Switch />` o `<Checkbox />` | `false` |
| `date` | `<Input type="date" />` | `null` |
| `select` | `<Dropdown />` | `null` |
| `textarea` | `<Textarea />` | `null` |

## Qué genera

Para una entidad `Vehiculo` en la app `Configuracion`, genera **7 archivos** (~485 líneas):

```
src/
├── pages/vehiculos/
│   ├── Vehiculos.tsx                          # Listado con tabla, búsqueda, botones
│   ├── drawers/
│   │   ├── CrearVehiculo.tsx                  # Drawer con formulario + validación
│   │   ├── EditarVehiculo.tsx                 # Drawer con form pre-poblado
│   │   └── EliminarVehiculo.tsx               # Drawer de confirmación
│   └── validations/
│       └── vehiculo.validations.ts            # Reglas ValiValid
├── interfaces/
│   └── IVehiculo.ts                           # DTOs + initial values
└── services/vehiculos/
    └── VehiculoService.ts                     # Funciones listar/crear/editar/eliminar
```

| Archivo | Contenido | Líneas aprox. |
|---------|-----------|---------------|
| `IVehiculo.ts` | `IVehiculo`, `IVehiculoWrite`, `initialVehiculoWrite()`, `initialVehiculo()` | ~20 |
| `vehiculo.validations.ts` | Reglas `ValiValid` por campo requerido | ~15 |
| `VehiculoService.ts` | Clase con métodos estáticos CRUD tipados | ~35 |
| `Vehiculos.tsx` | Página con tabla, búsqueda, botones Agregar/Actualizar, columna Opciones | ~150 |
| `CrearVehiculo.tsx` | Drawer con `Field`+`Input` por campo, validación, submit con loading | ~100 |
| `EditarVehiculo.tsx` | Drawer que pre-pobla desde item seleccionado, edita y recarga | ~110 |
| `EliminarVehiculo.tsx` | Drawer de confirmación simple con mensaje y botón Eliminar | ~50 |

## Detección automática de sabor

El skill detecta cuál patrón de componentes usa la app destino buscando imports en archivos existentes:

| Aspecto | Sabor A (Configuracion, Planilla) | Sabor B (GestionDocumentos, SoporteTicket, Ssgg) |
|---------|----------------------------------|--------------------------------------------------|
| Tabla | `TablaV9` de `components/tablas/` | `CommonTableComponent` de `common/components/table/` |
| Header | `CabeceraComponent` | `CommonHeaderComponent` |
| Botones menú | `ButtonMenuComponent` | `CommonButtonMenuComponent` |
| Drawer | `GenericDrawer` local | `CommonDrawerComponent` |
| Iconos | `useIcons` | `useIconsCatalogo` |
| Layout | Inline flex divs | `CommonPanelLayout` wrapper |
| Servicios | Axios instances directas (`axiosAdministracion.get(...)`) | Factory pattern (`getRequest(Services.X, ...)`) |

## Flujo del skill

```
1. Parsear entrada → extraer App, Entidad, Campos, Endpoint, Axios
2. Detectar sabor → buscar imports en archivos existentes de la app
3. Leer referencias → 1 listado + 1 drawer existente para copiar import paths
4. Diseñar CRUD → presentar resumen y pedir confirmación
5. Generar 7 archivos → interfaces, validations, service, listado, 3 drawers
6. Sugerir ruta → línea para agregar en ContentRouter.tsx
7. Resumen → tabla de archivos generados
```

## Patrones que sigue

### Listado (basado en `Clientes.tsx` y `GrupListadoGruposComponent.tsx`)
- `useState` para `data[]`, `loading`, `selectedItem`, `openDrawer{Crear|Editar|Eliminar}`
- `useEffect` con fetch inicial
- `IColumn[]` con columnas de datos + columna "Opciones" con `ButtonMenuComponent`
- `createOptionButtons()` retorna `IButtonGroup[]` con Editar y Eliminar
- `_leftButton` con Agregar y Actualizar

### Drawers (basado en `GrupCrearGrupoComponent.tsx` y `PanelAgregarSolucion.tsx`)
- Props: `IDrawerProperty` (`openPanel`, `setOpenPanel`, `reload`)
- Estado: `form` con `useState<IEntidadWrite>(initialEntidadWrite())`
- Validación: `ValiValid` con `managerValidation.handleChange(field, value, setForm, setErrors)`
- Submit: `setLoading(true)` → service call → success message → `setTimeout(close + reload, 3000)`
- Campos: `<Field label required validationState validationMessage>` + `<Input />`

### Service (basado en `ClienteService.ts` y `ArchivoServices.ts`)
- Clase con métodos estáticos: `listar()`, `crear()`, `editar()`, `eliminar()`
- Sabor A: retorna `Promise<AxiosResponse>` usando axios instance directo
- Sabor B: retorna `Promise<IResponseHttp<T>>` usando factory `getRequest`/`postRequest`

### Validaciones (basado en `grup.CrearGrupoValidations.ts`)
- `BuilderValidationConfig<IEntidadWrite>` con `ValidationType.Required` por campo

## Reglas

- **NUNCA inventar import paths** — siempre leer archivos existentes de la app para obtener los paths reales
- **PascalCase** para componentes/clases, **camelCase** para variables/carpetas
- **Interface prefix con `I`**: `IVehiculo`, `IVehiculoWrite`
- **Siempre usar `ValiValid`** para validación de formularios
- Sabor A: definir interface de drawer local; Sabor B: reusar `IDrawerProperty` de `common/`
- Verificar que el axios instance existe en `Http-Common.ts` antes de generar

## Verificación post-generación

- [ ] `npm run build` compila sin errores
- [ ] Los imports apuntan a componentes que existen en la app
- [ ] El axios instance referenciado existe en `Http-Common.ts`
- [ ] Las interfaces tienen funciones `initial*()`
- [ ] Las validaciones cubren los campos requeridos
- [ ] `ContentRouter.tsx` incluye la nueva ruta
