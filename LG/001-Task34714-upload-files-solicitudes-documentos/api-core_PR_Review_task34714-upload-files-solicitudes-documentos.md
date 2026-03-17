# PR Review — feature/task34714-upload-documento-solicitudes vs release/3.0
## Task 34714 — Upload files: Documentos de Solicitudes

**Branch:** `feature/task34714-upload-documento-solicitudes`  
**Base:** `release/3.0`  
**Commit:** `b807411`  
**Fecha de revisión:** 17/03/2026  
**Revisor:** GitHub Copilot (análisis automatizado)

---

## 1. Resumen ejecutivo

Este PR implementa el **endpoint de creación de `DocumentosSolicitud`**, permitiendo asociar un tipo de documento a una solicitud existente identificando el tipo por ID o por nombre exacto. Se incluye también la refactorización del query de lectura para usar AutoMapper, el nuevo método de repositorio `GetByNombreAsync`, el SP correspondiente y una cobertura de tests unitarios sólida.

| Categoría | Cantidad |
|-----------|----------|
| Archivos modificados / creados | 11 |
| Líneas agregadas | +846 / -18 |
| Tests agregados | 18 (11 command + 4 query + 3 repository) |
| Bugs críticos | 1 |
| Issues importantes | 2 |
| Issues menores | 3 |

---

## 2. Archivos modificados

| Archivo | Tipo de cambio |
|---------|---------------|
| `Core.Application/.../Commands/DocumentosSolicitudCreateCommand.cs` | Nuevo — command + handler completo con flujo de validación |
| `Core.Application/.../Mappers/CoreMappingProfile.cs` | Modificado — agrega `CreateMap<DocumentoSolicitudDbRow, DocumentoSolicitudDto>()` |
| `Core.Application/.../Requests/DocumentosSolicitudCreateRequestDto.cs` | Nuevo — DTO de entrada con `IdDocumentoTipo` o `DocumentoTipoNombre` |
| `Core.Application/.../Queries/DocumentosSolicitudBySolicitudGetQuery.cs` | Modificado — reemplaza mapeo manual por AutoMapper |
| `Core.Domain/Parametrizaciones/Contracts/IDocumentoTipoRepository.cs` | Modificado — agrega `GetByNombreAsync(string nombre)` |
| `Core.Infraestructure/.../Parametrizaciones/DocumentoTipoRepository.cs` | Modificado — implementa `GetByNombreAsync` usando SP |
| `Core.Presentation/.../v1/DocumentosSolicitudesController.cs` | Modificado — agrega `POST empresas/{idEmpresa}/solicitudes/{idSolicitud}/documentos-solicitud` |
| `Core.Presentation/Api/Extensions/IOCCoreSchema.cs` | Modificado — registra `DocumentosSolicitudCreateCommandHandler` |
| `Core.UnitTesting/.../Commands/DocumentosSolicitudCreateCommandHandlerTests.cs` | Nuevo — 11 test cases para el command handler |
| `Core.UnitTesting/.../Queries/DocumentosSolicitudBySolicitudGetQueryHandlerTests.cs` | Nuevo — 4 test cases para el query handler refactorizado |
| `Core.UnitTesting/.../Parametrizaciones/DocumentoTipoRepositoryTests.cs` | Modificado — 3 test cases para `GetByNombreAsync` |

---

## 3. Flujo del comando (positivo)

```
POST /empresas/{idEmpresa}/solicitudes/{idSolicitud}/documentos-solicitud
{
  "idDocumentoTipo": 5          // ← OR
  "documentoTipoNombre": "CartaBanco"
}
```

1. Valida que se recibió al menos un identificador (400 si ninguno)
2. Valida que la solicitud existe (404)
3. Valida que la solicitud pertenece a la empresa del route (404)
4. Resuelve el tipo de documento: por ID → `GetByIdAsync`, por nombre → `GetByNombreAsync` SP (404 si no trovado)
5. Valida duplicado: no puede existir otro documento del mismo tipo en la misma solicitud (409)
6. Inserta en `Core.DocumentosSolicitud`
7. Lee el registro completo y retorna el DTO (201 Created)

---

## 4. Observaciones

### 4.1 [CRÍTICO] `GetByIdAsync` no filtra por `Activo` — tipo inactivo puede asociarse

**Archivo:** `Core.Application/.../Commands/DocumentosSolicitudCreateCommand.cs`

Cuando el cliente envía `IdDocumentoTipo`, el handler usa el método genérico base `GetByIdAsync`, que **no filtra** por `Activo`. El SP `DocumentosTipos_ByNombre_Select` sí incluye `AND dt.Activo = 1`, por lo que la resolución por nombre está protegida, pero la resolución por ID no lo está.

