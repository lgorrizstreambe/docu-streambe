# PR Review — `feature/33346-COR-GSP-CSA` → `release/3.0`

**Fecha de revisión:** 2026-03-20
**Autor:** Federico Marcó (fmarco@garantizar.com.ar)
**PR:** #1363 · Merged 2026-03-19
**Commit de merge:** `367b0da`

**Scope:** `Api.Core` · 20 archivos · +560 / −6 líneas
**User Story:** US33346 · COR-GSP-CSA – Generación automática de Memorándum

---

## 1. Archivos del PR

### 1.1 Base de datos (SP provisto en la US)

| Artefacto | Descripción |
|---|---|
| `[Core].[Memorandums_ByIdCompleto_Select]` | SP que retorna el memorándum con todas sus solicitudes protectoras asociadas (JOINs a `Empresas`, `Fondos`, `TipoOperaciones`, `EstadosConfiguraciones`). Columnas alias prefijadas con `Memorandum*` para mapeo directo a `MemorandumCompletoDbRow`. Maneja dos ramas `@@TRANCOUNT = 0 / != 0` con y sin hints `WITH (NOLOCK)` |

### 1.2 Domain

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.Domain/Core/Contracts/IMemorandumRepository.cs` | Interface (patch) | Agrega `Task<List<MemorandumCompletoDbRow>> GetByIdCompletoAsync(int idMemorandum)` |
| `Core.Domain/Core/Entities/MemorandumCompleto.cs` | Entidad | Hereda de `Memorandum` + añade `MemorandumCodigoEstado` y `List<MemorandumSolicitudDetalle>` |
| `Core.Domain/Core/Entities/DbRows/MemorandumCompletoDbRow.cs` | DbRow | Mapeo plano del SP + método estático `AgruparYMapear()` que agrupa las N filas en un `MemorandumCompleto` |
| `Core.Domain/Maestros/Enums/EstadoEnum.cs` | Enum (patch) | Agrega `RevisadoConsejo` con código `"MEM_REV_CONSEJO"` y mapea `"MEM_PEND_CONSEJO"` a `PendienteConsejo` en `FromCodigo()` |
| `Core.Domain/Parametrizaciones/Contracts/IPlantillaRepository.cs` | Interface | Define `GetVigenteByTipoPlantillaAsync(int idTipoPlantilla)` |
| `Core.Domain/Parametrizaciones/Entities/Plantilla.cs` | Entidad | ORM de `[Parametrizaciones].[Plantillas]`: `IdPlantilla`, `IdTipoPlantilla`, `Version`, `Nombre`, `ContenidoFijo`, `JsonSchemaVariables`, `EsVigente` + auditoría |
| `Core.Domain/Parametrizaciones/Enums/TipoPlantillaEnum.cs` | Enum | `ActaConsejo = 1`, `Memorandum = 2` con `GetNombre()` |
| `Core.Domain/Parametrizaciones/Enums/TipoEntidadEnum.cs` | Enum (patch) | Agrega valor `Memorandum` para upload de archivos |
| `Core.Domain/Parametrizaciones/Contracts/IParametrizacionesSchemaContext.cs` | Interface (patch) | Expone `IPlantillaRepository Plantillas` |

### 1.3 Application

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.Application/Modules/Core/Commands/MemorandumGenerarPdfCommand.cs` | Command (207 líneas) | Lógica completa de generación, guardado y descarga del PDF del memorándum (ver §3) |
| `Core.Application/Modules/Core/Dtos/Responses/ArchivoDto.cs` | DTO | `byte[] Content`, `string ContentType`, `string FileName` |
| `Core.Application/Modules/Core/Dtos/Mappers/CoreMappingProfile.cs` | Mapper (patch) | `CreateMap<ArchivoDownloadResponse, ArchivoDto>()` |

