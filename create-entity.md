---
description: Genera entidades, repositorios, configuracion, DI y DbContext para un requerimiento. Soporta multiples entidades a la vez. Detecta automaticamente el patron del DbContext.
argument-hint: "Servicio: descripcion del requerimiento [--full|--crud|--with-controller]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

Requerimiento: **$ARGUMENTS**

## PASO 0 — Parsear entrada

Extraer de `$ARGUMENTS`:
- **SERVICIO**: texto antes de `:` o `/` (ej: `Administracion`, `AAA`)
- **REQUERIMIENTO**: texto libre despues — describe las tablas/entidades a crear
- **FLAGS** opcionales: `--full` (CQRS + Controller), `--crud` (solo Commands), `--with-controller`. Sin flag = solo infraestructura

Localizar root buscando `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Core/` desde CWD y directorios de trabajo adicionales.

## PASO 1 — Detectar patron y leer ejemplos

**1a. Leer DbContext:** `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Infrastructure/{SERVICIO}DbContext.cs`
- Contiene `ApplyConfigurationsFromAssembly` → **PATRON B** (IEntityTypeConfiguration, auto-discovery)
- Contiene `ModelConfig(ModelBuilder` → **PATRON A** (constructor-based, registro manual)
- Extraer `DEFAULT_SCHEMA` de `HasDefaultSchema("xxx")`

**1b. Leer 1 ejemplo existente** de cada tipo en el mismo servicio para copiar convenciones:
- 1 Entity existente → estructura, usings, constructor vacio
- 1 Configuration → patron de clase, usings, relaciones
- 1 Repository interface → using de IRepositoryBase
- 1 Repository implementation → estilo constructor (traditional vs primary)

Informar: `Patron: {A|B}, Schema: {schema}`

## PASO 2 — Diseñar entidades (PEDIR CONFIRMACION)

Analizar el requerimiento y diseñar las entidades. Presentar al usuario una tabla resumen ANTES de generar:

```
| # | Entidad | PK | Propiedades principales | Relaciones | Schema/Tabla |
```

Incluir:
- Nombre de cada entidad y su PK
- Propiedades con tipos (Guid, string, bool, DateTime, int, etc.)
- Relaciones entre entidades (FK, navigations, colecciones)
- Schema y nombre de tabla propuesto

**Esperar confirmacion del usuario.** Si pide cambios, ajustar antes de continuar.

## PASO 3 — Generar archivos

Para CADA entidad aprobada, crear los siguientes archivos replicando exactamente el estilo de los ejemplos leidos en Paso 1:

### 3a. Entity
**Ruta:** `Core/Entities/{ENTIDAD}.cs`
- Hereda `EntityBase` (de `IDSLatam.Common.Core.Base`)
- PK: `Guid {ENTIDAD}Id`
- Constructor vacio si los ejemplos lo tienen
- Navigation properties para FKs
- `List<T>` inicializadas con `new List<T>()` para colecciones

### 3b. Repository Interface
**Ruta:** `Application/Repositories/I{ENTIDAD}Repository.cs`
- Hereda `IRepositoryBase<{ENTIDAD}>`. Copiar usings del ejemplo.

### 3c. Repository Implementation
**Ruta:** `Infrastructure/Repositories/{ENTIDAD}Repository.cs`
- Hereda `RepositoryBase<{ENTIDAD}>`, implementa `I{ENTIDAD}Repository`
- Constructor recibe `{SERVICIO}DbContext`. Copiar estilo del ejemplo.

### 3d. Configuration (depende del patron)
**Ruta:** `Infrastructure/Configuration/{ENTIDAD}Configuration.cs`

**PATRON A:** Constructor con `EntityTypeBuilder<{ENTIDAD}>`. Configurar: `HasKey`, `IsRequired` por propiedad, `HasOne/HasMany` con `DeleteBehavior.NoAction`.

**PATRON B:** Implementa `IEntityTypeConfiguration<{ENTIDAD}>` con `Configure()`. Incluir `ToTable("{TABLA}", "{SCHEMA}")` dentro del metodo. Mismo contenido de config.

### 3e. Modificar DbContext
Leer el archivo antes de editar.

**AMBOS PATRONES:** Agregar DbSet por cada entidad:
```csharp
public DbSet<{ENTIDAD}> {PLURAL} { get; set; }
```

**SOLO PATRON A** (en `ModelConfig`): por cada entidad agregar:
```csharp
new {ENTIDAD}Configuration(modelBuilder.Entity<{ENTIDAD}>());
modelBuilder.Entity<{ENTIDAD}>().ToTable("{PLURAL}", "{SCHEMA}");
```
Si la tabla existe en otro microservicio, agregar `b => b.ExcludeFromMigrations()`.

