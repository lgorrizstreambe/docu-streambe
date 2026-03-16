# PR Review — `feature/34525-Comparacion-Carta-Resolucion`

> **Base:** `release/3.0` → **Head:** `feature/34525-Comparacion-Carta-Resolucion`
> **US:** [34022] COR-OTG-CBC – Comparar carta banco vs resolución del acta del Consejo
> **Fecha de revisión:** 16/03/2026

---

## 1. Resumen ejecutivo

El PR implementa la comparación de campos entre la carta banco y la resolución del acta del Consejo. La estructura Clean Architecture + CQRS está bien aplicada, el servicio de comparación es completo y las tolerancias están desacopladas vía parámetros de sistema. Se identifican **dos issues críticos** (división por cero, ausencia total de unit tests) y varias observaciones importantes relacionadas con la naturaleza mock de los datos de resolución.

**Veredicto:** ⚠️ Aprueba con correcciones requeridas

---

## 2. Inventario de cambios

### Core.Domain

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core/Contracts/ICoreSchemaContext.cs` | Modificado | Agrega `IComparacionCartaBancoResolucionRepository` |
| `Core/Contracts/IComparacionCartaBancoResolucionRepository.cs` | Nuevo | `CreateAsync`, `GetByIdAsync`, `GetUltimaComparacionBySolicitudAsync`, `UpdateComentarioAsync`, `DesactivarComparacionesAnterioresAsync` |
| `Core/Contracts/IComparacionCartaBancoService.cs` | Nuevo | Interfaz del servicio de dominio (no de infraestructura) |
| `Core/Entities/ComparacionCartaBancoResolucion.cs` | Nuevo | Entidad con `IdSolicitud`, `ResultadoOk`, `DetalleJson`, `Comentario` (hereda `EntityBase`) |
| `Core/Dtos/ComparacionCartaBancoResolucionResultDto.cs` | Nuevo | DTO de resultado con `Ok`, `RequiereComentario`, `FechaVencimiento`, `Detalle` |
| `Core/Dtos/DetalleCampoComparacionResultDto.cs` | Nuevo (nested) | Campo, OK/NOK, valoresComparados, mensaje de error |
| `Core/Dtos/ComparacionCartaBancoConfigDto.cs` | Nuevo | Configuración con `Tolerancias` (dict) y `MesesVigenciaCartaBanco` |
| `Core/Dtos/ToleranciaConfigDto.cs` | Nuevo | `TipoComparacion` + `Tolerancia` (decimal?) por campo |
| `Core/Dtos/ResolucionConsejoMockDto.cs` | Nuevo | DTO de mock para datos de la resolución del Consejo |
| `Core/Options/ComparacionCartaBancoOptions.cs` | Nuevo | `UseMockData` (bool) — feature flag para mock de carta banco |
| `n8n/Internal/CartaBanco/CartaBancoMetadata.cs` | Nuevo | Estructura de metadata de la carta banco leída desde API.Archivos |
| `Parametrizaciones/Contracts/IParametrosSistemaRepository.cs` | Nuevo | `GetByCodigoAsync` — acceso a parámetros de sistema por código |
| `Parametrizaciones/Contracts/IParametrizacionesSchemaContext.cs` | Modificado | Agrega `IParametrosSistemaRepository ParametrosSistema` |
| `Parametrizaciones/Entities/ParametroSistema.cs` | Nuevo | Entidad con `Codigo`, `Valor`, `ParametrosJson` |

### Core.Application

| Archivo | Tipo | Descripción |
|---|---|---|
| `Commands/ComparacionCartaBancoResolucionCommand.cs` | Nuevo | Orquesta: desactiva comparación anterior → llama servicio → persiste resultado |
| `Commands/GuardarComparacionCartaBancoCommand.cs` | Nuevo | Actualiza SOLO el comentario de una comparación existente |
| `Dtos/Requests/ComparacionCartaBancoResolucionRequestDto.cs` | Nuevo | `IdSolicitud` (sin validator registrado — ver Obs. 5.3) |
| `Dtos/Requests/GuardarComparacionCartaBancoRequestDto.cs` | Nuevo | `IdSolicitud`, `IdComparacion`, `Comentario` |
| `Dtos/Responses/GuardarComparacionCartaBancoResponseDto.cs` | Nuevo | Respuesta del update del comentario |
| `Dtos/Validators/GuardarComparacionCartaBancoRequestDtoValidator.cs` | Nuevo | Valida `IdSolicitud > 0`, `IdComparacion > 0`, `Comentario` NotEmpty + MaxLength(1000) |
| `Dtos/Mappers/CoreMappingProfile.cs` | Modificado | Agrega `using Core.Domain.Core.Dtos` |

### Core.Infrastructure

| Archivo | Tipo | Descripción |
|---|---|---|
| `Repositories/Core/ComparacionCartaBancoResolucionRepository.cs` | Nuevo | SPs: `_Insert`, `_ById_Select`, `_UltimaBySolicitud_Select`, `_Update`, `_ByIdSolicitud_Delete`. Métodos IRepository no utilizados lanzan `NotImplementedException` |
| `Repositories/Parametrizaciones/ParametrosSistemaRepository.cs` | Nuevo | Solo implementa `GetByCodigoAsync`, resto son `NotImplementedException` |
| `Services/Core/ComparacionCartaBancoService.cs` | Nuevo | **560 líneas** — lógica principal: obtiene carta banco desde API.Archivos, mock de resolución, compara 9 campos con tolerancias |
| `Persistence/Schemas/CoreSchemaContext.cs` | Modificado | Lazy-init `ComparacionesCartaBancoResolucion` |
| `Persistence/Schemas/ParametrizacionesSchemaContext.cs` | Modificado | Lazy-init `ParametrosSistema` |

### Core.Presentation

| Archivo | Tipo | Descripción |
|---|---|---|
| `ComparacionCartaBancoController.cs` | Nuevo | `POST /` (comparar) y `PATCH /comentario` (actualizar comentario) |
| `IOCCoreSchema.cs` | Modificado | Registra options, service, 1 validator, 2 command handlers |
| `appsettings.Development.json` | Modificado | Agrega `"ComparacionCartaBanco": { "UseMockData": true }`. **Reformatea todo el archivo** de 4 a 2 espacios (ruido en diff) |

**Archivos nuevos/modificados:** ~22

---

## 3. Cobertura de Criterios de Aceptación

| Criterio | Estado | Observación |
|---|---|---|
| **CA1** – Comparación exitosa → registra OK | ✅ | `ResultadoOk = true` cuando todos los campos están dentro de tolerancias |
| **CA2** – Comparación con observaciones → solicita comentario obligatorio | ✅ | `RequiereComentario = !Ok`; validator de `GuardarComparacionCartaBancoRequestDto` exige `Comentario` con `NotEmpty()` |
| **CA3** – Resultado asociado a la solicitud y disponible para consulta | ✅ | Persiste en `Core.ComparacionesCartaBancoResolucion`; existe `GetUltimaComparacionBySolicitudAsync` |
| **RN01** – Solo ejecuta si existe carta banco cargada | ✅ | `ObtenerDatosCartaBancoAsync` retorna null si no existe → `InvalidOperationException` |
| **RN02** – No modifica datos de solicitud ni contrato | ✅ | El command solo inserta en `ComparacionesCartaBancoResolucion` |
| **RN03** – Resultado Observado requiere comentario | ✅ | Endpoint `PATCH /comentario` + validator |
| **RN04** – No bloquea el flujo automáticamente | ✅ | Sin transición de estado, sin validación de `EsCompleto` en ningún otro endpoint |

### Cobertura de campos a comparar (US)

| Campo | Implementado | Observación |
|---|---|---|
| CUIT | ✅ | `CompararIgualdadExacta` — compara solicitud vs carta banco |
| Razón Social | ✅ | `CompararIgualdadExacta` — compara solicitud vs carta banco |
| Entidad acreedora | ✅ | Exacta — **fuente "resolución" es MOCK** (ver Obs. 5.1) |
| Monto | ✅ | ±10% por defecto, configurable. **División por cero si monto=0** (ver Obs. 5.2) |
| Moneda | ✅ | Exacta — **fuente "resolución" es MOCK** |
| Plazo | ✅ | ±1 período por defecto, configurable — **fuente "resolución" es MOCK** |
| Tasa | ✅ | ±1 punto por defecto, configurable — **fuente "resolución" es MOCK** |
| Tipo de producto / línea | ✅ (informativo) | Siempre `Ok=true` con mensaje opcional — **fuente "resolución" es MOCK** |

---

## 4. Puntos positivos

- **Tolerancias configurables desde DB.** `ParametrosSistema` con código `COMPARACION_CARTA_BANCO_RESOLUCION_CONSEJO` y JSON permite ajustar tolerancias y vigencia sin deploy. El fallback a valores por defecto está bien documentado.
- **Idempotencia de comparaciones.** Antes de guardar, desactiva (soft-delete) la comparación activa anterior. Permite re-ejecutar la comparación limpiamente.
- **Feature flag `UseMockData`.** El flag en `ComparacionCartaBancoOptions` aísla correctamente el comportamiento mock. En producción bastará con no configurar la sección (default `false`).
- **Separación clara de responsabilidades.** `IComparacionCartaBancoService` en Domain + `ComparacionCartaBancoService` en Infrastructure — el command es un orquestador delgado.
- **`RequiereComentario` calculado automáticamente.** La propiedad `RequiereComentario => !Ok` en el DTO evita lógica duplicada en el frontend.
- **5 escenarios de mock con variabilidad.** Los escenarios de mock basados en `IdSolicitud % 5` permiten testear distintos casos sin fixtures complejos.
- **Vigencia calculada o explícita.** `ObtenerFechaVencimiento` prioriza `FechaVencimiento` explícita y calcula desde `FechaEmision + MesesVigencia` como fallback.

---

## 5. Observaciones / Issues

### 5.1 🔴 Datos de la resolución son MOCK — la comparación no valida contra la resolución real del Consejo

**Severidad: Alta — la feature está incompleta desde el punto de vista funcional**

El método `ObtenerResolucionMock` devuelve directamente los datos de la **solicitud** como si fueran los de la resolución del Consejo:

```csharp
private ResolucionConsejoMockDto ObtenerResolucionMock(SolicitudCompletaDbRow solicitud)
{
    return new ResolucionConsejoMockDto
    {
        EntidadAcreedora = solicitud.NombreEntidadFinanciera,
        Monto = solicitud.MontoSolicitado,      // ← dato de solicitud, no de resolución
        Moneda = solicitud.NombreMoneda,
        Plazo = solicitud.PlazoMeses,
        ...
    };
}
```

Esto implica que **5 de los 9 campos siempre van a comparar igual** (la solicitud contra sí misma). El resultado del conjunto solo variará en la comparación de carta banco vs solicitud, no vs resolución del Consejo.

Este comportamiento no está señalizado en el código del command ni en la documentación Swagger, lo que puede llevar a malentendidos en QA.

**Recomendación:** Agregar un `// TODO: reemplazar cuando exista la tabla de resoluciones` explícito en el command (además del que ya existe en el service), y considerar incluir un campo `EsResolucionMock: bool` en el response DTO para que el frontend lo indique visualmente.

