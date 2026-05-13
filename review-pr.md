# Revisión de Pull Request

## Uso
`/review <número-PR>`

---

## 1. Flujo de Ejecución Obligatorio

Ejecuta estos pasos en orden. No omitas ninguno.

```bash
# Paso 1 — Obtener metadatos del PR
gh pr view $ARGUMENTS --json title,body,additions,deletions,changedFiles,baseRefName,headRefName

# Paso 2 — Obtener archivos cambiados
gh pr diff $ARGUMENTS

# Paso 3 — Publicar el review como comentario
gh pr comment $ARGUMENTS --body "..."
```

Al finalizar, responde **únicamente** con:
> ✅ Review publicado en PR #`$ARGUMENTS`

Sin más texto. Sin reproducir el review en la conversación.

---

## 2. Contexto del Proyecto

- **Stack**: .NET 9, ASP.NET Core Web API
- **ORM**: Entity Framework Core 7.x
- **Base de datos**: SQL Server (schemas por servicio, ej: `gbl`, `aaa`)
- **Arquitectura**: Clean Architecture — Api → Application → Core ← Infrastructure
- **Patrones**: CQRS con MediatR, FluentValidation, AutoMapper, JWT Bearer
- **Autenticación**: `.RequireAuthorization()` global en `Program.cs`, `ICurrentUserService` para usuario actual
- **Comunicación entre servicios**: HTTP proxies definidos en `Application/Proxies`

---

## 3. Clasificación de Hallazgos

| Severidad | Símbolo | Criterio | Acción |
|-----------|---------|----------|--------|
| Bloqueante | 🔴 | Bug que rompe producción, vulnerabilidad de seguridad, pérdida de datos | RECHAZAR |
| Importante | 🟠 | Performance crítica, inconsistencia funcional, violación de contrato | CAMBIOS REQUERIDOS |
| Menor | 🟡 | Código mejorable sin impacto en producción | CAMBIOS MENORES |
| Positivo | ✅ | Buenas prácticas, mejora real implementada | RECONOCER |

---

## 4. Proceso de Revisión (máx. 5 minutos)

### 4.1 Análisis Rápido (1 min)
- ¿Qué modifica este PR? (feature, fix, refactor, migración)
- ¿Crea endpoints nuevos? → obligatorio verificar autorización
- ¿Toca migraciones EF? → verificar rollback y schema

### 4.2 Búsqueda de Bloqueantes (2 min)

Escanea SOLO estos patrones críticos:

**Seguridad**
- ❌ SQL dinámico sin parametrizar (`$"SELECT ... {input}"`, interpolación en queries EF)
- ❌ Credenciales o connection strings hardcodeados en código fuente
- ❌ Endpoints sin `[Authorize]` cuando la ruta no está excluida globalmente
- ❌ Acceso a recursos sin validar que pertenecen al tenant/usuario actual

**Estabilidad**
- ❌ `.FirstOrDefault()` / `.SingleOrDefault()` sin null-check posterior
- ❌ `catch` vacío o que solo hace `log` sin relanzar ni retornar error
- ❌ Transacciones EF sin `try/catch` + rollback en errores
- ❌ `.ToList()` sin filtros en tablas potencialmente grandes

**Performance**
- ❌ N+1: llamadas a proxy o repositorio dentro de un `foreach`
- ❌ Carga sin `.AsNoTracking()` en queries de solo lectura

**Validación**
- ❌ Endpoints públicos sin validación de input (FluentValidation ausente para el comando/query)
- ❌ Uso de `object` / `dynamic` sin tipado donde se espera tipo concreto

### 4.3 Validación de Autorización (1 min)

Si el PR crea o modifica endpoints:
- ¿Tiene `[Authorize]` explícito o está cubierto por la política global?
- ¿Filtra datos por tenant usando `ICurrentUserService`?
- ¿Valida que el recurso solicitado pertenece al usuario/empresa del token?

### 4.4 Migraciones EF (30 seg, solo si aplica)

- ¿La migración es reversible (`Down` implementado)?
- ¿Agrega columnas NOT NULL sin valor por defecto en tablas existentes?
- ¿El schema corresponde al servicio correcto?

---

## 5. Plantilla del Comentario

Usa exactamente esta estructura al construir el body del `gh pr comment`:

```
## 🔍 Code Review — PR #{número}

**{título del PR}**
_{rama origen_ → _rama destino}_
_{N archivos cambiados, +X / -Y líneas}_

---

### Hallazgos

| Archivo | Línea | Tipo | Hallazgo |
|---------|-------|------|----------|
| `Handler.cs` | 45 | 🔴 | `.FirstOrDefault()` sin null-check |
| `Query.cs` | 120 | 🟠 | N+1 en loop de llamadas proxy |
| `Program.cs` | 88 | ✅ | DI correctamente configurado |

---

### Detalle de hallazgos 🔴 / 🟠

**`Handler.cs:45` — NullReference potencial**

```csharp
// ❌ Problema
var item = list.FirstOrDefault();
item.Nombre; // NullReferenceException si lista vacía

// ✅ Solución
var item = list.FirstOrDefault()
    ?? throw new KeyNotFoundException("Elemento no encontrado");
```

---

### Veredicto

**🔴 RECHAZAR** / **🟠 CAMBIOS REQUERIDOS** / **🟡 CAMBIOS MENORES** / **🟢 APROBAR**

**Cambios requeridos antes del merge:**
1. ...
2. ...

_(Review generado por Claude Code)_
```

---

## 6. Casos Especiales

**PR de subsanación** (corrección de observaciones previas):
- ✅ Corrección aplicada correctamente
- ⚠️ Corrección parcial — indica qué falta exactamente
- ❌ No corregido — cita el hallazgo original

**PR de refactor**:
- Verificar que no introduce NullReference, N+1 ni rompe contratos de API
- Aceptar cambios de estilo si mejoran legibilidad sin añadir riesgo

**PR de migración EF**:
- Validar que `Down()` revierte correctamente
- Detectar columnas NOT NULL sin default en tablas con datos existentes
- Confirmar que el schema corresponde al servicio

**PR de configuración / deploy**:
- Validar solo valores críticos (connection strings, keys, URLs)
- No bloquear por formato si la lógica es correcta

---

## 7. Lo que NO revisar

- ❌ Nombres de variables o métodos (salvo que sean engañosos)
- ❌ Orden de métodos o propiedades
- ❌ Comentarios XML incompletos
- ❌ Sugerencias de arquitectura "ideal" no solicitadas
- ❌ Test coverage (salvo que el PR elimine tests existentes)
- ❌ Estilo de llaves, espaciado, imports no usados

---

## 8. Notas

- **Idioma**: español en todo el output
- **El review NO se muestra en la conversación** — solo se publica vía `gh pr comment`
- Usa syntax highlighting en todos los bloques de código (`csharp`)
- Compara código anterior vs nuevo cuando ilustre claramente el problema
- La única respuesta visible al usuario es la línea de confirmación del paso 3