### 1.4 Infraestructura

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.Infraestructure/Persistence/Repositories/Core/MemorandumRepository.cs` | Repo (patch) | Implementa `UpdateAsync()` (antes `NotImplementedException`) + `GetByIdCompletoAsync()` llamando a `Core.Memorandums_ByIdCompleto_Select` |
| `Core.Infraestructure/Persistence/Repositories/Parametrizaciones/PlantillaRepository.cs` | Repo (nuevo) | Implementa solo `GetVigenteByTipoPlantillaAsync()` → SP `Parametrizaciones.Plantillas_UltimaVigentePorTipoPlantilla_Select`. Resto de métodos: `NotImplementedException` |
| `Core.Infraestructure/Persistence/Schemas/ParametrizacionesSchemaContext.cs` | Schema (patch) | Registra `IPlantillaRepository Plantillas` |

### 1.5 Presentación

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.Presentation/Api/Core/Controllers/v1/MemorandumsController.cs` | Controller (patch) | Agrega `POST /{idMemorandum}/pdf` → `MemorandumGenerarPdfCommand` |
| `Core.Presentation/Api/Extensions/IOCCoreSchema.cs` | DI (patch) | Sin detalles visibles en el diff (registro de dependencias) |
| `Core.Presentation/Api/Extensions/IOCParametrizacionesSchema.cs` | DI (patch) | Registro de `PlantillaRepository` en el contenedor |

### 1.6 Tests

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.UnitTesting/Infrastructure/Persistence/Repositories/Core/MemorandumRepositoryTests.cs` | Unit (patch) | Actualización de tests existentes (3 líneas modificadas) |

---

## 2. Modelo de datos

```
[Core].[Memorandums]  (tabla existente, ahora con Update implementado)
  IdMemorandum          PK
  NumeroMemorandum      INT
  Anio                  INT
  FechaSesion           DATETIME?
  FechaSolicitudDesde   DATETIME?
  FechaSolicitudHasta   DATETIME?
  CantidadSolicitudes   INT
  ImporteTotalEnPesos   DECIMAL
  IdEstadoConfiguracion FK → Parametrizaciones.EstadosConfiguraciones
  IdArchivo             FK → Archivos  (nullable — se llena al guardar el PDF)
  IdPlantilla           FK → Parametrizaciones.Plantillas  (nullable — registra qué plantilla se usó)

[Core].[Memorandums_SolicitudesProtectores]  (tabla existente)
  IdMemorandum          FK → Core.Memorandums
  IdSolicitudProtector  FK → Core.SolicitudesProtectores
  FechaBaja             DATETIME?

[Parametrizaciones].[Plantillas]  (tabla existente — creada en script 038)
  IdPlantilla           PK
  IdTipoPlantilla       FK → Parametrizaciones.TiposPlantillas  (2 = Memorandum)
  Version               INT
  Nombre                NVARCHAR
  ContenidoFijo         NVARCHAR(MAX)  — HTML con placeholders {Campo}
  EsVigente             BIT
```

---

## 3. Flujo del endpoint `POST /{idMemorandum}/pdf`

```
POST /v1/memorandums/{idMemorandum}/pdf
  → MemorandumGenerarPdfCommandHandler
       ├─ MemorandumRepository.GetByIdCompletoAsync(idMemorandum)
       │     → SP Core.Memorandums_ByIdCompleto_Select
       │     → MemorandumCompletoDbRow.AgruparYMapear() → MemorandumCompleto
       │
       ├─ EstadoExtensions.FromCodigo(codigoEstado)
       │
       ├─ Si estado == RevisadoConsejo:
       │     ├─ Si ya tiene IdArchivo → ApiArchivosService.DownloadAsync()  (retorna PDF guardado)
       │     └─ Si no tiene IdArchivo → GenerarYGuardarPdfAsync()
       │           ├─ PlantillaRepository.GetVigenteByTipoPlantillaAsync(TipoPlantillaEnum.Memorandum)
       │           ├─ InterpolateContent(plantilla, memorandumCompleto)
       │           ├─ PdfGenerator.GenerateFromHtmlAsync(html)
       │           ├─ ApiArchivosService.UploadAsync(pdf, TipoEntidadEnum.Memorandum)
       │           └─ MemorandumRepository.UpdateAsync() → persiste IdArchivo + IdPlantilla
       │
       └─ Si otro estado:
             ├─ PlantillaRepository.GetVigenteByTipoPlantillaAsync(Memorandum)
             ├─ InterpolateContent()
             ├─ PdfGenerator.GenerateFromHtmlAsync()
             └─ Retorna PDF sin persistir (preview/borrador)