---

### 5.2 🔴 División por cero en `CompararMonto` cuando `montoResolucion == 0`

**Severidad: Alta — lanza `DivideByZeroException` en runtime**

```csharp
var diferenciaPorcentaje = Math.Abs((montoCartaBanco - montoResolucion) / montoResolucion * 100);
```

Si `montoResolucion` es `0`, la división lanza excepción. Si bien es improbable en producción, un test con datos mock podría llegar a este caso y derribar el request sin un error útil.

**Fix:**
```csharp
if (montoResolucion == 0)
{
    detalle.Ok = false;
    detalle.MensajeError = "Monto de la resolución es 0, no se puede calcular tolerancia porcentual";
    return detalle;
}
var diferenciaPorcentaje = Math.Abs((montoCartaBanco - montoResolucion) / montoResolucion * 100);
```

---

### 5.3 ⚠️ Sin validator para `ComparacionCartaBancoResolucionRequestDto`

El endpoint `POST /` acepta `[FromBody] ComparacionCartaBancoResolucionRequestDto dto` pero no hay `IValidator<ComparacionCartaBancoResolucionRequestDto>` registrado en DI. Un `IdSolicitud = 0` llega al handler y falla con un error opaco desde el repositorio.

```csharp
// IOCCoreSchema.cs — solo registra el validator de Guardar, falta el de Comparar
builder.Services.AddTransient<IValidator<GuardarComparacionCartaBancoRequestDto>, GuardarComparacionCartaBancoRequestDtoValidator>();
// ← falta: AddTransient<IValidator<ComparacionCartaBancoResolucionRequestDto>, ...>
```

