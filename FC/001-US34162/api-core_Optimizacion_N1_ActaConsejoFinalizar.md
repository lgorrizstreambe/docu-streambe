# Propuesta de Optimización: Eliminación de N+1 en `ActaConsejoFinalizarCommand`

**US:** 34162 — Guardado final de Acta del Consejo  
**Archivo:** `Core.Application/Modules/Core/Commands/ActaConsejoFinalizarCommand.cs`  
**Fecha:** 16/03/2026  

---

## 1. Problema Actual

El método `ProcesarSolicitudIndividual`, invocado en un `foreach` sobre cada solicitud del acta, ejecuta **4 queries a la base de datos por iteración**:

| # | Query | Descripción |
|---|-------|-------------|
| 1 | `Solicitudes.GetByIdAsync` | Carga la solicitud por ID |
| 2 | `Empresas.GetByIdAsync` | Carga la empresa de la solicitud |
| 3 | `EstadosConfiguraciones.GetByIdTipoEntidadYEstadoNombre` | Estado destino |
| 4 | `EstadosConfiguraciones.GetByIdAsync` | Estado origen |

**Para un acta con 30 solicitudes de 10 empresas distintas → ~120 queries de lectura**, antes de ejecutar las escrituras.

### Código problemático (extracto)

```csharp
foreach (var actaSolicitud in actaExistente.Solicitudes)
{
    await ProcesarSolicitudIndividual(uow, idUsuarioActual, actaSolicitud, cancellationToken);
}

private async Task ProcesarSolicitudIndividual(...)
{
    var solicitud   = await uow.Core.Solicitudes.GetByIdAsync(actaSolicitud.IdSolicitud);        // 1 query
    var empresa     = await uow.Maestros.Empresas.GetByIdAsync(solicitud.IdEmpresa);              // 1 query
    var estadoDest  = await uow.Parametrizaciones.EstadosConfiguraciones                          // 1 query
                               .GetByIdTipoEntidadYEstadoNombre(...);
    var estadoOrig  = await uow.Parametrizaciones.EstadosConfiguraciones                          // 1 query
                               .GetByIdAsync(solicitud.IdEstadoConfiguracion);
    // + orquestador y escrituras...
}
```

---

## 2. Solución Propuesta

Reestructurar en dos pasos separados:

1. **Pre-carga en batch** (antes del `foreach` de escritura)
2. **Ejecución de la transición** con los datos ya cargados en memoria

Con las APIs batch ya disponibles en los repositorios:

| Repositorio | Método batch disponible |
|-------------|------------------------|
| `ISolicitudRepository` | `GetByLoteIdAsync(List<int> idsSolicitudes)` → `List<Solicitud>` |
| `IEstadoConfiguracionRepository` | `GetByIdTipoEntidad(int idTipoEntidad)` → `IEnumerable<EstadoConfiguracionDbRow>` |
| `IEmpresaRepository` | Sin método batch — se mitiga con `Dictionary<int, Empresa>` como caché local |

### Queries resultantes

| Query | Descripción |
|-------|-------------|
| 1 | `GetByLoteIdAsync` — todas las solicitudes en un solo llamado |
| 2 | `GetByIdTipoEntidad(Solicitud)` — todos los estados de tipo Solicitud en un solo llamado |
| 3..N | `Empresas.GetByIdAsync` — máximo 1 query por empresa **distinta** (caché de diccionario) |

**Para 30 solicitudes de 10 empresas: de ~120 queries → 12 queries (3 batch + 10 empresas únicas).**

---

## 3. Código Refactorizado

### 3.1 `ProcesarTransicionSolicitudes` (reemplaza la versión actual)

