# PR Review — `integration/sprint-9-instrumentacion`

> **Base:** `release/3.0` → **Head:** `integration/sprint-9-instrumentacion`
> **US cubiertas:** [34156] COR-OTG-FAC · [34157] COR-OTG-PDP · [34158] COR-OTG-AVA
> **Fecha de revisión:** 16/03/2026

> **v2 — 16/03/2026:** Revisión completa desde el código en disco. La v1 contenía varias observaciones incorrectas (validators de ruta que no existen, `DateTime.Now` incorrecto, `Exitoso=false` incorrecto, fallback de Pendientes eliminado — todos estos puntos fueron errores del análisis inicial). Esta versión refleja el estado real del código.

---

## 1. Resumen ejecutivo

El PR implementa el tramo final del flujo de instrumentación: solicitud de facturación → confirmación de pago → emisión de aval. Las tres features forman un pipeline coherente, siguen el patrón del proyecto, incluyen tests unitarios e integration tests de buena calidad, y corrigen las metadata hardcodeadas de `EstadoConfiguracionTransicionCommand`. También incorpora la feature de comparación carta banco (US 34022, branch `feature/34525`) como parte de la integración.

Se identifican **tres issues que requieren corrección** antes del merge (dos heredados de la feature de carta banco) y varias observaciones menores.

**Veredicto:** ⚠️ Aprueba con correcciones requeridas

---

## 2. Inventario de cambios

### US 34156 — Solicitar facturación

| Archivo | Tipo | Descripción |
|---|---|---|
| `Commands/SolicitarFacturacionCommand.cs` | Nuevo | Valida solicitud → conjunto contractual completo → no-duplicate → inserta evento |
| `Dtos/Responses/SolicitarFacturacionResponseDto.cs` | Nuevo | Respuesta con `Exitoso`, `IdSolicitudFacturacion`, `EstadoConjuntoContractual?` |
| `Core.Domain/Core/Contracts/ISolicitudesFacturacionRepository.cs` | Nuevo | `InsertAsync` + `ExistsBySolicitudAsync` |
| `Core.Domain/Core/Entities/SolicitudFacturacion.cs` | Nuevo | Entidad con `IdSolicitud`, `FechaSolicitud`, `IdUsuarioSolicitud`, hereda de `EntityBase` |
| `Core.Infraestructure/Repositories/Core/SolicitudesFacturacionRepository.cs` | Nuevo | SPs: `Core.SolicitudesFacturacion_Insert`, `Core.SolicitudesFacturacion_BySolicitud_Select` |
| `SolicitudesGarantiaController.cs` | Modificado | `POST {idSolicitud}/solicitar-facturacion` — `[FromRoute]` |
| `EstadoConfiguracionTransicionCommand.cs` | Modificado | Reemplaza mock `contrato_firmado` y `orden_emision_factura_existe` por consultas reales a DB |
| `UnitTesting/SolicitarFacturacionCommandHandlerTests.cs` | Nuevo | 7 tests (happy path, incompleto, duplicado, NotFound, IdCero, orden de llamadas, conjunto vacío) |
| `IntegrationTesting/SolicitudesFacturacionTests.cs` | Nuevo | 2 tests de integración |

### US 34157 — Pago confirmado

| Archivo | Tipo | Descripción |
|---|---|---|
| `Commands/SolicitudPagoConfirmarCommand.cs` | Nuevo | Valida solicitud → estado `PendientePagoGarantia` → no-duplicate → inserta evento |
| `Dtos/Responses/SolicitudPagoConfirmarResponseDto.cs` | Nuevo | Respuesta con `IdSolicitudPagoConfirmado`, `FechaPagoConfirmado`, `Mensaje` |
| `Core.Domain/Core/Contracts/ISolicitudesPagoConfirmadoRepository.cs` | Nuevo | `InsertAsync` + `ExistsBySolicitudAsync` |
| `Core.Domain/Core/Entities/SolicitudPagoConfirmado.cs` | Nuevo | Entidad con `FechaPagoConfirmado`, `IdUsuarioPagoConfirmado`, hereda de `EntityBase` |
| `Core.Infraestructure/Repositories/Core/SolicitudesPagoConfirmadoRepository.cs` | Nuevo | SPs: `Core.SolicitudesPagoConfirmado_Insert`, `Core.SolicitudesPagoConfirmado_BySolicitud_Select` |
| `SolicitudesGarantiaController.cs` | Modificado | `POST {idSolicitud}/confirmar-pago` — `[FromRoute]` |
| `EstadoConfiguracionTransicionCommand.cs` | Modificado | Reemplaza mock `factura_pagada` por `SolicitudesPagoConfirmado.ExistsBySolicitudAsync` real |
| `UnitTesting/SolicitudPagoConfirmarCommandHandlerTests.cs` | Nuevo | 8 tests (exitoso, duplicado, estado incorrecto, `[Theory]` con 3 estados, NotFound, IdCero, orden de llamadas) |
| `IntegrationTesting/SolicitudesPagoConfirmadoTests.cs` | Nuevo | 3 tests de integración |