```

### Lógica de interpolación (`InterpolateContent`)

Placeholders reemplazados en `Plantilla.ContenidoFijo`:

| Placeholder | Fuente |
|---|---|
| `{NumeroMemorandum}` | `memorandum.NumeroMemorandum` |
| `{Anio}` | `memorandum.Anio` |
| `{FechaSesion}` | `memorandum.FechaSesion?.ToString("dd/MM/yyyy")` |
| `{FechaSolicitudDesde}` / `{FechaSolicitudHasta}` | Rango de fechas del memorándum |
| `{CantidadSolicitudes}` | `memorandum.CantidadSolicitudes` |
| `{ImporteTotalEnPesos}` | `memorandum.ImporteTotalEnPesos.ToString("N2", es-AR)` |
| `{AportesRows}` / `{ReimposicionesRows}` / `{RetirosRows}` | `BuildRows()` filtrando por `TipoOperacion` |

`BuildRows()` genera HTML `<tr>` con `WebUtility.HtmlEncode()` para: RazonSocial, ImporteSolicitud (formato `"C0"` es-AR), Fondo, Observaciones. Si no hay filas, inserta `<td colspan="4">Sin registros</td>`.

---

## 4. Análisis del SP `Memorandums_ByIdCompleto_Select`

```sql
-- Estructura del SELECT (simplificado):
FROM [Core].[Memorandums] memos WITH (NOLOCK)
INNER JOIN [Core].[Memorandums_SolicitudesProtectores] memosSoli WITH (NOLOCK)
    ON memosSoli.IdMemorandum = memos.IdMemorandum AND memosSoli.FechaBaja IS NULL
LEFT JOIN [Core].[SolicitudesProtectores] soliPro WITH (NOLOCK)
    ON soliPro.IdSolicitud = memosSoli.IdSolicitudProtector
    -- (FechaBaja comentado: incluye solicitudes dadas de baja)
LEFT JOIN [Maestros].[Empresas], [Maestros].[Fondos], [Maestros].[TipoOperaciones]
LEFT JOIN [Parametrizaciones].[EstadosConfiguraciones]
WHERE memos.FechaBaja IS NULL AND memos.IdMemorandum = @pIdMemorandum
```

---

## 5. Análisis técnico

### ✅ Fortalezas

| Aspecto | Detalle |
|---|---|
| **Lógica de estado clara** | La bifurcación `RevisadoConsejo` vs otros estados es explícita y correcta: el PDF se persiste solo cuando el consejo ya revisó; antes opera como borrador |
| **Idempotencia en descarga** | Si `RevisadoConsejo` + `IdArchivo != null` → descarga el PDF ya guardado sin regenerar. Evita generar múltiples archivos |
| **`BuildRows` seguro contra XSS** | `WebUtility.HtmlEncode()` en todos los campos de texto antes de inyectar en HTML |
| **Trazabilidad de plantilla** | Al persistir, guarda `IdPlantilla` en `Memorandums`. Permite saber exactamente qué versión de la plantilla generó ese PDF |
| **Registro metadata en upload** | `JsonSerializer.Serialize(new { TipoDocumento = "Memorandum", Version = plantilla.Version })` como metadata del archivo |
| **`AgruparYMapear` en DbRow** | Patrón correcto para SP que retorna N filas (1 por solicitud): la agrupación ocurre en el DbRow, no en el handler |

### ⚠️ Observaciones / Mejoras sugeridas

#### 1. Validación de nulidad antes de agrupar (bug potencial)
```csharp
// Línea actual en el handler:
var memorandumCompleto = MemorandumCompletoDbRow.AgruparYMapear(memorandum); // puede retornar null
if (memorandum == null)   // ← verifica la lista, no el resultado agrupado
    throw new CustomException(HttpStatusCode.NotFound, "Memorándum no encontrado.");
