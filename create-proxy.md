---
description: Genera proxies inter-servicio con Refit. Crea interfaz API, proxy, DTOs, queries y registro DI. Estandariza al patron optimo (AddRefitClient + AuthHeaderHandler).
user-intent: "Servicio: necesito llamar al endpoint [verbo] [ruta] del servicio [Target] que retorna [tipo]"
---

Requerimiento: **$ARGUMENTS**

## PASO 0 — Parsear entrada

Extraer de `$ARGUMENTS`:
- **SERVICIO**: microservicio origen (el que LLAMA) — texto antes de `:` o `/`
- **TARGET**: microservicio destino (el que EXPONE el endpoint)
- **ENDPOINTS**: lista de endpoints a consumir (verbo HTTP + ruta + request type + response type)

Si el usuario no especifica endpoints detallados, preguntar. Ejemplo de entrada completa:
`SSGG: llamar GET /api/v1/Persona/{personaId} y POST /api/v1/Persona/ListarPersonasPorId del servicio Administracion`

Localizar root buscando `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Application/` desde CWD.

## PASO 1 — Detectar estado actual del servicio

**1a. AuthHeaderHandler:** Buscar `Proxies/Commons/AuthHeaderHandler.cs` en `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Application/`
- Existe → reutilizar
- No existe → se creara en Paso 3a

**1b. Proxy existente al target:** Buscar carpeta `Proxies/{TARGET}/`
- Existe → agregar metodos a archivos existentes, NO crear nuevos
- No existe → crear estructura completa

**1c. InfrastructureServiceRegistration:** Leer `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Infrastructure/InfrastructureServiceRegistration.cs`
- Buscar si ya tiene `AddRefitClient<I{TARGET}API>` registrado
- Buscar si ya tiene `services.AddTransient<AuthHeaderHandler>()`
- Anotar donde insertar nuevos registros

**1d. Referencia de estilo:** Si el servicio ya tiene proxies, leer 1 proxy existente para copiar convenciones de usings y namespaces.

**1e. URL en config:** Verificar que `Common/IDSLatam.Common.Core/Configurations/IDSLatamConfig.cs` tiene `Url{TARGET}`. Las URLs existentes incluyen: UrlAdministracion, UrlConfiguracion, UrlIdentidad, UrlDispositivo, UrlGlobal, UrlContrato, UrlSSOMA, UrlMovilizacion, UrlAsistencia, UrlKPIs, UrlValorSocial, UrlIntegracion, UrlControlAcceso, UrlGeneral, UrlAAA, UrlTeseo, UrlTalento, UrlGobierno, UrlVisita, UrlSSGG, UrlVehiculo, UrlSaludAsistencial, UrlFactura, UrlProyecto, UrlFuerzaLaboral, UrlADMS, UrlEntrenamiento, UrlAutorizacionInterna, UrlICMA, UrlEncuesta, UrlNotificacion, UrlSms.

Informar:
```
AuthHeaderHandler: {existe|a crear}
Proxy {TARGET}: {nuevo|agregar metodos}
Refit client DI: {registrado|a registrar}
URL config: Url{TARGET} = {existe|NO EXISTE — agregar a IDSLatamConfig}
```

## PASO 2 — Diseñar proxy (PEDIR CONFIRMACION)

Presentar tabla con los endpoints a generar:

```
| # | Verbo | Ruta API | Request Type | Response Type | Metodo Proxy |
```

Incluir:
- Verbo HTTP (GET, POST, PUT, DELETE)
- Ruta completa del endpoint (`/api/v1/Controller/Action`)
- Tipo de request (path param, query, body DTO)
- Tipo de response (DTO, List, DataCollection)
- Nombre del metodo en la interfaz proxy (con suffix `Async`)

Listar tambien los DTOs y Queries/Commands que se crearan.

**Esperar confirmacion del usuario.** Si pide cambios, ajustar antes de continuar.

## PASO 3 — Generar archivos

### 3a. AuthHeaderHandler (solo si NO existe)
**Ruta:** `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Application/Proxies/Commons/AuthHeaderHandler.cs`

```csharp
using System.Net.Http.Headers;
using IDSLatam.Service.{SERVICIO}.Application.Interface;

namespace IDSLatam.Service.{SERVICIO}.Application.Proxies.Commons
{
    public class AuthHeaderHandler : DelegatingHandler
    {
        private readonly ICurrentUserService _currentUserService;

        public AuthHeaderHandler(ICurrentUserService currentUserService)
        {
            _currentUserService = currentUserService;
        }

        protected override async Task<HttpResponseMessage> SendAsync(
            HttpRequestMessage request,
            CancellationToken cancellationToken)
        {
            request.Headers.Authorization =
                new AuthenticationHeaderValue("Bearer", _currentUserService.Token);

            return await base.SendAsync(request, cancellationToken);
        }
    }
}
```