### US 34158 — Emitir aval

| Archivo | Tipo | Descripción |
|---|---|---|
| `Commands/EmitirAvalCommand.cs` | Nuevo | Valida pago confirmado → obtiene FlujoEntidad → verifica idempotencia → registra tracking |
| `Dtos/Responses/EmitirAvalResponseDto.cs` | Nuevo | Respuesta con `IdSolicitud`, `IdTracking`, `FechaEmision`, `Mensaje` |
| `Core.Domain/GestionFlujos/Contracts/IFlujoEntidadTrackingRepository.cs` | Modificado | Agrega `ExistsByFlujoEntidadAndOrigenEventoAsync` |
| `Core.Infraestructure/Repositories/GestionFlujos/FlujoEntidadTrackingRepository.cs` | Modificado | Implementa el método con SP `GestionFlujos.FlujosEntidadesTracking_ByFlujoEntidadAndOrigenEvento_Select` |
| `SolicitudesGarantiaController.cs` | Modificado | `POST {idSolicitud}/emitir-aval` — `[FromRoute]` |
| `UnitTesting/EmitirAvalCommandHandlerTests.cs` | Nuevo | 8 tests (exitoso, tracking correcto, BadRequest sin pago, Conflict duplicado, NotFound, orden de llamadas, lista vacía, `[Theory]` con 3 IDs) |
| `IntegrationTesting/EmitirAvalIntegrationTests.cs` | Nuevo | 6 tests de integración |

### US 34022 — Comparación carta banco (incluida por integración)

| Archivo | Tipo | Descripción |
|---|---|---|
| `Commands/ComparacionCartaBancoResolucionCommand.cs` | Nuevo | Ver review `MP/001-34022-COR-OTG-CBC` |
| `Commands/GuardarComparacionCartaBancoCommand.cs` | Nuevo | Ver review `MP/001-34022-COR-OTG-CBC` |
| `Controller/ComparacionCartaBancoController.cs` | Nuevo | `POST /` y `PATCH /comentario` |
| (+ demás archivos de la feature) | Nuevo | Ver review MP para detalle completo |

### Archivos transversales

| Archivo | Cambio |
|---|---|
| `ICoreSchemaContext.cs` | Agrega 3 nuevos repositorios |
| `CoreSchemaContext.cs` | Lazy-init para los 3 nuevos repositorios |
| `IParametrizacionesSchemaContext.cs` | Agrega `IParametrosSistemaRepository` |
| `ParametrizacionesSchemaContext.cs` | Lazy-init `ParametrosSistema` |
| `IOCCoreSchema.cs` | Registra handlers, validator carta banco, service carta banco, opciones, repositorios |
| `FirmasDocumentosSolicitudRepository.cs` | Refactoriza `GetEstadoConjuntoAsync` con manejo explícito de lista vacía |
| `EstadoConfiguracionTransicionCommandHandlerTests.cs` | Agrega mocks para los 3 repos nuevos |
| `FirmasDocumentosSolicitudTests.cs` | Nuevo — 3 tests de integración de repo firmas |
| `appsettings.Development.json` | Agrega `"ComparacionCartaBanco": { "UseMockData": true }`, reformateo completo |

**Archivos nuevos/modificados:** ~38

---

## 3. Cobertura de Criterios de Aceptación

### US 34156 — Solicitar facturación