**Fix:** Crear `ComparacionCartaBancoResolucionRequestDtoValidator` con `RuleFor(x => x.IdSolicitud).GreaterThan(0)`.

---

### 5.4 ⚠️ Ausencia total de unit tests

El PR no incluye ningún test. La lógica de comparación en `ComparacionCartaBancoService` tiene múltiples ramas no triviales:

- Tolerancia de monto (%, división por cero)
- Tolerancia de plazo (períodos)
- Tolerancia de tasa (puntos porcentuales, casos null)
- Comparación exacta de cadenas (case-insensitive, trim)
- Cálculo de fecha de vencimiento (3 ramas)
- 5 escenarios de mock

Estas son exactamente las funciones que más se benefician de unit tests parametrizados (`[Theory]`).

**Recomendación:** Al menos cubrir:
- `CompararMonto` — casos: dentro de tolerancia, fuera, exactamente en límite, monto=0
- `CompararTasa` — casos: ambas null, solo una null, dentro/fuera de tolerancia
- `ObtenerFechaVencimiento` — 3 ramas (explícita, calculada, sin datos)
- `CompararIgualdadExacta` — exacto, case-insensitive, trim, uno vacío, ambos vacíos

---

### 5.5 ⚠️ `int.Parse` sin manejo de errores en los command handlers