Verificar que `ICurrentUserService` existe en `Application/Interface/` y que tiene propiedad `Token`. Si el servicio usa `Interfaces` en vez de `Interface`, ajustar el namespace.

### 3b. Interfaz Refit API
**Ruta:** `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Application/Proxies/{TARGET}/I{TARGET}API.cs`

```csharp
using Refit;
// usings de DTOs y Queries

namespace IDSLatam.Service.{SERVICIO}.Application.Proxies.{TARGET}
{
    public interface I{TARGET}API
    {
        [Get("/api/v1/Controller/{id}")]
        Task<ResponseDTO> MetodoGet(Guid id);

        [Post("/api/v1/Controller/Action")]
        Task<ResponseDTO> MetodoPost([Body] RequestDTO cmd);
    }
}
```

Reglas de atributos Refit:
- `[Get]` / `[Post]` / `[Put]` / `[Delete]` con ruta completa
- Path params: directos en la firma (`Guid id`)
- Body: decorar con `[Body]`
- Query string: decorar con `[Query]`
- Return: siempre `Task<T>`

### 3c. Proxy (interfaz + implementacion en mismo archivo)
**Ruta:** `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Application/Proxies/{TARGET}/{TARGET}Proxy.cs`

```csharp
// usings de DTOs y Queries

namespace IDSLatam.Service.{SERVICIO}.Application.Proxies.{TARGET}
{
    public interface I{TARGET}Proxy
    {
        Task<ResponseDTO> MetodoGetAsync(Guid id);
        Task<ResponseDTO> MetodoPostAsync(RequestDTO cmd);
    }

    public class {TARGET}Proxy : I{TARGET}Proxy
    {
        private readonly I{TARGET}API _api;

        public {TARGET}Proxy(I{TARGET}API api)
        {
            _api = api;
        }

        public Task<ResponseDTO> MetodoGetAsync(Guid id)
            => _api.MetodoGet(id);

        public Task<ResponseDTO> MetodoPostAsync(RequestDTO cmd)
            => _api.MetodoPost(cmd);
    }
}
```

Reglas:
- Interfaz `I{TARGET}Proxy` define el contrato de negocio — metodos con suffix `Async`
- Implementacion inyecta `I{TARGET}API` (Refit) y delega directamente
- Metodos son expression-bodied (`=>`) cuando solo delegan

### 3d. DTOs
**Ruta:** `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Application/Proxies/{TARGET}/DTO/{NombreDTO}.cs`

Un archivo por cada DTO de response. Estructura:
```csharp
namespace IDSLatam.Service.{SERVICIO}.Application.Proxies.{TARGET}.DTO
{
    public class {NombreDTO}
    {
        public Guid EntidadId { get; set; }
        public string? Nombre { get; set; }
        public string Codigo { get; set; } = string.Empty;
    }
}
```

Reglas:
- Auto-properties
- `string?` para opcionales, `string` con `= string.Empty` para requeridos
- `Guid?` para FKs opcionales
- NO importar DTOs del servicio target — definir localmente
- Si el usuario proporciona la estructura del DTO, usarla. Si no, inferir de la ruta/nombre y preguntar

### 3e. Queries / Commands
**Ruta:** `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Application/Proxies/{TARGET}/Queries/{NombreQuery}.cs`
o `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Application/Proxies/{TARGET}/Command/{NombreCommand}.cs`

```csharp
namespace IDSLatam.Service.{SERVICIO}.Application.Proxies.{TARGET}.Queries
{
    public class {NombreQuery}
    {
        public string? Search { get; set; }
        public int Skip { get; set; } = 0;
        public int Take { get; set; } = 50;
    }
}
```

Reglas:
- Usar `Queries/` para requests de lectura, `Command/` para requests de escritura
- Constructor vacio implicito
- Defaults: `Skip = 0`, `Take = 50` para paginados
- Colecciones inicializadas: `= []`

## PASO 4 — Registrar DI

Leer `{SERVICIO}/IDSLatam.Service.{SERVICIO}.Infrastructure/InfrastructureServiceRegistration.cs` antes de editar.