| Criterio | Estado | Observación |
|---|---|---|
| **CA1** – Bloquear si documentación incompleta | ✅ | `CustomException(422)` con lista de documentos pendientes |
| **CA2** – Registrar evento de facturación | ✅ | `InsertAsync` → retorna `IdSolicitudFacturacion` |
| **CA3** – Trazabilidad: fecha y usuario | ✅ | `SolicitudFacturacion` hereda de `EntityBase`; SP recibe `IdUsuarioSolicitud` |
| **RN01** – Acción manual | ✅ | Solo se ejecuta por llamada explícita al endpoint |
| **RN02** – No permite con docs incompletos | ✅ | `GetEstadoConjuntoAsync` + `!estadoConjunto.EsCompleto` |
| **RN03** – No permite duplicar | ✅ | `ExistsBySolicitudAsync` + `409 Conflict` |

### US 34157 — Pago confirmado

| Criterio | Estado | Observación |
|---|---|---|
| **CA1** – Habilitar emisión de aval tras pago | ✅ | `EmitirAvalCommand` consulta `ExistsBySolicitudAsync` como precondición |
| **CA2** – Registrar pago con fecha y usuario | ✅ | `SolicitudPagoConfirmado` + `InsertAsync` con `idUsuario` |
| **CA3** – Validar estado `PendientePagoGarantia` | ✅ | Compara `estadoConfig.Estado.Nombre` con `EstadoEnum.PendientePagoGarantia.GetNombre()` |
| **RN01** – Pago manual | ✅ | Endpoint explícito |
| **RN02** – Evento único por solicitud | ✅ | `ExistsBySolicitudAsync` + `409 Conflict` |
| **RN03** – Auditoría | ✅ | Entidad hereda de `EntityBase` |

### US 34158 — Emitir aval

| Criterio | Estado | Observación |
|---|---|---|
| **CA1** – Bloquear sin pago confirmado | ✅ | `400 BadRequest` si `!pagoConfirmado` |
| **CA2** – Registrar evento "Aval emitido" | ✅ | `FlujoEntidadTracking.OrigenEvento = "Aval emitido"` |
| **CA3** – Trazabilidad: fecha y usuario | ✅ | `FechaInicioPaso = DateTime.UtcNow` + `idUsuario` en `CreateAsync` |
| **RN01** – Requiere pago confirmado | ✅ | Primera validación del handler |
| **RN02** – Acción manual | ✅ | Endpoint explícito |
| **RN03** – Idempotencia (no emitir dos veces) | ✅ | `ExistsByFlujoEntidadAndOrigenEventoAsync` + `409 Conflict` |

---

## 4. Puntos positivos

- **Pipeline de tres pasos cohesivo.** Cada step valida que el anterior se completó: `SolicitarFacturacion → ConfirmarPago → EmitirAval`.
- **Eliminación de mocks TODO en `EstadoConfiguracionTransicionCommand`.** Los campos `contrato_firmado`, `orden_emision_factura_existe` y `factura_pagada` ahora se calculan desde la DB real. Mejora crítica de consistencia.
- **`Convert.ToInt32` consistente.** Los tres nuevos command handlers usan `Convert.ToInt32(_userService.CurrentUserId)`, alineado con el patrón del proyecto.
- **`DateTime.UtcNow` consistente.** Todos los timestamps usan UTC.
- **Tests de orden de llamadas.** Los tests de `EmitirAvalCommand` y `SolicitarFacturacionCommand` verifican el orden exacto de validaciones con callbacks.
- **`[Theory]` en `SolicitudPagoConfirmarCommandHandlerTests`.** Verifica que múltiples estados incorrectos sean rechazados sin duplicar código.
- **Integration tests de solo lectura.** Todos usan IDs altos/negativos/inexistentes, sin efectos secundarios sobre la DB.
- **`GetEstadoConjuntoAsync` refactorizado.** Maneja explícitamente el caso de lista vacía con early return, y preserva el fallback `TituloDocumentoTipo ?? NombreDocumentoTipo`.
- **Fallback en `Pendientes` preservado.** `.Select(f => f.TituloDocumentoTipo ?? f.NombreDocumentoTipo)` sigue presente; no hay riesgo de nulls en la lista.

---

## 5. Observaciones / Issues

### 5.1 ⚠️ `SolicitarFacturacionResponseDto.EstadoConjuntoContractual` nunca se popula

El campo está declarado con el comentario _"Se incluye cuando la validación falla para informar al usuario qué documentos están pendientes"_, pero el handler **lanza `CustomException(HttpStatusCode.UnprocessableEntity, ...)` en lugar de retornar el DTO** con ese campo populado:

