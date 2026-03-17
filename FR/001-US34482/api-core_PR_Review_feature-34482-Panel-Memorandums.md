# PR Review — feature/34482 vs release/3.0
## US: COR-GS2-CPA — Panel de Memorándums (Gestión y Revisión)

**Branch:** `feature/34482`  
**Base:** `release/3.0`  
**Commit:** `f698f82`  
**Fecha de revisión:** 17/03/2026  
**Revisor:** GitHub Copilot (análisis automatizado)

---

## 1. Resumen ejecutivo

Este PR implementa el **backend de lectura** del Panel de Memorándums: historial paginado con filtros, solicitudes pendientes de memo, y endpoint genérico de estados por tipo de entidad. El alcance es **parcial** respecto a la US completa; las funcionalidades de escritura (generar, actualizar, configurar fecha de sesión, cards de resumen, descarga de PDF, auditoría) quedan pendientes para una entrega posterior.

| Categoría | Cantidad |
|-----------|----------|
| Archivos modificados / creados | 17 |
| Líneas agregadas | +315 / -1 |
| Bugs críticos | 3 |
| Issues importantes | 3 |
| Issues menores | 3 |
| Observaciones de alcance | 1 |

---

## 2. Archivos modificados

| Archivo | Tipo de cambio |
|---------|---------------|
| `Core.Application/.../Mappers/CoreMappingProfile.cs` | Modificado — agrega mapping `MemorandumResponseDbRow → MemorandumResponseDto` |
| `Core.Application/.../Requests/MemorandumGetRequestDto.cs` | Nuevo — DTO de filtros para el historial |
| `Core.Application/.../Responses/MemorandumResponseDto.cs` | Nuevo — DTO de respuesta del historial |
| `Core.Application/.../Queries/MemorandumPaginatedGetQuery.cs` | Nuevo — query paginada del historial |
| `Core.Application/.../Queries/SolicitudProtectorPendienteMemoGetQuery.cs` | Nuevo — query de solicitudes sin memo |
| `Core.Application/.../Queries/EstadoConfiguracionGetByTipoEntidadQuery.cs` | Nuevo — query de estados por tipo entidad |
| `Core.Domain/Core/Contracts/IMemorandumRepository.cs` | Modificado — agrega `GetPaginatedAsync` |
| `Core.Domain/Core/Contracts/ISolicitudProtectorRepository.cs` | Modificado — agrega `GetPendienteMemoCompleteAsync` |
| `Core.Domain/Core/Entities/DbRows/MemorandumResponseDbRow.cs` | Nuevo — DbRow de resultado del SP |
| `Core.Domain/Parametrizaciones/Enums/TipoEntidadEnum.cs` | Modificado — agrega `Memorandum = 10` |
| `Core.Infraestructure/.../Core/MemorandumRepository.cs` | Modificado — implementa `GetPaginatedAsync` |
| `Core.Infraestructure/.../Core/SolicitudProtectorRepository.cs` | Modificado — implementa `GetPendienteMemoCompleteAsync` |
| `Core.Presentation/.../v1/MemorandumsController.cs` | Modificado — agrega endpoint GET paginado |
| `Core.Presentation/.../v1/ProtectoresController.cs` | Modificado — agrega endpoint solicitudes pendientes |
| `Core.Presentation/Api/Extensions/IOCCoreSchema.cs` | Modificado — registra 2 handlers nuevos |
| `Core.Presentation/Api/Extensions/IOCParametrizacionesSchema.cs` | Modificado — registra handler de estados por tipo entidad |
| `Core.Presentation/.../EstadosConfiguracionesController.cs` | Nuevo — controller para estados configuraciones |

---

## 3. Observaciones

### 3.1 [CRÍTICO] NullReferenceException en `MemorandumsController.GetMemorandums`

**Archivo:** `Core.Presentation/Api/Core/Controllers/v1/MemorandumsController.cs`

En el controller, el parámetro `parameters` de tipo `MemorandumGetRequestDto` tiene valor por defecto `null`:

```csharp
public async Task<ApiResult<PaginatedResult<MemorandumResponseDto>>> GetMemorandums(
    [FromQuery] int page = 1,
    [FromQuery] int perPage = 20,
    [FromQuery] MemorandumGetRequestDto parameters = null  // ← puede ser null
    )
{
    return await _dispatcher.Send(new MemorandumGetPaginatedQuery
    {
        Page = page,
        PerPage = perPage,
        Parameters = parameters  // ← sobreescribe el new() del record con null
    });
}
```