```csharp
// Resolución por ID → usa GetByIdAsync base → no filtra Activo
var documentoTipo = request.Dto.IdDocumentoTipo.HasValue
    ? await docTiposRepo.GetByIdAsync(request.Dto.IdDocumentoTipo.Value)   // ← sin filtro Activo
    : await docTiposRepo.GetByNombreAsync(request.Dto.DocumentoTipoNombre!); // ← SP filtra Activo = 1
```

Si un tipo de documento fue desactivado (`Activo = 0`), puede seguir siendo referenciado por ID, creando una asociación con un tipo inactivo sin ninguna advertencia.

**Corrección sugerida:** Agregar validación en el handler luego de resolver el tipo:

```csharp
if (documentoTipo != null && documentoTipo.Activo != true)
{
    throw new CustomException(
        HttpStatusCode.NotFound,
        $"El tipo de documento con id {request.Dto.IdDocumentoTipo} no está activo.");
}
```

O alternativamente, crear `GetByIdActivoAsync` que filtre por activo en el repositorio.

---

### 4.2 [IMPORTANTE] Seed de `DocumentosRequeridos` no es idempotente

**Archivo:** Script SQL proporcionado

El SEED inserta en `DocumentosTipos` con guarda `IF NOT EXISTS`, pero el INSERT en `DocumentosRequeridos` **no tiene guard de idempotencia**:

```sql
-- DocumentosTipo: CartaBanco (idempotente) ✓
IF NOT EXISTS (SELECT 1 FROM [Parametrizaciones].[DocumentosTipos] WHERE Nombre = N'CartaBanco')
    INSERT INTO [Parametrizaciones].[DocumentosTipos] ...;

-- DocumentosRequeridos: SIN guard ✗
INSERT INTO [Parametrizaciones].[DocumentosRequeridos]
    (IdFlujo, IdDocumentoTipo, EsObligatorio, ...)
VALUES (...);
-- Si este script se ejecuta dos veces, crea duplicados en DocumentosRequeridos
```

Si el script se ejecuta más de una vez (error humano, re-deploy, rollback parcial), se generarán filas duplicadas en `DocumentosRequeridos` que modificarán el comportamiento del flujo `SOLICITUD_GARANTIAS`.

**Corrección sugerida:**

```sql
IF NOT EXISTS (
    SELECT 1 FROM [Parametrizaciones].[DocumentosRequeridos]
    WHERE IdFlujo = @IdFlujoSolicitud
      AND IdDocumentoTipo = @IdDocTipo
      AND FechaBaja IS NULL
)
    INSERT INTO [Parametrizaciones].[DocumentosRequeridos]
        (IdFlujo, IdDocumentoTipo, EsObligatorio, TieneVencimiento, ...)
    VALUES (...);
```

---

### 4.3 [IMPORTANTE] Comportamiento silencioso cuando se envían ambos identificadores

**Archivo:** `Core.Application/.../Requests/DocumentosSolicitudCreateRequestDto.cs`  
**Archivo:** `Core.Application/.../Commands/DocumentosSolicitudCreateCommand.cs`

El DTO documenta ambos campos como "uso excluyente" pero no hay validación que lo enforce ni en el DTO ni en el handler. Si el cliente envía ambos, `IdDocumentoTipo` gana silenciosamente y `DocumentoTipoNombre` es ignorado, sin retornar error ni advertencia:

```csharp
// Si ambos son non-null: IdDocumentoTipo tiene prioridad silenciosamente
var documentoTipo = request.Dto.IdDocumentoTipo.HasValue
    ? await docTiposRepo.GetByIdAsync(request.Dto.IdDocumentoTipo.Value)
    : await docTiposRepo.GetByNombreAsync(request.Dto.DocumentoTipoNombre!);
```

Esto puede causar comportamientos inesperados: cliente envía `IdDocumentoTipo = 5, DocumentoTipoNombre = "CartaBanco"` pensando que son coherentes, pero si el id 5 no corresponde al nombre, se inserta el tipo incorrecto sin error.

**Corrección sugerida:**

```csharp
// En el handler, a continuación de la validación del paso 1:
if (request.Dto.IdDocumentoTipo.HasValue && !string.IsNullOrWhiteSpace(request.Dto.DocumentoTipoNombre))
{
    throw new CustomException(
        HttpStatusCode.BadRequest,
        "Especifique IdDocumentoTipo o DocumentoTipoNombre, no ambos.");
}
```

---

### 4.4 [MENOR] Posible null reference en mensaje del 409 Conflict

**Archivo:** `Core.Application/.../Commands/DocumentosSolicitudCreateCommand.cs`

```csharp
throw new CustomException(
    HttpStatusCode.Conflict,
    $"Ya existe un documento de tipo '{documentoTipo.Nombre}' en la solicitud {request.IdSolicitud}.");
```