**PATRON B:** Solo los DbSets. La configuracion se descubre automaticamente.

Agregar usings necesarios para los namespaces de las entidades nuevas.

### 3f. Registrar DI
**Archivo:** `Infrastructure/InfrastructureServiceRegistration.cs`
Agregar un `AddScoped` por cada entidad:
```csharp
services.AddScoped<I{ENTIDAD}Repository, {ENTIDAD}Repository>();
```
Agregar usings necesarios.

## PASO 4 — CQRS (solo si flags)

Si se especifico `--crud`, `--full` o `--with-controller`:

Leer **1 Command Handler existente** y **1 Query Handler existente** del servicio como referencia.

**Por cada entidad** (segun flag):

**--crud o --full:** Crear en `Application/Commands/{ENTIDAD}/`:
- `Crear/` → Command + Validator + Handler (transaccion, validar FKs, AddAsync)
- `Actualizar/` → Command + Validator + Handler (buscar por PK, UpdateAsync, NO setear Modified)
- `Eliminar/` → Command + Validator + Handler (soft delete: `Deleted = DateTimePst()`)

**--full:** Crear en `Application/Queries/{ENTIDAD}/`:
- `Listar/` → Query (Skip/Take) + DTO + Handler (predicado `Deleted==null`, DataCollection)
- `ObtenerPorId/` → Query + DTO + Handler (retornar null si no existe)

**--full o --with-controller:** Crear en `Api/Controllers/`:
- `{ENTIDAD}Controller.cs` → hereda `Controller`, inyecta `IMediator`, endpoints segun operaciones

**Reglas CQRS:**
- Transacciones: `using (var tx = await _repo.BeginTransactionAsync())`
- Validaciones: `List<ValidationFailure>` + `throw new ValidationException(failures)`
- Predicados: siempre `x.Deleted == null`, NO usar `.Compile()`
- DeleteBehavior: siempre `NoAction`
- Controller: solo `IMediator`, retornar `Ok(result)`, NO try-catch

## PASO 5 — Resumen

Mostrar tabla de todos los archivos creados/modificados:

| Tipo | Archivos |
|------|----------|
| Entidades | `{lista de .cs creados}` |
| Repositories (Interface) | `{lista}` |
| Repositories (Impl) | `{lista}` |
| Configurations | `{lista}` |
| DbContext | Modificado — {N} DbSets + {detalle patron} |
| DI Registration | Modificado — {N} AddScoped |
| Commands/Queries/Controller | `{si aplica}` |

Indicar patron detectado (A/B), schema usado, y que el usuario debe ejecutar las migraciones manualmente.

---

## REGLAS

**Entidades:**
- Heredar `EntityBase`, PK como `Guid {Entidad}Id`
- Constructor vacio si el servicio lo usa
- `List<T>` inicializadas: `= new List<T>()`
- Nullable (`?`) para propiedades opcionales

**Relaciones:**
- SIEMPRE `OnDelete(DeleteBehavior.NoAction)`
- FK = propiedad Guid/Guid? terminada en "Id" + navigation property correspondiente
- FK sin navigation property → solo `.IsRequired()`, NO modelar relacion
- Si la relacion ya esta configurada en otra Configuration → NO duplicar (comentar referencia)

**Naming:**
- camelCase para variables: `ProcesoEmpresa` → `procesoEmpresa`
- Plural espanol para DbSets/tablas: `EstadoEmpresa` → `EstadosEmpresas`
- Namespace: `IDSLatam.Service.{SERVICIO}.{Layer}`

**Migraciones (referencia):**
```bash
cd {SERVICIO}/IDSLatam.Service.{SERVICIO}.Infrastructure
dotnet ef --startup-project ../IDSLatam.Service.{SERVICIO}.Api/IDSLatam.Service.{SERVICIO}.Api.csproj migrations add {Nombre}
dotnet ef --startup-project ../IDSLatam.Service.{SERVICIO}.Api/IDSLatam.Service.{SERVICIO}.Api.csproj migrations script --idempotent -o MigrationScripts/{archivo}.sql
```

Archivos generados por migracion (4):
- `Migrations/{timestamp}_{Nombre}.cs`
- `Migrations/{timestamp}_{Nombre}.Designer.cs`
- `Migrations/{SERVICIO}DbContextModelSnapshot.cs`
- `MigrationScripts/{archivo}.sql`