Cuando el cliente llama a `GET /memorandums` sin query params, `parameters` es `null`. Al asignarlo al record `MemorandumGetPaginatedQuery`, se sobreescribe su inicializador por defecto (`= new()`), dejando `Parameters = null`. El handler luego accede a `request.Parameters.NumeroMemorandum` sin null-check, lanzando `NullReferenceException`.

**Corrección sugerida:**

```csharp
Parameters = parameters ?? new MemorandumGetRequestDto()
```

---

### 3.2 [CRÍTICO] Bug de routing absoluto en `EstadosConfiguracionesController`

**Archivo:** `Core.Presentation/Api/Parametrizaciones/Controllers/v1/EstadosConfiguracionesController.cs`

```csharp
[HttpGet("/TipoEntidad/{tipoEntidad}")]  // ← la barra inicial hace la ruta ABSOLUTA
```

En ASP.NET Core, una ruta que comienza con `/` es absoluta: ignora completamente el prefijo del controller y del `ApiVersion`. El endpoint quedará expuesto como `GET /TipoEntidad/{tipoEntidad}` en lugar de la ruta convencional del proyecto (ej: `GET /api/v1/EstadosConfiguraciones/TipoEntidad/{tipoEntidad}`).

**Corrección sugerida:**

```csharp
[HttpGet("TipoEntidad/{tipoEntidad}")]  // sin barra inicial
```

---

### 3.3 [CRÍTICO] Filtro de fecha del historial es por rango del memo, no por fecha de sesión

**Archivo SP:** `Core.Memorandums_ByFilters_Select`  
**Archivo:** `Core.Application/.../Queries/MemorandumPaginatedGetQuery.cs`

La US define como filtros del historial: **"Fecha sesión desde / hasta"**. El DTO de request los llama `FechaSesionDesde` y `FechaSesionHasta`, lo que coincide con lo esperado. Sin embargo, el SP filtra sobre columnas distintas:

```sql
-- SP filtra por FechaSolicitudDesde/Hasta (el *rango* del memo)
AND (@pFechaSolicitudDesde IS NULL OR memo.FechaSolicitudDesde >= @pFechaSolicitudDesde)
AND (@pFechaSolicitudHasta IS NULL OR memo.FechaSolicitudHasta <= @pFechaSolicitudHasta)
-- ← debería filtrar por memo.FechaSesion
```

`FechaSolicitudDesde/Hasta` es el rango de solicitudes incluidas en el memo (ej: "27/12/25 a 04/01/26"), no la fecha en que se realizará la sesión de consejo (`FechaSesion`). Al aplicar el filtro, el usuario buscaría "memos con sesión entre X e Y" pero el sistema filtraría por "memos cuyo rango de solicitudes caiga entre X e Y", retornando resultados incorrectos.

**Corrección sugerida en el SP:**

```sql
AND (@pFechaSesionDesde IS NULL OR memo.FechaSesion >= @pFechaSesionDesde)
AND (@pFechaSesionHasta IS NULL OR memo.FechaSesion <= @pFechaSesionHasta)
```

Y renombrar los parámetros del SP y del repositorio consistentemente a `FechaSesion`.

---

### 3.4 [IMPORTANTE] `SolicitudProtectorPendienteMemoGetQuery` rompe el patrón UnitOfWork

**Archivo:** `Core.Application/.../Queries/SolicitudProtectorPendienteMemoGetQuery.cs`

La query y su handler no siguen el patrón establecido en el proyecto. Todas las demás queries extienden de `ReadUnitOfWork<T>` (record) y `HandlerUnitOfWork<TQuery, TResponse>` (handler), accediendo a datos a través de `IUnitOfWork`. Esta nueva query inyecta `ISolicitudProtectorRepository` directamente, bypasseando el UoW:

```csharp
// Patrón esperado (como MemorandumGetPaginatedQuery):
public sealed record SolicitudProtectorPendienteMemoGetQuery
    : ReadUnitOfWork<IEnumerable<SolicitudProtectorCompletaDto>>,
      IRequest<IEnumerable<SolicitudProtectorCompletaDto>>;

public sealed class SolicitudProtectorPendienteMemoGetQueryHandler
    : HandlerUnitOfWork<SolicitudProtectorPendienteMemoGetQuery, IEnumerable<SolicitudProtectorCompletaDto>>,
      IRequestHandler<...>
{
    public override async Task<...> Handle(...)
    {
        var uow = (IUnitOfWork)_unitOfWork;
        var entities = await uow.Core.SolicitudesProtectores.GetPendienteMemoCompleteAsync(...);
        ...
    }
}

// Implementación actual (incorrecta):
public sealed class SolicitudProtectorPendienteMemoGetQueryHandler : IRequestHandler<...>
{
    private readonly ISolicitudProtectorRepository _repository;  // inyectado directo
    ...
    var entities = await _repository.GetPendienteMemoCompleteAsync(...);
}
```

Esto además rompe la pipeline de `UnitOfWorkBehavior` (no aplica el comportamiento de UoW para reads).

---

### 3.5 [IMPORTANTE] Strings no-nullable en DTOs con nullable reference types activo

**Archivos:** `MemorandumResponseDto.cs`, `MemorandumResponseDbRow.cs`

Los campos de tipo `string` en los DTOs y DbRow no están marcados como `string?`:

```csharp
// MemorandumResponseDto
public string NumeroMemorandumTexto { get; set; }   // ← debería ser string?
public string RangoSolicitudTexto { get; set; }     // ← debería ser string?
public string EstadoDescripcion { get; set; }       // ← debería ser string?

// MemorandumResponseDbRow
public string EstadoDescripcion { get; set; }       // ← debería ser string?
```

En el mapping de `RangoSolicitudTexto`, si ambas fechas son null, el resultado será `" a "` (string con solo " a "), que podría no ser el comportamiento deseado para el UI.

---

### 3.6 [IMPORTANTE] Parsing silencioso de `NumeroMemorandum` sin feedback de error

**Archivo:** `Core.Application/.../Queries/MemorandumPaginatedGetQuery.cs`

El parsing del campo `NumeroMemorandum` (formato `"12/2026"`) falla silenciosamente sin notificar al cliente:

```csharp
var parts = request.Parameters.NumeroMemorandum
    .Split('/', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);

if (parts.Length >= 1 && int.TryParse(parts[0], out var parsedNumero))
    numeroMemorandum = parsedNumero;

if (parts.Length >= 2 && int.TryParse(parts[1], out var parsedAnio))
    anioMemorandum = parsedAnio;
```

Si el usuario ingresa `"abc/2026"`, `numeroMemorandum` quedará `null` pero `anioMemorandum = 2026` — filtrando por año igual a 2026 en lugar de retornar error o "no encontrado". Si ingresa `"12/abc"`, filtrará solo por número 12 ignorando el año. Debería validarse el formato con `BadRequest` si no es parseable como `int/int`.

---

### 3.7 [MENOR] Indentación inconsistente en `MemorandumGetRequestDto.cs`

**Archivo:** `Core.Application/.../Requests/MemorandumGetRequestDto.cs`

La propiedad `IdEstado` tiene un espacio extra de indentación:

```csharp
public DateTime? FechaSesionDesde { get; set; }
public DateTime? FechaSesionHasta { get; set; }
public string? NumeroMemorandum { get; set; }
 public int? IdEstado { get; set; }   // ← 5 espacios en lugar de 8
```

---

### 3.8 [MENOR] Confusión de nombres en la firma de `GetPaginatedAsync`

**Archivo:** `Core.Domain/Core/Contracts/IMemorandumRepository.cs` y su implementación

El método declara parámetros llamados `fechaSolicitudDesde/Hasta` pero recibe valores de `request.Parameters.FechaSesionDesde/Hasta`. Los nombres son semánticamente distintos (rango de solicitudes vs fecha de sesión), lo que agrava el bug 3.3 y dificulta el rastreo:

```csharp
// Interfaz:
Task<(...)> GetPaginatedAsync(
    ...
    DateTime? fechaSolicitudDesde = null,   // ← nombre confuso
    DateTime? fechaSolicitudHasta = null,
    ...);

// Handler:
var (data, totalData) = await uow.Core.Memorandum.GetPaginatedAsync(
    page,
    perPage,
    request.Parameters.FechaSesionDesde,   // ← nombre diferente
    request.Parameters.FechaSesionHasta,
    ...);
```

