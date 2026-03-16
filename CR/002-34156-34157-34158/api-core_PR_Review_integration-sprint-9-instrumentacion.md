# PR Review — `integration/sprint-9-instrumentacion`

> **Base:** `release/3.0` → **Head:** `integration/sprint-9-instrumentacion`
> **US cubiertas:** [34156] COR-OTG-FAC · [34157] COR-OTG-PDP · [34158] COR-OTG-AVA
> **Fecha de revisión:** 16/03/2026

---

## 1. Resumen ejecutivo

El PR implementa el tramo final del flujo de instrumentación: solicitud de facturación → confirmación de pago → emisión de aval. Las tres features forman un pipeline coherente, usan el mismo patrón de repositorio/CQRS del proyecto y cuentan con cobertura de unit tests e integration tests. Se identifican **dos issues que requieren corrección** antes del merge y varias observaciones menores.

**Veredicto:** ⚠️ Aprueba con correcciones requeridas

---

## 2. Inventario de cambios

### US 34156 — Solicitar facturación

| Archivo | Tipo | Descripción |
|---|---|---|
| `Commands/SolicitarFacturacionCommand.cs` | Nuevo | Valida conjunto contractual completo, verifica no-duplicate, inserta evento de facturación |
| `Dtos/Requests/SolicitarFacturacionRequestDto.cs` | Nuevo | DTO de request (IdSolicitud desde ruta) |
| `Dtos/Responses/SolicitarFacturacionResponseDto.cs` | Nuevo | Respuesta con `Exitoso`, `IdSolicitudFacturacion`, `EstadoConjuntoContractual` |
| `Dtos/Validators/SolicitarFacturacionRequestDtoValidator.cs` | Nuevo | Valida `IdSolicitud > 0` |
| `Core.Domain/Core/Contracts/ISolicitudesFacturacionRepository.cs` | Nuevo | `InsertAsync` + `ExistsBySolicitudAsync` |
| `Core.Domain/Core/Entities/SolicitudFacturacion.cs` | Nuevo | Entidad de dominio con `IdSolicitud`, `FechaSolicitud`, `IdUsuarioSolicitud` |
| `Core.Infraestructure/Repositories/Core/SolicitudesFacturacionRepository.cs` | Nuevo | SPs: `Core.SolicitudesFacturacion_Insert`, `Core.SolicitudesFacturacion_BySolicitud_Exists` |
| `SolicitudesGarantiaController.cs` | Modificado | `POST {idSolicitud}/solicitar-facturacion` |
| `EstadoConfiguracionTransicionCommand.cs` | Modificado | **Reemplaza mocks TODOs** de `contrato_firmado` y `orden_emision_factura_existe` por consultas reales a DB |
| `UnitTesting/SolicitarFacturacionCommandHandlerTests.cs` | Nuevo | 8 tests (happy path, incompleto, duplicado, NotFound, IdCero, orden de llamadas, vacío, IdFacturacion) |
| `IntegrationTesting/SolicitudesFacturacionTests.cs` | Nuevo | 2 tests (ExistsFalse, tipo booleano) |

### US 34157 — Pendiente de Pago / Pago confirmado

| Archivo | Tipo | Descripción |
|---|---|---|
| `Commands/SolicitudPagoConfirmarCommand.cs` | Nuevo | Valida estado `PendientePagoGarantia`, verifica no-duplicate, inserta evento de pago confirmado |
| `Dtos/Requests/SolicitudPagoConfirmarRequestDto.cs` | Nuevo | DTO de request (IdSolicitud desde ruta) |
| `Dtos/Responses/SolicitudPagoConfirmarResponseDto.cs` | Nuevo | Respuesta con `IdSolicitudPagoConfirmado`, `FechaPagoConfirmado`, `Mensaje` |
| `Dtos/Validators/SolicitudPagoConfirmarRequestDtoValidator.cs` | Nuevo | Valida `IdSolicitud > 0` |
| `Core.Domain/Core/Contracts/ISolicitudesPagoConfirmadoRepository.cs` | Nuevo | `InsertAsync` + `ExistsBySolicitudAsync` |
| `Core.Domain/Core/Entities/SolicitudPagoConfirmado.cs` | Nuevo | Entidad de dominio con `FechaPagoConfirmado`, `IdUsuarioPagoConfirmado` |
| `Core.Infraestructure/Repositories/Core/SolicitudesPagoConfirmadoRepository.cs` | Nuevo | SPs: `Core.SolicitudesPagoConfirmado_Insert`, `Core.SolicitudesPagoConfirmado_BySolicitud_Exists` |
| `SolicitudesGarantiaController.cs` | Modificado | `POST {idSolicitud}/confirmar-pago` |
| `EstadoConfiguracionTransicionCommand.cs` | Modificado | Reemplaza mock `factura_pagada` por `SolicitudesPagoConfirmado.ExistsBySolicitudAsync` real |
| `UnitTesting/SolicitudPagoConfirmarCommandHandlerTests.cs` | Nuevo | 9 tests (exitoso, duplicado, estado incorrecto, `[Theory]` con 3 estados, NotFound, IdCero, orden de llamadas) |
| `IntegrationTesting/SolicitudesPagoConfirmadoTests.cs` | Nuevo | 3 tests (SinPago→False, IdCero→False, IdNegativo→False) |