```csharp
private async Task ProcesarTransicionSolicitudes(
    IUnitOfWork uow,
    int idUsuarioActual,
    Domain.Core.Entities.ActaConsejo actaExistente,
    CancellationToken cancellationToken)
{
    if (actaExistente.Solicitudes == null || !actaExistente.Solicitudes.Any())
        return;

    // ─── PASO 1: Validar TODAS antes de escribir ninguna ─────────────────────
    foreach (var actaSolicitud in actaExistente.Solicitudes)
    {
        if (string.IsNullOrWhiteSpace(actaSolicitud.EstadoDestino))
            throw new CustomException(
                HttpStatusCode.BadRequest,
                $"La solicitud {actaSolicitud.IdSolicitud} no tiene Estado Destino asignado. " +
                 "Asigne el resultado (Aprobado/Rechazado/Vuelve a aprobación) antes de finalizar el acta.");

        if (EstadoExtensions.FromCodigo(actaSolicitud.EstadoDestino) == null)
            throw new CustomException(
                HttpStatusCode.BadRequest,
                $"El código de Estado Destino '{actaSolicitud.EstadoDestino}' de la solicitud " +
                $"{actaSolicitud.IdSolicitud} no es válido.");
    }

    // ─── PASO 2: Pre-carga en batch ───────────────────────────────────────────

    // 2a. 1 query — todas las solicitudes del acta
    var idsSolicitudes = actaExistente.Solicitudes.Select(s => s.IdSolicitud).ToList();
    var solicitudesCargadas = await uow.Core.Solicitudes.GetByLoteIdAsync(idsSolicitudes);
    var solicitudesDict = solicitudesCargadas.ToDictionary(s => s.IdSolicitud);

    // 2b. 1 query — todos los estados activos de tipo Solicitud
    var estadoConfigsRows = await uow.Parametrizaciones.EstadosConfiguraciones
        .GetByIdTipoEntidad((int)TipoEntidadEnum.Solicitud);

    var estadoConfigsPorNombre = estadoConfigsRows
        .Where(e => e.FechaBaja == null && e.Estados_Nombre != null)
        .ToDictionary(e => e.Estados_Nombre!, e => e);

    var estadoConfigsPorId = estadoConfigsRows
        .Where(e => e.FechaBaja == null)
        .ToDictionary(e => e.IdEstadoConfiguracion, e => e);

    // 2c. Caché local de empresas (1 query por empresa distinta, máximo)
    var empresasCache = new Dictionary<int, Domain.Maestros.Entities.Empresa>();

    // ─── PASO 3: Ejecutar transiciones con datos pre-cargados ─────────────────
    foreach (var actaSolicitud in actaExistente.Solicitudes)
    {
        if (!solicitudesDict.TryGetValue(actaSolicitud.IdSolicitud, out var solicitud))
            throw new CustomException(
                HttpStatusCode.InternalServerError,
                $"No se encontró la solicitud con ID {actaSolicitud.IdSolicitud}.");

        if (!empresasCache.TryGetValue(solicitud.IdEmpresa, out var empresa))
        {
            empresa = await uow.Maestros.Empresas.GetByIdAsync(solicitud.IdEmpresa);
            if (empresa == null)
                throw new CustomException(
                    HttpStatusCode.InternalServerError,
                    $"No se encontró la empresa con ID {solicitud.IdEmpresa} asociada a la solicitud {solicitud.IdSolicitud}.");
            empresasCache[solicitud.IdEmpresa] = empresa;
        }

        var estadoDestinoNombre = EstadoExtensions.FromCodigo(actaSolicitud.EstadoDestino)!.GetNombre();

        if (!estadoConfigsPorNombre.ContainsKey(estadoDestinoNombre))
            throw new CustomException(
                HttpStatusCode.InternalServerError,
                $"No se encontró configuración de estado '{estadoDestinoNombre}' para Solicitud {actaSolicitud.IdSolicitud}.");

        var estadoOrigenNombre = estadoConfigsPorId.TryGetValue(solicitud.IdEstadoConfiguracion, out var origenRow)
            ? origenRow.Estados_Nombre ?? "Desconocido"
            : "Desconocido";

        await EjecutarTransicionSolicitud(
            uow, idUsuarioActual,
            actaSolicitud, solicitud, empresa,
            estadoDestinoNombre, estadoOrigenNombre,
            cancellationToken);
    }
}
```

### 3.2 `EjecutarTransicionSolicitud` (reemplaza a `ProcesarSolicitudIndividual`)

Responsabilidad única: armar el contexto y delegar al orquestador, luego aplicar las escrituras.