```csharp
// Handler — falla tirando excepción, nunca con Exitoso=false
if (!estadoConjunto.EsCompleto)
{
    var pendientes = string.Join(", ", estadoConjunto.Pendientes);
    throw new CustomException(
        HttpStatusCode.UnprocessableEntity,
        $"...Faltan firmar {estadoConjunto.Pendientes.Count} documento(s): {pendientes}");
}
```

Resultado: el campo `EstadoConjuntoContractual` del DTO **siempre es `null`** en todo response exitoso, y en el caso de falla no llega al frontend porque es una excepción HTTP.

**Opciones:**
1. Eliminar `EstadoConjuntoContractual` del DTO response (si el mensaje de error en la excepción es suficiente).
2. Cambiar el handler para que retorne el DTO con `Exitoso = false` + `EstadoConjuntoContractual` poblado (requiere que el controller no mapee 200→OK automáticamente para fallos).

---

### 5.2 ⚠️ SP de existencia nombrado como `_Select` en lugar de `_Exists`

`SolicitudesFacturacionRepository` y `SolicitudesPagoConfirmadoRepository` usan `ExecuteScalarAsync<bool>` sobre SPs llamados `_BySolicitud_Select`:

```csharp
// Devuelve bool desde un SP llamado _Select
return await _executor.ExecuteScalarAsync<bool>(
    "Core.SolicitudesFacturacion_BySolicitud_Select", parameters);

return await _executor.ExecuteScalarAsync<bool>(
    "Core.SolicitudesPagoConfirmado_BySolicitud_Select", parameters);
```

El sufijo `_Select` en la convención del proyecto implica un resultado de filas (ej: `FirmasDocumentosSolicitud_BySolicitud_Select` retorna `IEnumerable<T>`). Los SPs de existencia deberían usar `_Exists` o `_BySolicitud_Exists` para indicar que retornan un escalar booleano. Si los SPs realmente retornan una fila y se castea el primer valor como bool, la nomenclatura es confusa.

**Recomendación:** Renombrar los SPs a `Core.SolicitudesFacturacion_BySolicitud_Exists` y `Core.SolicitudesPagoConfirmado_BySolicitud_Exists`, o documentar explícitamente que retornan `bit`.

---

### 5.3 🔴 División por cero en `CompararMonto` (heredado de feature/34525)

`ComparacionCartaBancoService.CompararMonto` divide sin verificar que `montoResolucion != 0`:

```csharp
var diferenciaPorcentaje = Math.Abs((montoCartaBanco - montoResolucion) / montoResolucion * 100);
```

Si `montoResolucion == 0`, lanza `DivideByZeroException`. Ver Obs. 5.2 de `MP/001-34022-COR-OTG-CBC` para el fix propuesto.

---

### 5.4 🔴 Validator faltante para `ComparacionCartaBancoResolucionRequestDto` (heredado de feature/34525)

El endpoint `POST /comparacion-carta-banco` acepta `[FromBody] ComparacionCartaBancoResolucionRequestDto` pero no hay validator registrado. `IdSolicitud = 0` llega al handler sin error de validación. Ver Obs. 5.3 de `MP/001-34022-COR-OTG-CBC`.

---

### 5.5 🔴 `FechaComparacion` nunca asignada en `GuardarComparacionCartaBancoResponseDto` (heredado de feature/34525)

El handler `GuardarComparacionCartaBancoCommandHandler` nunca asigna `FechaComparacion` en el response:

```csharp
return new GuardarComparacionCartaBancoResponseDto
{
    IdComparacionCartaBancoResolucion = dto.IdComparacion,
    IdSolicitud = dto.IdSolicitud,
    ResultadoOk = comparacionExistente.ResultadoOk,
    Comentario = dto.Comentario
    // FechaComparacion siempre 0001-01-01T00:00:00
};
```

Ver Obs. 5.10 de `MP/001-34022-COR-OTG-CBC`.

---

### 5.6 ⚠️ `int.Parse` en handlers de carta banco (heredado de feature/34525)

`ComparacionCartaBancoResolucionCommandHandler` y `GuardarComparacionCartaBancoCommandHandler` usan `int.Parse(_userService.CurrentUserId)`, inconsistente con los nuevos handlers de este PR que usan `Convert.ToInt32`:

```csharp
// handlers de este PR — correcto
var idUsuario = Convert.ToInt32(_userService.CurrentUserId);

// handlers de carta banco — inconsistente
var idUsuario = int.Parse(_userService.CurrentUserId); // lanza FormatException si null
```

---

### 5.7 ℹ️ `InicializarConjuntoContractualRequestDto` agregado pero no referenciado

El DTO `InicializarConjuntoContractualRequestDto` fue agregado como archivo nuevo, pero no está referenciado en ningún controller, command ni validator del branch. Podría ser código preparatorio para una futura tarea o un archivo huérfano.

**Recomendación:** Eliminar si no corresponde a este PR. Si es preparatorio, moverlo a una PR específica.

---

### 5.8 ℹ️ Reformateo completo de `appsettings.Development.json`

El archivo cambió de indentación 4 espacios a 2 espacios en su totalidad. Genera ruido en el diff. Solo el bloque `"ComparacionCartaBanco": { "UseMockData": true }` es funcional.

---

## 6. SPs esperados en base de datos

| SP | Acción |
|---|---|
| `Core.SolicitudesFacturacion_Insert` | Inserta evento de facturación |
| `Core.SolicitudesFacturacion_BySolicitud_Select` | Verifica existencia de facturación por solicitud (scalar bool) |
| `Core.SolicitudesPagoConfirmado_Insert` | Inserta evento de pago confirmado |
| `Core.SolicitudesPagoConfirmado_BySolicitud_Select` | Verifica existencia de pago por solicitud (scalar bool) |
| `GestionFlujos.FlujosEntidadesTracking_ByFlujoEntidadAndOrigenEvento_Select` | Verifica existencia de tracking por flujo y origen evento |
| + SPs de la feature carta banco | Ver review MP/001-34022-COR-OTG-CBC |

---

## 7. Checklist de revisión

| Criterio | Estado |
|---|---|
| Sigue Clean Architecture | ✅ |
| Sigue CQRS | ✅ |
| Naming conventions | ✅ |
| Sin SQL inline (todo vía SPs) | ✅ |
| `Convert.ToInt32` en handlers nuevos | ✅ |
| `DateTime.UtcNow` en handlers | ✅ |
| Validators en DI solo para `[FromBody]` DTOs | ✅ |
| Idempotencia en los 3 comandos | ✅ |
| Eliminación de mocks en EstadoConfiguracionTransicionCommand | ✅ |
| `GetEstadoConjuntoAsync` manejo de lista vacía | ✅ |
| Fallback `TituloDocumentoTipo ?? NombreDocumentoTipo` | ✅ |
| Unit tests (3 archivos, 23+ tests) | ✅ |
| Integration tests (4 archivos) | ✅ |
| `SolicitarFacturacionResponseDto.EstadoConjuntoContractual` nunca populado | ⚠️ (Obs. 5.1) |
| Naming de SPs de existencia (`_Select` vs `_Exists`) | ⚠️ (Obs. 5.2) |
| División por cero en `CompararMonto` (carta banco) | 🔴 (Obs. 5.3) |
| Validator faltante para ComparacionCartaBancoResolucionRequestDto | 🔴 (Obs. 5.4) |
| `FechaComparacion` sin asignar (carta banco) | 🔴 (Obs. 5.5) |
| `int.Parse` en handlers carta banco | ⚠️ (Obs. 5.6) |
| Scripts SQL de SPs | ❓ (no visible en diff) |

---

## 8. Conclusión

Las tres US principales (34156, 34157, 34158) están implementadas correctamente, con tests de calidad y sin los problemas que se mencionaron erróneamente en la v1 de esta revisión. Los issues bloqueantes son todos heredados de la feature de carta banco (US 34022) que fue integrada en este branch. Antes del merge:

1. **Obs. 5.3 — División por cero en `CompararMonto`:** Riesgo real en runtime.
2. **Obs. 5.4 — Validator faltante para `ComparacionCartaBancoResolucionRequestDto`:** `IdSolicitud = 0` llega al handler sin validación.
3. **Obs. 5.5 — `FechaComparacion` sin asignar:** El cliente siempre recibe `0001-01-01`.
4. **Obs. 5.1 — `EstadoConjuntoContractual` dead field:** Definir si es error de diseño o campo a eliminar del DTO.
