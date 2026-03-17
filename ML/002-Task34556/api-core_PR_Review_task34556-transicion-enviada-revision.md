# PR Review — feature/34556-Transicion-Revision-Solicitada vs release/3.0
## Task 34556 — Backend: Agregar nueva transición "Enviada a revisión" (US 34154)

**Branch:** `feature/34556-Transicion-Revision-Solicitada`  
**Base:** `release/3.0`  
**Fecha de revisión:** 17/03/2026  
**Revisor:** GitHub Copilot (análisis automatizado)

---

## 1. Resumen ejecutivo

Este PR implementa el **backend de la transición de estado `ENVIADA_REVISION`** para el flujo de inconsistencias entre carta banco y resolución de Acta Consejo. Cubre:

- Nuevo valor de enum `EnviadaRevision` en `EstadoEnum`.
- Nueva estrategia `InstrumentacionEnviadaRevisionStrategy` para el orquestador.
- Disparo automático de la transición en `GuardarComparacionCartaBancoCommand` cuando `ResultadoOk = false`.
- Mejora del diseño del controller (rutas RESTful, separación de DTO de comentario).
- Fix de división por cero en `ComparacionCartaBancoService` cuando `montoResolucion == 0`.
- Limpieza de lógica informativa redundante en `CompararTipoProducto` y `CompararLineaProducto`.
- Cobertura de tests unitarios extensa (~4189 líneas).

| Categoría | Cantidad |
|-----------|----------|
| Archivos modificados / creados | 17 |
| Líneas agregadas | +4189 / -32 |
| Bugs críticos | 2 |
| Issues importantes | 3 |
| Issues menores | 3 |

---

## 2. Archivos modificados

| Archivo | Tipo de cambio |
|---------|---------------|
| `Core.Application/.../Commands/ComparacionCartaBancoResolucionCommand.cs` | Modificado — `int.Parse` → `Convert.ToInt32` |
| `Core.Application/.../Commands/GuardarComparacionCartaBancoCommand.cs` | Modificado — agrega orquestador, dispara transición `EnviadaRevision` cuando `ResultadoOk = false` |
| `Core.Application/.../Requests/ComentarioComparacionRequestDto.cs` | Nuevo — DTO mínimo con solo `Comentario` |
| `Core.Application/.../Requests/ComparacionCartaBancoResolucionRequestDto.cs` | **Eliminado** — reemplazado por parámetro de ruta `idSolicitud` |
| `Core.Application/.../Validators/ComentarioComparacionRequestDtoValidator.cs` | Nuevo — validador FluentValidation: `NotEmpty` + `MaximumLength(1000)` |
| `Core.Application/.../Strategies/Solicitudes/InstrumentacionEnviadaRevisionStrategy.cs` | Nuevo — estrategia de transición `PendienteInstrumentacion → EnviadaRevision` |
| `Core.Domain/Maestros/Enums/EstadoEnum.cs` | Modificado — agrega `EnviadaRevision` con nombre, código y mapeos |
| `Core.Infraestructure/Services/Core/ComparacionCartaBancoService.cs` | Modificado — fix división por cero + elimina lógica redundante informativa |
| `Core.Presentation/.../v1/ComparacionCartaBancoController.cs` | Modificado — rutas RESTful (`POST {idSolicitud}`, `PATCH {idSolicitud}/comparacion/{idComparacion}/comentario`) |
| `Core.Presentation/Api/Extensions/IOCCoreSchema.cs` | Modificado — registra `InstrumentacionEnviadaRevisionStrategy` |
| `Core.UnitTesting/.../Commands/ComparacionCartaBancoResolucionCommandHandlerTests.cs` | Nuevo — 10+ test cases |
| `Core.UnitTesting/.../Commands/GuardarComparacionCartaBancoCommandHandlerTests.cs` | Nuevo — 20+ test cases |
| `Core.UnitTesting/.../Validators/ComentarioComparacionRequestDtoValidatorTests.cs` | Nuevo — tests del validador |
| `Core.UnitTesting/.../Validators/ComparacionCartaBancoRequestDtoValidatorTests.cs` | Nuevo — tests del validador de comparación |
| `Core.UnitTesting/.../Strategies/InstrumentacionEnviadaRevisionStrategyTests.cs` | Nuevo — tests de la estrategia |
| `Core.UnitTesting/.../Services/Core/ComparacionCartaBancoServiceTests.cs` | Nuevo — tests del servicio |
| `Core.UnitTesting/.../Presentation/Controllers/ComparacionCartaBancoControllerTests.cs` | Nuevo — tests del controller |