### US 34158 — Emitir aval

| Archivo | Tipo | Descripción |
|---|---|---|
| `Commands/EmitirAvalCommand.cs` | Nuevo | Valida pago confirmado → obtiene FlujoEntidad → verifica idempotencia via tracking → registra `"Aval emitido"` |
| `Dtos/Requests/EmitirAvalRequestDto.cs` | Nuevo | DTO de request (no utilizado por el endpoint, ver Obs. 5.1) |
| `Dtos/Responses/EmitirAvalResponseDto.cs` | Nuevo | Respuesta con `IdSolicitud`, `IdTracking`, `FechaEmision`, `Mensaje` |
| `Dtos/Validators/EmitirAvalRequestDtoValidator.cs` | Nuevo | Valida `IdSolicitud > 0` (no invocado, ver Obs. 5.1) |
| `Core.Domain/GestionFlujos/Contracts/IFlujoEntidadTrackingRepository.cs` | Modificado | Agrega `ExistsByFlujoEntidadAndOrigenEventoAsync` para control de idempotencia |
| `Core.Infraestructure/Repositories/GestionFlujos/FlujoEntidadTrackingRepository.cs` | Modificado | Implementa el método con SP `GestionFlujos.FlujosEntidadesTracking_ByFlujoEntidadAndOrigenEvento_Select` |
| `SolicitudesGarantiaController.cs` | Modificado | `POST {idSolicitud}/emitir-aval` |
| `UnitTesting/EmitirAvalCommandHandlerTests.cs` | Nuevo | 8 tests (exitoso, tracking correcto, BadRequest pago, Conflict aval duplicado, NotFound, orden de llamadas, lista vacía, `[Theory]` con 3 ids) |
| `IntegrationTesting/EmitirAvalIntegrationTests.cs` | Nuevo | 6 tests (ExistsPago negativos × 3, ExistsTracking negativo × 2, GetFlujoEntidad vacío) |

### Archivos transversales modificados

| Archivo | Cambio |
|---|---|
| `ICoreSchemaContext.cs` | Agrega `ISolicitudesFacturacionRepository` e `ISolicitudesPagoConfirmadoRepository` |
| `CoreSchemaContext.cs` | Lazy-init de los 2 nuevos repositorios |
| `IOCCoreSchema.cs` | Registra 3 validators, 3 command handlers, 2 repositorios nuevos |
| `IntegrationTesting/FirmasDocumentosSolicitudTests.cs` | Nuevo — 3 tests de integración para el repositorio de firmas |
| `EstadoConfiguracionTransicionCommandHandlerTests.cs` | Agrega mocks para los 3 nuevos repositorios de Core |

**Archivos nuevos/modificados en este PR:** ~35

---

## 3. Cobertura de Criterios de Aceptación

### US 34156 — Solicitar facturación

| Criterio | Estado | Observación |
|---|---|---|
| **CA1** – Bloquear si documentación incompleta | ✅ | `Exitoso=false` + `EstadoConjuntoContractual` con detalle de pendientes |
| **CA2** – Facturación exitosa → registrar evento | ✅ | `InsertAsync` + retorna `IdSolicitudFacturacion` |
| **CA3** – Trazabilidad: fecha y usuario | ✅ | `SolicitudFacturacion` hereda de `EntityBase`, el SP recibe `IdUsuarioSolicitud` |
| **RN01** – Acción manual | ✅ | Solo se ejecuta cuando el usuario llama al endpoint |
| **RN02** – No permite con docs incompletos | ✅ | `GetEstadoConjuntoAsync` + `!estadoConjunto.EsCompleto` |
| **RN03** – Pasa a estado "Pendiente de pago" | ⚠️ | El comando **no actualiza el estado de la solicitud** explícitamente. La transición de estado queda en manos del SP de `EstadoConfiguracionTransicion`. Verificar si el SP de Insert también actualiza el estado o si el flujo depende de un trigger externo. |
| **RN04** – No permite duplicar | ✅ | `ExistsBySolicitudAsync` + `409 Conflict` |