**4a. Si NO tiene `services.AddTransient<AuthHeaderHandler>()`:**
Agregar:
```csharp
services.AddTransient<AuthHeaderHandler>();
```

**4b. Si NO tiene `APIIDSLatamConfig` configurado:**
Agregar:
```csharp
services.Configure<APIIDSLatamConfig>(opt => configuration.GetSection("APIIDSLatamConfig").Bind(opt));
```

**4c. Registrar Refit client** (si no existe para este target):
```csharp
services.AddRefitClient<I{TARGET}API>()
    .ConfigureHttpClient((sp, c) =>
    {
        var config = sp.GetRequiredService<IOptions<APIIDSLatamConfig>>().Value;
        c.BaseAddress = new Uri(config.Url{TARGET});
    })
    .AddHttpMessageHandler<AuthHeaderHandler>();
```

**4d. Registrar proxy:**
```csharp
services.AddScoped<I{TARGET}Proxy, {TARGET}Proxy>();
```

**4e. Agregar usings necesarios:**
```csharp
using Refit;
using Microsoft.Extensions.Options;
using IDSLatam.Common.Core.Configurations;
using IDSLatam.Service.{SERVICIO}.Application.Proxies.Commons;
using IDSLatam.Service.{SERVICIO}.Application.Proxies.{TARGET};
```

Si el servicio NO tiene el paquete `Refit.HttpClientFactory` referenciado en su `.csproj`, informar al usuario que debe agregarlo:
```xml
<PackageReference Include="Refit.HttpClientFactory" Version="7.0.0" />
```

## PASO 5 — Resumen

Mostrar tabla de todos los archivos creados/modificados:

| Tipo | Archivos |
|------|----------|
| AuthHeaderHandler | `{creado o ya existia}` |
| Refit API Interface | `I{TARGET}API.cs` |
| Proxy (Interface + Impl) | `{TARGET}Proxy.cs` |
| DTOs | `{lista de .cs}` |
| Queries/Commands | `{lista de .cs}` |
| DI Registration | Modificado — AddRefitClient + AddScoped |

Ejemplo de uso en un handler:
```csharp
// Inyectar en constructor
private readonly I{TARGET}Proxy _{target}Proxy;

// Usar en Handle()
var result = await _{target}Proxy.{Metodo}Async(query);
```

---

## REGLAS

**Patron DI:**
- SIEMPRE usar `AddRefitClient` + `AuthHeaderHandler`. NUNCA `RestService.For`
- `AuthHeaderHandler` registrado como `Transient`
- Refit clients registrados via `AddRefitClient<T>().ConfigureHttpClient().AddHttpMessageHandler<AuthHeaderHandler>()`
- Proxy registrado como `Scoped`

**AuthHeaderHandler:**
- Reutilizar si ya existe en el servicio. NUNCA duplicar
- Si existe pero con diferente namespace, usar el existente
- Patron exacto: `DelegatingHandler` que inyecta `ICurrentUserService.Token`

**Estructura de archivos:**
- Interfaz `I{TARGET}Proxy` + clase `{TARGET}Proxy` en el MISMO archivo
- `I{TARGET}API` en archivo separado (solo contiene la interfaz Refit)
- DTOs en subcarpeta `DTO/`
- Requests en subcarpeta `Queries/` o `Command/`
- Archivos compartidos en `Proxies/Commons/`

**Naming:**
- `I{TARGET}API` para la interfaz Refit (ej: `IAdministracionAPI`)
- `I{TARGET}Proxy` para la interfaz de negocio (ej: `IAdministracionProxy`)
- `{TARGET}Proxy` para la implementacion (ej: `AdministracionProxy`)
- Metodos en proxy con suffix `Async` (ej: `ObtenerPersonaPorIdAsync`)
- Metodos en Refit API SIN suffix Async (ej: `ObtenerPersonaPorId`)
- Namespace: `IDSLatam.Service.{SERVICIO}.Application.Proxies.{TARGET}`

**DTOs:**
- Definir LOCALMENTE en el proxy, NUNCA importar del proyecto del servicio target
- Copiar la estructura exacta del response del endpoint target
- Auto-properties con nullable types cuando corresponda

**Proxy existente:**
- Si ya existe `Proxies/{TARGET}/`, agregar metodos a los archivos existentes
- Agregar nuevos metodos a `I{TARGET}API`, `I{TARGET}Proxy`, y `{TARGET}Proxy`
- Agregar nuevos DTOs/Queries en las subcarpetas existentes
- NO duplicar registros DI que ya existen