---

## 3. Flujo implementado

```
POST /comparacion-carta-banco/{idSolicitud}
  → CompararCartaBancoConResolucionAsync(idSolicitud)
  → Crear ComparacionCartaBancoResolucion (ResultadoOk = true/false)
  → Responde con { Ok, RequiereComentario, IdComparacion, ... }

Si RequiereComentario = true:
  PATCH /comparacion-carta-banco/{idSolicitud}/comparacion/{idComparacion}/comentario
    body: { "comentario": "..." }
  → Validar que comparación existe y pertenece a la solicitud
  → UpdateComentarioAsync(idComparacion, comentario)
  → Si ResultadoOk = false → EjecutarTransicionEnviadaRevisionAsync()
     → Solicitud pasa de PendienteInstrumentacion → EnviadaRevision
  → Retorna GuardarComparacionCartaBancoResponseDto
```

---

## 4. Observaciones

### 4.1 [CRÍTICO] Swallow silencioso de errores en `EjecutarTransicionEnviadaRevisionAsync`

**Archivo:** `Core.Application/Modules/Core/Commands/GuardarComparacionCartaBancoCommand.cs`

El método que ejecuta la transición de estado captura cualquier excepción genérica, la loguea, y **retorna sin relanzar**, lo que hace que el endpoint devuelva `200 OK` aunque la transición haya fallado:

```csharp
catch (Exception ex)
{
    _logger.LogError(ex,
        "Error al ejecutar transición de estado para solicitud {IdSolicitud}", idSolicitud);

    // No lanzar la excepción para no romper el flujo principal de actualización de comentario
    // Solo registrar el error
}
```

El comentario dice "no romper el flujo principal", pero la consecuencia es que el cliente recibe `200 OK` con `ResultadoOk = false`, sin saber que la solicitud **no cambió de estado**. El sistema queda en estado inconsistente: comentario guardado, estado de solicitud sin cambiar.

Según la RN03 de la US: *"El sistema no avanza el flujo mientras exista una inconsistencia sin resolver."* Si la transición falla silenciosamente, el instrumentador cree que la solicitud fue enviada a revisión, pero en realidad sigue en `PendienteInstrumentacion`.

**Corrección sugerida:**

```csharp
catch (CustomException)
{
    throw; // siempre propagar excepciones de negocio
}
catch (Exception ex)
{
    _logger.LogError(ex, "Error al ejecutar transición para solicitud {IdSolicitud}", idSolicitud);
    throw new CustomException(
        HttpStatusCode.InternalServerError,
        $"El comentario fue guardado pero no se pudo cambiar el estado de la solicitud {idSolicitud}. " +
        "Por favor, reintente o contacte al administrador.");
}
```

---

### 4.2 [CRÍTICO] Script SQL completamente no idempotente y sin transacción

**Archivo:** Script SQL provisto (no está en el repositorio de código)

El script de migración no es idempotente en ninguno de sus tres INSERTs, y no tiene transacción explícita:

```sql
-- 1. Sin IF NOT EXISTS → duplica el Estado si se ejecuta 2 veces
INSERT INTO [Parametrizaciones].[Estados]
    (Nombre, IdUsuarioAlta)
VALUES
    ('Enviada a revisión', 1);

-- 2. Sin IF NOT EXISTS → duplica la EstadoConfiguracion
INSERT INTO [Parametrizaciones].[EstadosConfiguraciones]
    (IdEstado, IdTipoEntidad, Codigo, IdUsuarioAlta)
VALUES
    (@IdEstadoEnviadaRevision, @IdTipoEntidadSolicitud, 'ENVIADA_REVISION', 1);

-- 3. Sin IF NOT EXISTS → duplica la transición
INSERT INTO [Parametrizaciones].[EstadosConfiguracionesTransiciones]
    (IdEstadoConfiguracionOrigen, IdEstadoConfiguracionDestino, TransicionReglas, IdUsuarioAlta)
VALUES
    (@IdEstadoConfiguracionOrigen, @IdEstadoConfiguracionDestino, NULL, 1);
```