### US 34157 — Pendiente de Pago / Pago OK

| Criterio | Estado | Observación |
|---|---|---|
| **CA1** – Bloquear emisión de aval sin pago | ✅ | `EmitirAvalCommand` verifica `ExistsBySolicitudAsync` |
| **CA2** – Registrar pago confirmado → fecha y usuario | ✅ | `SolicitudPagoConfirmado` + `InsertAsync` con `idUsuario` |
| **CA3** – Habilitar emisión de aval | ✅ | El `EmitirAvalCommand` consulta el mismo repositorio como precondición |
| **RN01** – Pago manual | ✅ | Endpoint explícito |
| **RN02** – No emitir aval sin pago | ✅ | `400 BadRequest` en `EmitirAvalCommand` |
| **RN03** – Evento único por solicitud | ✅ | `ExistsBySolicitudAsync` + `409 Conflict` |
| **RN04** – Auditoría | ✅ | Entidad hereda de `EntityBase` |

### US 34158 — Emitir aval

| Criterio | Estado | Observación |
|---|---|---|
| **CA1** – Bloquear sin pago confirmado | ✅ | `400 BadRequest` si `!pagoConfirmado` |
| **CA2** – Registrar evento "Aval emitido" | ✅ | `FlujoEntidadTracking.OrigenEvento = "Aval emitido"` |
| **CA3** – Trazabilidad: fecha y usuario | ✅ | `FechaInicioPaso = DateTime.UtcNow` + `idUsuario` en `CreateAsync` |
| **RN01** – Requiere pago confirmado | ✅ | Primera validación del handler |
| **RN02** – Acción manual | ✅ | Endpoint explícito |
| **RN03** – Auditoría | ✅ | `FlujoEntidadTracking` |
| **RN04** – Habilita paso de certificado | ⚠️ | El tracking queda registrado, pero no hay lógica explícita que "habilite" el siguiente paso. Depende de cómo el frontend interprete la presencia del tracking con `OrigenEvento = "Aval emitido"`. |

---

## 4. Puntos positivos

- **Pipeline de tres pasos cohesivo.** Las tres US forman exactamente la cadena: `SolicitarFacturacion → ConfirmarPago → EmitirAval`, donde cada paso valida que el anterior se completó.
- **Eliminación de mocks TODO en `EstadoConfiguracionTransicionCommand`.** Los campos `contrato_firmado`, `orden_emision_factura_existe` y `factura_pagada` ahora se calculan desde la DB real. Mejora crítica de consistencia.
- **Idempotencia en los tres commands.** Cada uno verifica existencia previa antes de insertar, retornando `409 Conflict` ante duplicados.
- **Tests de orden de llamadas (call sequence).** Los tests de `EmitirAvalCommand` y `SolicitarFacturacionCommand` verifican el orden exacto de las validaciones usando callbacks. Técnica de alta calidad.
- **`[Theory]` en `SolicitudPagoConfirmarCommandHandlerTests`.** Verifica que múltiples estados incorrectos sean rechazados sin duplicar código de test.
- **Integration tests de solo lectura.** Todos los tests de integración usan IDs altos/negativos o inexistentes, sin efectos secundarios sobre la DB.
- **`ExistsByFlujoEntidadAndOrigenEventoAsync` en `IFlujoEntidadTrackingRepository`.** Extiende correctamente la interfaz existente en lugar de crear una nueva entidad para el tracking del aval.

---

## 5. Observaciones / Issues

### 5.1 🔴 Validators con campos de ruta — mismo patrón ya corregido en COR-OTG-SFI

**Severidad: Alta — puede provocar comportamientos inesperados en producción**

Los tres nuevos validators están registrados en DI pero **ninguno de los endpoints tiene `[FromBody]`**, por lo que nunca son invocados:

```csharp
// Ninguno de estos DTOs es un [FromBody], vienen de la ruta
builder.Services.AddTransient<IValidator<SolicitarFacturacionRequestDto>, SolicitarFacturacionRequestDtoValidator>();
builder.Services.AddTransient<IValidator<SolicitudPagoConfirmarRequestDto>, SolicitudPagoConfirmarRequestDtoValidator>();
builder.Services.AddTransient<IValidator<EmitirAvalRequestDto>, EmitirAvalRequestDtoValidator>();
```