---

### 3.9 [MENOR] `null ?? (object)DBNull.Value` redundante

**Archivo:** `Core.Infraestructure/.../Core/SolicitudProtectorRepository.cs`

```csharp
new("CodigoEstado", codigoEstado ?? (object)DBNull.Value, ParamDirectionEnum.Input)
```

El helper `DbParameterModel` del proyecto (y Dapper subyacente) ya trata `null` como `DBNull.Value` automáticamente para parámetros SQL. El cast explícito es redundante y puede confundir. Basta con:

```csharp
new("CodigoEstado", codigoEstado, ParamDirectionEnum.Input)
```

---

### 3.10 [ALCANCE] PR cubre solo las queries de lectura

Según la US, el backend debe incluir:

| Funcionalidad | Estado en este PR |
|---------------|-------------------|
| Historial paginado de memos (GET con filtros) | ✅ Implementado |
| Solicitudes pendientes de memo (GET) | ✅ Implementado |
| Estados por tipo entidad (GET, para combo de filtros) | ✅ Implementado |
| Cards de resumen (total pendientes, por tipo + montos) | ❌ Pendiente |
| Generar memorándum (POST: crear + asignar solicitudes + PDF) | ❌ Pendiente |
| Actualizar memorándum (POST: agregar solicitudes + regenerar PDF) | ❌ Pendiente |
| Configurar fecha de sesión (POST/PUT) | ❌ Pendiente |
| Ver detalle de memo | ❌ Pendiente |
| Descargar PDF | ❌ Pendiente |
| Auditoría (log de generación y actualización) | ❌ Pendiente |

Esto es esperado si el PR es una entrega incremental. Se recomienda confirmar en la US que este alcance parcial es intencional y documentar los próximos PRs.

---

## 4. Resumen de hallazgos

| # | Criticidad | Archivo | Descripción |
|---|-----------|---------|-------------|
| 3.1 | 🔴 Crítico | `MemorandumsController.cs` | NullReferenceException cuando se llama sin query params |
| 3.2 | 🔴 Crítico | `EstadosConfiguracionesController.cs` | Ruta absoluta (`/TipoEntidad/...`) ignora prefijo del controller |
| 3.3 | 🔴 Crítico | SP `Memorandums_ByFilters_Select` | Filtro de fechas es por rango del memo, no por fecha de sesión |
| 3.4 | 🟠 Importante | `SolicitudProtectorPendienteMemoGetQuery.cs` | No usa patrón `ReadUnitOfWork` / `HandlerUnitOfWork` |
| 3.5 | 🟠 Importante | `MemorandumResponseDto.cs`, `MemorandumResponseDbRow.cs` | Strings no-nullable + RangoSolicitudTexto con " a " cuando fechas son null |
| 3.6 | 🟠 Importante | `MemorandumPaginatedGetQuery.cs` | Parsing silencioso de NumeroMemorandum sin validación ni feedback |
| 3.7 | 🟡 Menor | `MemorandumGetRequestDto.cs` | Indentación inconsistente en propiedad `IdEstado` |
| 3.8 | 🟡 Menor | `IMemorandumRepository.cs` | Nombres de parámetros `fechaSolicitud` vs `fechaSesion` confuso |
| 3.9 | 🟡 Menor | `SolicitudProtectorRepository.cs` | Cast redundante `null ?? (object)DBNull.Value` |
| 3.10 | ℹ️ Alcance | — | PR parcial: solo lectura, escritura/pdf/cards pendientes |

---

## 5. Checklist de aprobación

- [ ] 3.1 — Corregir null de `Parameters` en controller
- [ ] 3.2 — Quitar `/` inicial en ruta del `EstadosConfiguracionesController`
- [ ] 3.3 — Corregir SP para filtrar por `FechaSesion` o alinear nomenclatura con lo que realmente se filtra
- [ ] 3.4 — Refactorizar `SolicitudProtectorPendienteMemoGetQuery` para seguir patrón UoW
- [ ] 3.5 — Marcar strings como `string?` en DTOs y DbRows
- [ ] 3.6 — Agregar validación de formato para `NumeroMemorandum` o retornar 400 con mensaje claro