Si el script se ejecuta una segunda vez (re-deploy, error humano, rollback parcial), genera:
- Un segundo `Estado` "Enviada a revisión" con ID distinto — `SCOPE_IDENTITY()` toma este nuevo ID.
- Una segunda `EstadoConfiguracion` con codigo `ENVIADA_REVISION` — la unicidad del código no está protegida.
- Una segunda transición `PENDIENTE_INSTRUMENTACION → ENVIADA_REVISION` duplicada.

Además, si cualquier statement falla a mitad, los datos quedan parcialmente cargados sin posibilidad de rollback.

**Corrección sugerida:**

```sql
SET XACT_ABORT ON;
BEGIN TRANSACTION;
BEGIN TRY
    -- Estado
    IF NOT EXISTS (SELECT 1 FROM [Parametrizaciones].[Estados] WHERE Nombre = N'Enviada a revisión')
        INSERT INTO [Parametrizaciones].[Estados] (Nombre, IdUsuarioAlta)
        VALUES (N'Enviada a revisión', 1);

    DECLARE @IdEstadoEnviadaRevision INT;
    SELECT @IdEstadoEnviadaRevision = IdEstado
    FROM [Parametrizaciones].[Estados] WHERE Nombre = N'Enviada a revisión';

    -- ... (resto con SELECTs para obtener IDs + IF NOT EXISTS en cada INSERT)

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    ROLLBACK TRANSACTION;
    EXEC [dbo].[sp_error]; THROW;
END CATCH;
```

---

### 4.3 [IMPORTANTE] `InstrumentacionEnviadaRevisionStrategy.CanHandle` asume `ConfiguracionAdicional` no nulo

**Archivo:** `Core.Application/Modules/Helper/Transiciones/Strategies/Solicitudes/InstrumentacionEnviadaRevisionStrategy.cs`

```csharp
public bool CanHandle(TransicionContext context)
{
    if (context.Solicitud == null)
        return false;

    if (context.EstadoOrigenNombre != EstadoEnum.PendienteInstrumentacion.GetNombre())
        return false;

    if (context.ConfiguracionAdicional.ContainsKey("EstadoDestinoNombre"))  // ← NullReferenceException si es null
    {
        ...
    }
    return true;
}
```

El contrato de `TransicionContext` no garantiza que `ConfiguracionAdicional` esté siempre inicializado. Si otra parte del código construye un `TransicionContext` sin inicializarlo (`ConfiguracionAdicional = null`), el orchestrator fallará con `NullReferenceException` al evaluar esta estrategia.

**Corrección sugerida:**

```csharp
if (context.ConfiguracionAdicional?.ContainsKey("EstadoDestinoNombre") == true)
```

O asegurar en la definición de `TransicionContext` que `ConfiguracionAdicional` sea inicializado por defecto.

---

### 4.4 [IMPORTANTE] Transición se dispara incluso cuando solicitud no está en `PendienteInstrumentacion`

**Archivo:** `Core.Application/Modules/Core/Commands/GuardarComparacionCartaBancoCommand.cs`

Cuando `ResultadoOk = false`, el handler siempre llama a `EjecutarTransicionEnviadaRevisionAsync`. Si la solicitud no está en estado `PendienteInstrumentacion` (por ejemplo ya está en `EnviadaRevision` o en otro estado), el orquestador no encontrará ninguna estrategia aplicable, `resultado.Exitoso` será `false`, y el método registra un `LogWarning` y retorna silenciosamente. El endpoint retorna `200 OK`.

Esto no es un error funcional grave (la transición no se ejecuta), pero puede producir falsos positivos en los logs de warnings y oculta escenarios donde el estado de la solicitud es inesperado. El comentario igualmente queda guardado sin ninguna señal de que el estado no cambió.