Adicionalmente, `EmitirAvalRequestDto` **no está instanciado en ningún lugar del controller o command**:

```csharp
// EmitirAval endpoint — no hay new EmitirAvalRequestDto en ningún lado
public async Task<ApiResult<EmitirAvalResponseDto>> EmitirAval([FromRoute] int idSolicitud)
{
    return await _dispatcher.Send(new EmitirAvalCommand { IdSolicitud = idSolicitud }, ...);
}
```

**Recomendación:**
- Eliminar los tres registros de DI (validators inutilizados).
- Eliminar `EmitirAvalRequestDto`, `EmitirAvalRequestDtoValidator`, `SolicitarFacturacionRequestDto`, `SolicitarFacturacionRequestDtoValidator`, `SolicitudPagoConfirmarRequestDto`, `SolicitudPagoConfirmarRequestDtoValidator` — los tres DTOs de request son innecesarios ya que los commands reciben los valores directamente desde la ruta.

---

### 5.2 🔴 Regresión en `RegistrarFirmaDocumentoSolicitudRequestDtoValidator` y reintroducción de `SolicitarFirmaDocumentoSolicitudRequestDtoValidator`

**Severidad: Alta — regresión de un fix aplicado al branch anterior**

El diff contra `release/3.0` muestra que esta rama **agrega de nuevo** las reglas de ruta en el validator:

```diff
+ RuleFor(x => x.IdSolicitud).GreaterThan(0)...
+ RuleFor(x => x.IdDocumentoSolicitud).GreaterThan(0)...
  RuleFor(x => x.IdArchivoFirmado).GreaterThan(0)...
```

Y en `IOCCoreSchema.cs` vuelve a aparecer:

```csharp
builder.Services.AddTransient<IValidator<SolicitarFirmaDocumentoSolicitudRequestDto>, SolicitarFirmaDocumentoSolicitudRequestDtoValidator>();
```

Ambas correcciones fueron aplicadas al branch `COR-OTG-SFI-firma-conjunto-contractual` pero **no fueron incorporadas** a esta rama de integración.

**Recomendación:** Verificar que el merge de `COR-OTG-SFI` en esta rama incluya los fixes. Corregir antes del merge a `release/3.0`.

---

### 5.3 ⚠️ `SolicitarFacturacionCommand` retorna 200 OK con `Exitoso=false` en lugar de 400

El handler retorna un DTO con `Exitoso = false` cuando el conjunto contractual está incompleto, en vez de lanzar `CustomException(HttpStatusCode.BadRequest, ...)`. Esto es inconsistente con el resto de la aplicación donde las violaciones de reglas de negocio se expresan con HTTP 4xx.

```csharp
// Comportamiento actual — HTTP 200 con cuerpo de error
return new SolicitarFacturacionResponseDto { Exitoso = false, Mensaje = "..." };

// Patrón consistente del proyecto
throw new CustomException(HttpStatusCode.BadRequest, "...");
```

El frontend debe chequear `Exitoso` en vez de confiar en el status code. Sin embargo, la ventaja del enfoque actual es que devuelve `EstadoConjuntoContractual` con el detalle de los documentos pendientes, lo cual es valioso para el UX.

**Recomendación:** Mantener el detalle de `EstadoConjuntoContractual` pero cambiar a `throw new CustomException(HttpStatusCode.UnprocessableEntity, ...)` y pasar el estado del conjunto en el mensaje o como objeto adjunto. Alternativamente, documentar la decisión de diseño explícitamente en el handler.

---

### 5.4 ⚠️ `DateTime.Now` vs `DateTime.UtcNow` — inconsistencia

`SolicitudPagoConfirmarCommandHandler` usa `DateTime.Now` (hora local del servidor):

```csharp
FechaPagoConfirmado = DateTime.Now,
```

`EmitirAvalCommandHandler` usa `DateTime.UtcNow`:

```csharp
var fechaEmision = DateTime.UtcNow;
```

El resto del proyecto (por ejemplo `FirmasDocumentosSolicitudRepository`) no fija la fecha en la capa de aplicación sino en el SP. Estandarizar con `DateTime.UtcNow` o delegar la fecha al SP.

---

### 5.5 ⚠️ `Pendientes` puede contener nulls tras refactor en `GetEstadoConjuntoAsync`

El refactor en `FirmasDocumentosSolicitudRepository` cambió de:

```csharp
.Select(f => f.TituloDocumentoTipo ?? f.NombreDocumentoTipo)
```

a:

```csharp
.Select(f => f.TituloDocumentoTipo)
```