```
Si `memorandum` (la lista) es `null`, `AgruparYMapear` ya retorna `null`, pero se llama ANTES del check. Si la lista está vacía (memorándum inexistente), `AgruparYMapear` retorna `null` y luego `memorandumCompleto` es `null` pero el check `if (memorandum == null)` pasa porque la lista no es null — es vacía. El handler seguiría con `memorandumCompleto null` y lanzaría `NullReferenceException` en lugar del `CustomException 404`.

**Fix sugerido:**
```csharp
var rows = await uow.Core.Memorandum.GetByIdCompletoAsync(request.IdMemorandum);
var memorandumCompleto = MemorandumCompletoDbRow.AgruparYMapear(rows)
    ?? throw new CustomException(HttpStatusCode.NotFound, "Memorándum no encontrado.");
```

#### 2. `PlantillaRepository` con 4 `NotImplementedException`
Los métodos `CreateAsync`, `GetByIdAsync`, `UpdateAsync`, `DeleteAsync` y `GetAllAsync` lanzan `NotImplementedException`. No son un problema hoy, pero si otro handler invoca cualquiera de ellos explotará en producción. Recomendable marcarlos con `throw new NotSupportedException("No implementado para Plantillas.")` o agregar un comentario explícito.

#### 3. SP — `FechaBaja` comentada en `SolicitudesProtectores`
```sql
LEFT JOIN [Core].[SolicitudesProtectores] soliPro ON soliPro.IdSolicitud = memosSoli.IdSolicitudProtector
  --AND soliPro.FechaBaja IS NULL
```
La condición `FechaBaja IS NULL` está comentada. Esto incluye solicitudes dadas de baja en el memorándum. Si es intencional (para preservar el historial), debe documentarse con un comentario; si no, es un bug silencioso.

#### 4. SP — duplicación de código entre ramas `@@TRANCOUNT`
El SP tiene dos bloques `SELECT` idénticos (uno con `WITH (NOLOCK)` y otro sin). Podría simplificarse con un CTE o extrayendo la lógica, aunque esto es menor porque SQL Server no permite deduplicación simple aquí.

#### 5. Generación semanal automática — no implementada en este PR
La US describe generación automática semanal (martes a las 23:59). Este PR solo implementa la generación on-demand via `POST /{idMemorandum}/pdf`. El trigger automático (job, n8n, Azure Function, etc.) queda pendiente para otro PR/US. **No bloquea la aprobación** dado que la US prioriza la generación correcta del documento.

#### 6. Notificación automática (AC6) — no implementada
El AC6 dice "notificar al Consejo y al área de Socios Protectores con un enlace al documento". No hay llamada a `IApiMailService` ni a `IApiSMSService` en el handler. Similar al punto anterior, probablemente planificado para un PR posterior.

---

## 6. Criterios de aceptación — cobertura

| AC | Escenario | Estado |
|---|---|---|
| AC1 | Inclusión automática al confirmar solicitud | ⚠️ No visible en este PR (lógica preexistente o en otro PR) |
| AC2 | Generación semanal automática (martes 23:59) | ❌ No implementado — solo generación on-demand |
| AC3 | Formato del documento (logo, número, fondo, tablas por tipo) | ✅ Implementado vía `InterpolateContent` + `BuildRows` con plantilla HTML |
| AC5 | Guardado y trazabilidad (`IdArchivo`, `IdPlantilla`, usuario) | ✅ `GenerarYGuardarPdfAsync` persiste `IdArchivo` + `IdPlantilla` en `Memorandums.UpdateAsync` |
| AC6 | Notificación automática al Consejo y Socios | ❌ No implementado en este PR |

---

## 7. Resumen ejecutivo

| | |
|---|---|
| **Archivos** | 20 (+560 / −6) |
| **Autor** | Federico Marcó |
| **Aprobación** | ✅ Aprobar con observaciones |

**Observación crítica (requiere fix antes de próximo release):**
- Bug de orden de validación en el handler (#1 arriba): si el memorándum no existe, el código lanzará `NullReferenceException` en lugar del `CustomException 404` esperado.

**Observaciones menores (no bloquean el merge):**
- `FechaBaja IS NULL` comentada en SP — documentar intención (#3)
- `PlantillaRepository` con métodos `NotImplementedException` sin documentar (#2)
- Generación automática semanal y notificación pendientes de implementación (#5, #6)