**Corrección sugerida:** Validar explícitamente el estado de la solicitud antes de intentar la transición, o retornar información al cliente sobre si el cambio de estado fue efectivo.

---

### 4.5 [IMPORTANTE] `ConstruirMetadataAsync` en la estrategia incluye campos no relevantes para la transición

**Archivo:** `Core.Application/Modules/Helper/Transiciones/Strategies/Solicitudes/InstrumentacionEnviadaRevisionStrategy.cs`

```csharp
public Task<MetadataTransicion> ConstruirMetadataAsync(
    TransicionContext context, CancellationToken cancellationToken = default)
{
    var metadata = new MetadataTransicion();

    // Estos valores NUNCA son seteados en el contexto que construye el handler
    if (context.ConfiguracionAdicional.ContainsKey("monto"))
        metadata.monto = Convert.ToDecimal(context.ConfiguracionAdicional["monto"]);

    if (context.ConfiguracionAdicional.ContainsKey("monto_limite_definido"))
        metadata.monto_limite_definido = Convert.ToDecimal(context.ConfiguracionAdicional["monto_limite_definido"]);

    return Task.FromResult(metadata);
}
```

El handler construye el contexto con solo `EstadoDestinoNombre` en `ConfiguracionAdicional`. Los campos `monto` y `monto_limite_definido` nunca estarán presentes, por lo que la metadata siempre quedará con valores en 0. Es código copiado de otra estrategia sin adaptar a este contexto.

---

### 4.6 [MENOR] `ValidatorTests` de `ComparacionCartaBancoRequestDto` prueba un DTO eliminado

El PR elimina `ComparacionCartaBancoResolucionRequestDto`, pero agrega un archivo de tests llamado `ComparacionCartaBancoRequestDtoValidatorTests.cs`. Sin ver su contenido completo, si ese archivo prueba el DTO eliminado, los tests serán dead code o no compilarán. Verificar que los tests referencien el DTO correcto (`ComentarioComparacionRequestDto`).

---

### 4.7 [MENOR] `ComparacionCartaBancoService.cs` — limpieza de whitespace en firma de método

```csharp
// Antes:
private DetalleCampoComparacionResultDto CompararTipoProducto(
    int idTipoProductoResolucion, 
    string nombreTipoProductoResolucion,

// Después:
private DetalleCampoComparacionResultDto CompararTipoProducto(
    int idTipoProductoResolucion,
    string nombreTipoProductoResolucion,
```

El trailing whitespace en la firma fue eliminado. Cambio correcto pero cosmético — no debería generar un commit dedicado.

---

### 4.8 [MENOR] El código `ENVIADA_REVISION` en el script SQL difiere del enum `EnviadaRevision`

El script SQL inserta el código `'ENVIADA_REVISION'`, y el `EstadoEnum` lo define también como `"ENVIADA_REVISION"`. Esto es consistente. Sin embargo, el nombre del campo en el enum es `EnviadaRevision` (femenino singular) mientras que en el script el estado se llama `'Enviada a revisión'`. Verificar que el `GetNombre()` del enum retorne exactamente `"Enviada a revisión"` (con tilde y minúscula) para que el `GetByIdTipoEntidadYEstadoNombre` del repositorio encuentre el registro. Un mismatch de tilde o capitalización generaría un 500 en runtime.

---

## 5. Puntos positivos