Ambos command handlers usan `int.Parse(_userService.CurrentUserId)`. El resto del proyecto usa `Convert.ToInt32(...)`.

```csharp
// Patrón en este PR (lanza FormatException si hay null o valor no numérico)
var idUsuario = int.Parse(_userService.CurrentUserId);

// Patrón del resto del proyecto
int idUsuario = Convert.ToInt32(_userService.CurrentUserId);
```

**Fix:** Reemplazar `int.Parse` por `Convert.ToInt32` para consistencia y manejo de null.

---

### 5.6 ⚠️ `TipoProducto` y `LineaProducto` siempre retornan `Ok = true` con posible `MensajeError`

El campo tiene `Ok = true` aunque la comparación no coincida, pero puede tener `MensajeError != null`. Esto puede causar confusión en el frontend que podría esperar que `Ok=false` cuando hay un mensaje de error:

```csharp
detalle.Ok = true;   // ← siempre true
if (!coincide)
    detalle.MensajeError = "El tipo de producto difiere (informativo)";  // ← pero hay mensaje
```

**Recomendación:** Si la intención es que sea informativo y no excluyente, renombrar el campo `MensajeError` a `MensajeInformativo` en estos dos casos para diferenciarlos visualmente en el frontend.

---

### 5.7 ℹ️ Reformateo completo de `appsettings.Development.json`

El archivo cambió de indentación 4 espacios a 2 espacios en su totalidad. Esto genera ruido en el diff y puede ocultar cambios reales. Solo el bloque `ComparacionCartaBanco` es funcional.

---

### 5.8 ℹ️ Endpoints usan `[FromBody]` con `IdSolicitud` en lugar de `[FromRoute]`

El patrón del resto de controllers (p.ej. `SolicitudesGarantiaController`, `DocumentosSolicitudesController`) usa `{idSolicitud}` como segmento de ruta. Aquí ambos endpoints reciben `IdSolicitud` en el body, lo que es inconsistente:

```csharp
// Patrón del proyecto
[HttpPost("{idSolicitud}/solicitar-facturacion")]
public async Task<...> (..., [FromRoute] int idSolicitud)

// Este PR
[HttpPost]
public async Task<...> CompararCartaBancoConResolucion([FromBody] ComparacionCartaBancoResolucionRequestDto dto)
```

No bloquea, pero dificulta el logging, el rate limiting por solicitud y la navegación con Swagger.

---

### 5.9 ℹ️ `NotImplementedException` en métodos de IRepository no utilizados

`ComparacionCartaBancoResolucionRepository` y `ParametrosSistemaRepository` implementan `IRepository<T>` que exige `CreateAsync`, `DeleteAsync`, `GetAllAsync`, `UpdateAsync`, `GetByIdAsync`. Los métodos no necesarios lanzan `NotImplementedException`. Es la práctica habitual del proyecto, pero conviene no lanzar excepciones en métodos que podrían ser llamados por reflexión en testing.

---

## 6. SPs esperados en base de datos

| SP | Acción |
|---|---|
| `Core.ComparacionesCartaBancoResolucion_Insert` | Inserta resultado de comparación |
| `Core.ComparacionesCartaBancoResolucion_ById_Select` | Obtiene comparación por ID |
| `Core.ComparacionesCartaBancoResolucion_UltimaBySolicitud_Select` | Obtiene última comparación activa por solicitud |
| `Core.ComparacionesCartaBancoResolucion_Update` | Actualiza comentario |
| `Core.ComparacionesCartaBancoResolucion_ByIdSolicitud_Delete` | Soft-delete de comparaciones anteriores por solicitud |
| `Parametrizaciones.ParametrosSistema_ByCodigo_Select` | Obtiene parámetro por código (tolerancias) |

Además: el parámetro de sistema con codigo `COMPARACION_CARTA_BANCO_RESOLUCION_CONSEJO` debe existir en la tabla con el JSON de configuración de tolerancias.

---

## 7. Checklist de revisión

| Criterio | Estado |
|---|---|
| Sigue Clean Architecture | ✅ |
| Sigue CQRS | ✅ |
| Naming conventions | ✅ |
| Sin SQL inline (todo vía SPs) | ✅ |
| Validator registrado para endpoint comparar | 🔴 (Obs. 5.3) |
| Validator de guardar comentario | ✅ |
| Handlers registrados en DI | ✅ |
| Division por cero en CompararMonto | 🔴 (Obs. 5.2) |
| Datos de resolución (no mock real) | ⚠️ (Obs. 5.1 — por diseño, pero debe señalizarse) |
| Unit tests | 🔴 (Obs. 5.4 — ausentes) |
| Feature flag para mock | ✅ |
| Tolerancias configurables desde DB | ✅ |
| Idempotencia de comparaciones | ✅ |
| HTTP codes correctos | ✅ |
| Consistencia de ruta vs body | ⚠️ (Obs. 5.8) |
| Scripts SQL de SPs y parámetro de sistema | ❓ (no visible en diff) |

---

## 8. Conclusión

La implementación cubre correctamente la interfaz de comparación, la persistencia del resultado y el flujo de comentario obligatorio. Las tolerancias desacopladas vía `ParametrosSistema` son una decisión de diseño sólida.

Los puntos a resolver antes del merge son:

1. **Obs. 5.2 — División por cero en `CompararMonto`:** Fix de una línea, riesgo real en runtime.
2. **Obs. 5.3 — Falta validator para `ComparacionCartaBancoResolucionRequestDto`:** Permite `IdSolicitud = 0` al handler.
3. **Obs. 5.4 — Sin unit tests:** La lógica de comparación con tolerancias es el corazón de la feature y el código más propenso a regresiones junto con los edge cases de tasa y fechas.
4. **Obs. 5.1 — Señalizar explícitamente que la resolución es mock:** Agregar indicador en response para evitar malentendidos en QA y frontend.