Si `TituloDocumentoTipo` es null para algún documento, la lista `Pendientes` contendrá nulls. Esto puede causar `NullReferenceException` en el frontend al iterar la lista.

**Recomendación:** Restaurar el fallback `?? f.NombreDocumentoTipo` o usar `.Where(v => v != null)` tras el select.

---

### 5.6 ℹ️ Visibilidad de repositorios: `internal` → `public`

`FirmasDocumentosSolicitudRepository` cambió de `internal sealed` a `public sealed`. Los dos nuevos repositorios son `public sealed` / `public class`. La convención del proyecto (ver otros repositorios) es `internal sealed` para mantener el encapsulamiento dentro del proyecto de infraestructura.

---

### 5.7 ℹ️ `SolicitudPagoConfirmarCommand` — navegación nullable `estadoConfig.Estado?.Nombre`

```csharp
var estadoActualNombre = estadoConfig.Estado?.Nombre ?? string.Empty;
```

Si `Estado` es null, `estadoActualNombre` es `""` y la comparación con el estado esperado fallará con `400 BadRequest` en vez de `500`. Debería validarse explícitamente que `Estado` no sea null.

---

### 5.8 ℹ️ `EmitirAvalCommand` — selección de `FlujoEntidad` por `LastOrDefault`

```csharp
var flujoEntidad = flujosEntidad?
    .Where(f => f != null)
    .OrderBy(f => f!.IdFlujoEntidad)
    .LastOrDefault();
```

Toma el último `FlujoEntidad` por ID ascendente. Si la solicitud tiene múltiples entradas de flujo (ej. por reactivaciones), esto podría dar un resultado inesperado. Sería más explícito `.OrderByDescending(...).FirstOrDefault()`.

---

## 6. SPs esperados en base de datos

| SP | US | Acción |
|---|---|---|
| `Core.SolicitudesFacturacion_Insert` | 34156 | Inserta evento de facturación |
| `Core.SolicitudesFacturacion_BySolicitud_Exists` | 34156 | Verifica existencia |
| `Core.SolicitudesPagoConfirmado_Insert` | 34157 | Inserta evento de pago confirmado |
| `Core.SolicitudesPagoConfirmado_BySolicitud_Exists` | 34157 | Verifica existencia de pago |
| `GestionFlujos.FlujosEntidadesTracking_ByFlujoEntidadAndOrigenEvento_Select` | 34158 | Verifica idempotencia de "Aval emitido" |

⚠️ **Verificar** que estos 5 SPs estén incluidos en los scripts de migración de la release.

---

## 7. Checklist de revisión

| Criterio | Estado |
|---|---|
| Sigue Clean Architecture (capas y dependencias correctas) | ✅ |
| Sigue CQRS (Commands/Queries/Handlers separados) | ✅ |
| Naming conventions del proyecto | ✅ |
| Sin SQL inline (todo vía SPs) | ✅ |
| Validators útiles registrados correctamente en DI | 🔴 (Obs. 5.1) |
| Regresión de fix de validators de COR-OTG-SFI | 🔴 (Obs. 5.2) |
| Handlers registrados en DI | ✅ |
| Repositorios registrados en DI | ✅ |
| Idempotencia protegida (no duplicados) | ✅ |
| Manejo de errores con `CustomException` y HTTP codes correctos | ⚠️ (Obs. 5.3) |
| Consistencia de fechas UTC | ⚠️ (Obs. 5.4) |
| `Pendientes` sin nulls | ⚠️ (Obs. 5.5) |
| Unit tests (happy path + error paths) | ✅ |
| Integration tests | ✅ |
| Mock TODOs reemplazados por datos reales | ✅ |
| Script SQL de SPs incluido | ❓ (no visible en el diff) |

---

## 8. Conclusión

Las tres US están implementadas correctamente y forman el pipeline de instrumentación esperado. Los issues a resolver antes del merge son:

1. **Obs. 5.1 — Validators / DTOs de request innecesarios:** Eliminar los tres pares DTO+Validator que no son invocados por ningún endpoint. Si el command valida directamente (usando FluentValidation sobre el command), es la vía correcta.
2. **Obs. 5.2 — Regresión de fixes de COR-OTG-SFI:** Aplicar o re-mergear los fixes del branch anterior antes de mergear a `release/3.0`.
3. **Obs. 5.5 — Posibles nulls en lista `Pendientes`:** Restaurar el fallback `?? NombreDocumentoTipo`.

Los puntos 5.3, 5.4, 5.6–5.8 son mejoras de calidad que pueden resolverse en este PR o quedar como deuda técnica documentada según criterio del equipo.