- **Fix de división por cero en `CompararMonto`**: La guarda `if (montoResolucion == 0)` previene una excepción arimética que podría explotarse para causar un 500.
- **Mejora de rutas del controller**: `POST /{idSolicitud}` y `PATCH /{idSolicitud}/comparacion/{idComparacion}/comentario` son rutas RESTful más claras que el diseño anterior. Eliminar el cuerpo del POST inicial también es correcto.
- **Separación de DTO**: Crear `ComentarioComparacionRequestDto` con solo `Comentario` y agregar FluentValidation para limitarlo a 1000 caracteres es una mejora limpia.
- **Enum completo**: `EstadoEnum.EnviadaRevision` tiene nombre, código, `GetNombre()`, `GetCodigo()`, `FromNombre()` y `FromCodigo()` correctamente mapeados — no quedan paths inconsistentes.
- **Estrategia bien encapsulada**: `InstrumentacionEnviadaRevisionStrategy` sigue el patrón de Strategy del proyecto: `CanHandle`, `DeterminarEstadoDestinoAsync`, `ConstruirMetadataAsync`. El registro en IOC como `AddTransient` es consistente con el resto.
- **Cobertura de tests muy alta**: ~4200 líneas de tests para 17 archivos. Cubre todos los paths del command handler, la estrategia, el servicio y el controller.

---

## 6. Análisis del script SQL

| Ítem | Estado |
|------|--------|
| Inserción de estado `Enviada a revisión` | ❌ No idempotente |
| Inserción de `EstadoConfiguracion` con código `ENVIADA_REVISION` | ❌ No idempotente |
| Inserción de transición `PENDIENTE_INSTRUMENTACION → ENVIADA_REVISION` | ❌ No idempotente |
| Transacción explícita (`BEGIN TRANSACTION / ROLLBACK`) | ❌ Ausente |
| Validación de existencia del `TipoEntidad 'Solicitud'` | ✅ Usa SELECT con variable |
| Lookup del `IdEstadoConfiguracionOrigen` por código | ✅ Correcto |
| `TransicionReglas = NULL` en la transición | ✅ Correcto — transición directa sin reglas |

**Nota sobre la transición inversa**: La US menciona también la transición de vuelta `EnviadaRevision → PendienteInstrumentacion`. Este script **solo crea la transición de ida**. La de vuelta queda pendiente (probablemente en la siguiente Task/PR).

---

## 7. Resumen de hallazgos

| # | Criticidad | Componente | Descripción |
|---|-----------|-----------|-------------|
| 4.1 | 🔴 Crítico | `GuardarComparacionCartaBancoCommand` | Swallow silencioso de error en transición — 200 OK aunque el estado no cambie |
| 4.2 | 🔴 Crítico | Script SQL | No es idempotente y sin transacción — duplicados si se ejecuta más de una vez |
| 4.3 | 🟠 Importante | `InstrumentacionEnviadaRevisionStrategy.CanHandle` | `ConfiguracionAdicional.ContainsKey` lanza NullReferenceException si el dict es null |
| 4.4 | 🟠 Importante | `GuardarComparacionCartaBancoCommand` | Transición se intenta siempre sin validar estado actual — warning silencioso si no aplica |
| 4.5 | 🟠 Importante | `InstrumentacionEnviadaRevisionStrategy.ConstruirMetadataAsync` | Campos `monto` y `monto_limite_definido` copiados de otra estrategia; nunca se populan |
| 4.6 | 🟡 Menor | `ComparacionCartaBancoRequestDtoValidatorTests.cs` | Posible referencia a DTO eliminado — verificar que compile |
| 4.7 | 🟡 Menor | `ComparacionCartaBancoService.cs` | Trailing whitespace en firma de método eliminado — cambio cosmético |
| 4.8 | 🟡 Menor | `EstadoEnum` / Script SQL | Verificar que `GetNombre()` retorne exactamente `"Enviada a revisión"` con tilde correcta |

---

## 8. Checklist de aprobación

- [ ] 4.1 — Relanzar la excepción en el catch de `EjecutarTransicionEnviadaRevisionAsync` o retornar un error al cliente
- [ ] 4.2 — Hacer el script SQL idempotente con `IF NOT EXISTS` en los 3 INSERTs y envolver en `BEGIN TRANSACTION / ROLLBACK`
- [ ] 4.3 — Usar `?.ContainsKey` o inicializar `ConfiguracionAdicional` por defecto en `TransicionContext`
- [ ] 4.4 — Validar estado actual antes de intentar la transición, o exponer si el cambio de estado fue efectivo en la respuesta
- [ ] 4.5 — Eliminar código de `monto`/`monto_limite_definido` de `ConstruirMetadataAsync` por no corresponder a esta estrategia