```csharp
private async Task EjecutarTransicionSolicitud(
    IUnitOfWork uow,
    int idUsuarioActual,
    Domain.Core.Entities.ActaConsejoSolicitud actaSolicitud,
    Domain.Core.Entities.Solicitud solicitud,
    Domain.Maestros.Entities.Empresa empresa,
    string estadoDestinoNombre,
    string estadoOrigenNombre,
    CancellationToken cancellationToken)
{
    var context = new TransicionContext
    {
        UnitOfWork = uow,
        IdEntidad = solicitud.IdSolicitud,
        IdTipoEntidad = (int)TipoEntidadEnum.Solicitud,
        IdEstadoConfiguracionOrigen = solicitud.IdEstadoConfiguracion,
        EstadoOrigenNombre = estadoOrigenNombre,
        IdTipoSocio = (TipoSocioEnum)empresa.IdTipoSocio,
        IdUsuario = idUsuarioActual,
        Solicitud = solicitud,
        Empresa = empresa,
        ConfiguracionAdicional = new Dictionary<string, object>
        {
            ["EstadoDestinoNombre"] = estadoDestinoNombre
        },
        MotivoEstado = $"Acta de Consejo finalizada. Estado destino: {estadoDestinoNombre}",
        OrigenEvento = "Sistema - Finalización Acta"
    };

    var result = await _orchestrator.EjecutarTransicionAsync(context, cancellationToken);

    if (!result.Exitoso || !result.IdEstadoConfiguracionDestino.HasValue)
        throw new CustomException(
            HttpStatusCode.InternalServerError,
            $"No se pudo completar la transición de Solicitud {solicitud.IdSolicitud} a estado '{estadoDestinoNombre}'");

    // Si el destino es PENDIENTE_APROBACION_CONSEJO, eliminar la relación
    // para que la solicitud pueda ser incluida en otra acta
    if (actaSolicitud.EstadoDestino?.ToUpperInvariant() == EstadoEnum.PendienteAprobacionConsejo.GetCodigo())
    {
        await uow.Core.ActasConsejoSolicitudes.DeleteAsync(actaSolicitud.IdActaConsejoSolicitud, idUsuarioActual);

        _logger?.LogInformation(
            "Relación eliminada: ActaConsejoSolicitud {IdActaConsejoSolicitud} para Solicitud {IdSolicitud} con estado PENDIENTE_APROBACION_CONSEJO",
            actaSolicitud.IdActaConsejoSolicitud, actaSolicitud.IdSolicitud);
    }

    solicitud.IdEstadoConfiguracion = result.IdEstadoConfiguracionDestino.Value;
    await uow.Core.Solicitudes.UpdateAsync(solicitud, idUsuarioActual);

    _logger?.LogInformation(
        "Solicitud {IdSolicitud} transicionada exitosamente a estado '{EstadoDestino}'",
        solicitud.IdSolicitud, estadoDestinoNombre);
}
```

---

## 4. Comparación de Responsabilidades

| Aspecto | `ProcesarSolicitudIndividual` (actual) | `EjecutarTransicionSolicitud` (propuesto) |
|---------|----------------------------------------|-------------------------------------------|
| Carga de datos | Hace 4 queries por llamada | Recibe datos ya cargados |
| Validación de estados | Requiere query y lanza excepción en medio del loop | Validado en pre-carga |
| Responsabilidad | Carga + validación + orquestación + escritura | Solo orquestación + escritura |
| Testabilidad | Difícil de aislar (requiere mocks de 4 repos) | Fácil (solo orquestador + 2 escrituras) |

---

## 5. Impacto de Performance

| Escenario | Queries actuales | Queries propuestas |
|-----------|-----------------|-------------------|
| 10 solicitudes, 10 empresas | ~40 queries | 12 (1+1+10 empresas) |
| 30 solicitudes, 10 empresas | ~120 queries | 12 (1+1+10 empresas) |
| 30 solicitudes, 30 empresas | ~120 queries | 32 (1+1+30 empresas) |

> Las queries de **escritura** (orchestrator, UpdateAsync, DeleteAsync) no cambian en cantidad.

---

## 6. Dependencias

No requiere cambios en repositorios ni en infraestructura. Todos los métodos batch utilizados (`GetByLoteIdAsync`, `GetByIdTipoEntidad`) ya existen en sus respectivas interfaces de dominio.

El refactor es **puramente de la capa Application** y no rompe ningún contrato externo.