`DocumentoTipo.Nombre` puede ser `null` (es un string no anotado como non-nullable en la entidad). Generar el mensaje sin null-check produciría `"Ya existe un documento de tipo '' en la solicitud..."`. Menor impacto en runtime, pero el mensaje sería confuso.

---

### 4.5 [MENOR] Test faltante: tipo de documento inactivo resuelto por ID

**Archivo:** `Core.UnitTesting/.../Commands/DocumentosSolicitudCreateCommandHandlerTests.cs`

La cobertura de tests es completa para los casos del happy path y errores documentados, pero falta el caso que corresponde al bug 4.1:

```
Escenario no cubierto:
  - GetByIdAsync retorna un DocumentoTipo con Activo = false
  - Resultado esperado (tras aplicar la corrección del 4.1): 404 NotFound
```

También falta cobertura del escenario donde ambos campos son enviados simultáneamente (relacionado con 4.3).

---

### 4.6 [MENOR] Refactoring de `DocumentosSolicitudBySolicitudGetQuery` introduce dependencia de IMapper sin actualizar IOC

**Archivo:** `Core.Application/.../Queries/DocumentosSolicitudBySolicitudGetQuery.cs`  
**Archivo:** `Core.Presentation/Api/Extensions/IOCCoreSchema.cs`

El handler fue modificado para recibir `IMapper` por constructor:

```csharp
public DocumentosSolicitudBySolicitudGetQueryHandler(IMapper mapper)
{
    _mapper = mapper;
}
```

`IMapper` es registrado globalmente por el setup de AutoMapper (no requiere registro explícito en IOC), por lo que el DI container lo resolverá correctamente. **No es un error**, pero si el handler era previamente sin constructor (y el contenedor lo instanciaba con constructor vacío), puede haber tests de integración que fallen si mockeaban el handler directamente. Verificar pipeline de CI.

---

## 5. Puntos positivos

- **Capa de validación completa**: El command valida en orden lógico (parámetros → solicitud → empresa → tipo → duplicado) antes de cualquier escritura. Consistente con el patrón del proyecto.
- **Doble modalidad de resolución del tipo**: soporte por ID y por nombre es útil para flujos automatizados (n8n, integrations) que conocen el nombre pero no el ID.
- **SP bien escrito**: `DocumentosTipos_ByNombre_Select` sigue el estándar del proyecto (`CREATE OR ALTER`, `SET NOCOUNT ON`, `TRY/CATCH`, `NOLOCK` condicional por `@@TRANCOUNT`, `GRANT` a ambos usuarios).
- **Tests de alta calidad**: 18 tests unitarios con buena granularidad — cada escenario de error tiene su propio test, y se verifica tanto el resultado como las llamadas al repositorio (via `Verify`).
- **Refactoring del query handler**: La eliminación del mapeo manual en `DocumentosSolicitudBySolicitudGetQuery` reduce duplicación y es consistente con el resto de handlers del proyecto.
- **SEED idempotente para `DocumentosTipos`**: La guarda `IF NOT EXISTS` en la tabla de tipos está correctamente implementada.

---

## 6. Resumen de hallazgos

| # | Criticidad | Componente | Descripción |
|---|-----------|-----------|-------------|
| 4.1 | 🔴 Crítico | `DocumentosSolicitudCreateCommand` | `GetByIdAsync` no filtra `Activo = 1` — tipo inactivo puede asociarse sin error |
| 4.2 | 🟠 Importante | Script SEED SQL | INSERT en `DocumentosRequeridos` no es idempotente — duplicados si se re-ejecuta |
| 4.3 | 🟠 Importante | Handler + DTO | Comportamiento silencioso cuando se envían `IdDocumentoTipo` Y `DocumentoTipoNombre` juntos |
| 4.4 | 🟡 Menor | Handler | `documentoTipo.Nombre` puede ser null en mensaje del Conflict 409 |
| 4.5 | 🟡 Menor | Tests | Falta test de tipo inactivo resuelto por ID y de ambos campos simultáneos |
| 4.6 | 🟡 Menor | Query handler + IOC | Constructor nuevo con `IMapper` — sin impacto en runtime pero verificar CI |

---

## 7. Checklist de aprobación

- [ ] 4.1 — Agregar validación de `Activo` tras resolver documentoTipo por ID
- [ ] 4.2 — Agregar guard `IF NOT EXISTS` al INSERT en `DocumentosRequeridos` del seed
- [ ] 4.3 — Agregar validación que rechace `BadRequest` cuando ambos identificadores son enviados
- [ ] 4.4 — Agregar null-check a `documentoTipo.Nombre` en el mensaje del Conflict
- [ ] 4.5 — Agregar test cases para tipo inactivo y ambos campos simultáneos (previo a 4.1 y 4.3)
