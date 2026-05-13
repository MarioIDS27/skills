# Pre-PR Review — IDSLatam Services Backend

Revisa el codigo del branch actual contra los patrones reales del proyecto antes de hacer PR.

## Proceso

1. **Obtener cambios del branch:** `git diff main...HEAD --name-only` para ver archivos modificados.
2. **Leer cada archivo nuevo/modificado** relevante (controllers, commands, queries, validators, DTOs, repos, migrations).
3. **Verificar cada seccion del checklist** que aplique a los cambios.
4. **Reportar resultados:** listar OK / FALLA por seccion. Para cada falla, indicar archivo y linea.
5. **Si hay red flags:** bloquear y pedir correccion antes de continuar con el PR.

---

## Checklist

### Controllers
- Hereda de `Controller` (NO `ControllerBase`)
- Inyecta solo `IMediator`; valida con `?? throw new ArgumentNullException(...)`
- Ruta: `[Route("api/v1/[controller]")]`
- Cada endpoint: atributo HTTP + `Name` descriptivo + `[ProducesResponseType]`
- Retorna solo `Ok(result)`; nunca logica de negocio

### Commands
- Naming: `{Verbo}{Entidad}Command` — verbos: Crear, Editar, Eliminar, Agregar
- Handler implementa `IRequestHandler<TCommand, TResponse>`
- Tiene `private readonly DateTimeHelper _dateTimeHelper = new()`
- Usa transaccion: `using (var tx = await _repo.BeginTransactionAsync())`
- Acumula errores en `List<ValidationFailure> failures`; lanza con `if (failures.Any()) throw new FluentValidation.ValidationException(failures)`
- Entidades nuevas: `Created = _dateTimeHelper.DateTimePst()` (nunca `DateTime.Now`)
- Termina con `await tx.CommitAsync()` antes del return

### Queries
- Handler tiene `private readonly DateTimeHelper _dateTimeHelper = new()` y `PageHelper _pageHelper = new()`
- Proyecta con LINQ `select new DTO { }` — **nunca AutoMapper**
- Todos los predicados incluyen `x.Deleted == null`
- Predicados con prefijo `f`: `fGrupo`, `fRuta`
- `DataCollection`: establece `Items`, `Total`, `Page`, `Pages`
- Filtra por `CustomerId` / `CuentaId` en operaciones multi-tenant

### Validators (FluentValidation)
- Hereda de `AbstractValidator<T>`
- Campos requeridos primero (`NotEmpty`, `NotNull`, `NotEqual(Guid.Empty)`)
- Validaciones async al final con `.MustAsync(...)`; verifican `entity != null && entity.Deleted == null`
- Mensajes en espanol

### DTOs
- Constructor vacio: `public XxxDTO() { }`
- Propiedades en **PascalCase** (nunca camelCase)
- Response DTOs heredan de `BaseResponseDTO` e incluyen el ID creado
- Sin atributos `[JsonPropertyName]` ni `required`

### Entity Framework
- Entidades heredan de `EntityBase`
- Schema `gbl` (o `inc` para reportes/incidentes) en `ToTable`
- FK con `DeleteBehavior.NoAction` siempre
- `EntityConfiguration` recibe `EntityTypeBuilder<T>` en constructor; no hereda `IEntityTypeConfiguration<T>`

### Repositorios & DI
- Interfaz vacia en `Application/Repositories/` hereda `IRepositoryBase<T>`
- Implementacion en `Infrastructure/Repositories/` hereda `RepositoryBase<T>`
- **Registrado** en `InfrastructureServiceRegistration` como `AddScoped<IXxxRepository, XxxRepository>`

---

## Red Flags — Bloquear PR si

- No compila
- `DeleteBehavior.Cascade` en cualquier FK
- Queries sin `x.Deleted == null`
- DTOs con propiedades en camelCase
- Controller inyecta repositorio directamente
- Command sin transaccion o sin `CommitAsync`
- AutoMapper usado en proyecciones de queries
- `DateTime.Now` en lugar de `_dateTimeHelper.DateTimePst()`
- Nuevo repositorio no registrado en DI
- Migration con schema incorrecto o `DeleteBehavior.Cascade`
- Entidad expuesta directamente como respuesta (sin proyectar a DTO)

---

## Reglas

- Reportar en espanol.
- Indicar archivo:linea para cada falla.
- Si no hay cambios en una categoria, omitirla del reporte.
- Inconsistencias legacy conocidas (transacciones en queries, calcularPages en camelCase) no se reportan como falla.
