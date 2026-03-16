# PR Review — `feature/34572` → `release/3.0`

**US:** 34162 — COR-OTG-ACT — Guardado final del Acta del Consejo  
**Rama fuente:** `feature/34572`  
**Rama destino:** `release/3.0`  
**Revisado por:** GitHub Copilot (Claude Sonnet 4.6)  
**Fecha:** 2025-07-18

---

## 1. Resumen ejecutivo

El PR implementa el endpoint `POST /actas-consejo/{id}/final` que finaliza un Acta del Consejo en estado Borrador: cambia el estado del acta a "Finalizado" via el `TransicionEstadoOrchestrator`, transiciona cada solicitud incluida al estado destino que ya tenía asignado, descarga el PDF previamente generado y retorna el acta completa con el archivo adjunto.

Incluye además la primera versión del feature de **Comparación Carta Banco vs. Resolución del Consejo** (commands, repositorio, servicio), que pertenece en rigor a US 34022 (COR-OTG-CBC) y estaba en la rama `feature/34525`.

La cobertura de tests unitarios es amplia (1861 líneas para el handler + 596 para la estrategia + tests de controller). La arquitectura sigue los patrones del proyecto. Se identifican un bug de formato de hora, ausencia de transacción de BD y varias mejoras.

---

## 2. Archivos modificados / creados

### Feature principal — US 34162 (ActaConsejoFinalizar)
| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.Application/Modules/Core/Commands/ActaConsejoFinalizarCommand.cs` | NUEVO | Command handler principal (288 líneas) |
| `Core.Application/Modules/Helper/Transiciones/Strategies/ActasConsejo/ActaConsejoBorradorAFinalizadoStrategy.cs` | NUEVO | Estrategia de transición Borrador → Finalizado |
| `Core.Application/Modules/Helper/EstadoTransicionHelper.cs` | MOD | Agrega `TipoEntidadEnum.ActaConsejo → FlujoEnum.ActaConsejo` |
| `Core.Application/Modules/Core/Dtos/Mappers/CoreMappingProfile.cs` | MOD | Agrega mapeos AutoMapper para Acta, Solicitudes e Integrantes |
| `Core.Application/Modules/Core/Dtos/Responses/ActaConsejoCompletaDto.cs` | MOD | Agrega `ArchivoActaDto` y campo `Archivo` al DTO raíz |
| `Core.Domain/GestionFlujos/Enums/FlujoEnum.cs` | MOD | Agrega `ActaConsejo = 5` |
| `Core.Domain/GestionFlujos/Enums/PasoEnum.cs` | MOD | Agrega `FormularioAltaActaConsejo` |
| `Core.Presentation/Api/Core/Controllers/v1/ActasConsejoController.cs` | MOD | Agrega endpoint `POST {id}/final` |
| `Core.Presentation/Api/Extensions/IOCCoreSchema.cs` | MOD | Registra handler, estrategia y servicio |
| `Core.UnitTesting/Application/Core/Commands/ActaConsejoFinalizarCommandHandlerTests.cs` | NUEVO | 1861 líneas de tests unitarios |
| `Core.UnitTesting/Application/Helper/.../ActaConsejoBorradorAFinalizadoStrategyTests.cs` | NUEVO | 596 líneas de tests de la estrategia |
| `Core.UnitTesting/Presentation/Core/ActasConsejoControllerTests.cs` | MOD | Agrega tests del endpoint `FinalizarActa` |

### Feature incidental — US 34022 (ComparacionCartaBanco)
| Archivo | Tipo |
|---|---|
| `Core.Application/Modules/Core/Commands/ComparacionCartaBancoResolucionCommand.cs` | NUEVO |
| `Core.Application/Modules/Core/Commands/GuardarComparacionCartaBancoCommand.cs` | NUEVO |
| `Core.Application/Modules/Core/Dtos/Requests/ComparacionCartaBancoResolucionRequestDto.cs` | NUEVO |
| `Core.Application/Modules/Core/Dtos/Requests/GuardarComparacionCartaBancoRequestDto.cs` | NUEVO |
| `Core.Application/Modules/Core/Dtos/Responses/GuardarComparacionCartaBancoResponseDto.cs` | NUEVO |
| `Core.Application/Modules/Core/Dtos/Validators/GuardarComparacionCartaBancoRequestDtoValidator.cs` | NUEVO |
| `Core.Domain/Core/Contracts/IComparacionCartaBancoResolucionRepository.cs` | NUEVO |
| `Core.Domain/Core/Contracts/IComparacionCartaBancoService.cs` | NUEVO |
| `Core.Domain/Core/Contracts/ICoreSchemaContext.cs` | MOD |
| `Core.Domain/Core/Dtos/ComparacionCartaBanco*.cs` (×3) | NUEVO |
| `Core.Domain/Core/Entities/ComparacionCartaBancoResolucion.cs` | NUEVO |
| `Core.Domain/n8n/Internal/CartaBanco/CartaBancoMetadata.cs` | NUEVO |
| `Core.Domain/Options/ComparacionCartaBancoOptions.cs` | NUEVO |
| `Core.Domain/Parametrizaciones/Contracts/IParametrizacionesSchemaContext.cs` | MOD |
| `Core.Domain/Parametrizaciones/Contracts/IParametrosSistemaRepository.cs` | NUEVO |
| `Core.Domain/Parametrizaciones/Entities/ParametroSistema.cs` | NUEVO |
| `Core.Infraestructure/Persistence/Repositories/Core/ComparacionCartaBancoResolucionRepository.cs` | NUEVO |
| `Core.Infraestructure/Persistence/Repositories/Parametrizaciones/ParametrosSistemaRepository.cs` | NUEVO |
| `Core.Infraestructure/Persistence/Schemas/CoreSchemaContext.cs` | MOD |
| `Core.Infraestructure/Persistence/Schemas/ParametrizacionesSchemaContext.cs` | MOD |
| `Core.Infraestructure/Services/Core/ComparacionCartaBancoService.cs` | NUEVO (560 líneas) |
| `Core.Presentation/Api/Core/Controllers/v1/ComparacionCartaBancoController.cs` | NUEVO |
| `Core.Presentation/appsettings.Development.json` | MOD (reformatting + nueva sección) |

---

## 3. Flujo implementado — ActaConsejoFinalizarCommand

```
POST /actas-consejo/{id}/final
  │
  ├─ 1. GetByIdCompletaAsync(idActa)           → SP ActasConsejo_ByIdCompleta_Select
  ├─ ValidarActaParaFinalizar()                → chequea Borrador + IdArchivo notNull
  ├─ 4. GetEstadoFinalizado(TipoEntidad=Acta)
  ├─ 5. GetEstadoOrigen(actaExistente.IdEstado)
  ├─ 6. orchestrator.EjecutarTransicionAsync(acta→Finalizado)
  ├─ 7. UpdateAsync(acta, nuevoEstado)         → SP ActasConsejo_Update
  │
  ├─ 8. foreach solicitud en acta
  │      ├─ GetByIdAsync(solicitud)
  │      ├─ GetByIdAsync(empresa)
  │      ├─ FromCodigo(EstadoDestino).GetNombre()  → lookup dinámico
  │      ├─ GetEstadoDestinoConfig(Solicitud, estadoNombre)
  │      ├─ GetEstadoOrigenConfig(solicitud.IdEstado)
  │      ├─ orchestrator.EjecutarTransicionAsync(solicitud→estadoDestino)
  │      ├─ [si PENDIENTE_APROBACION_CONSEJO] DeleteAsync(actaConsejoSolicitud)
  │      └─ UpdateAsync(solicitud, nuevoEstado)
  │
  ├─ 9.  GetByIdCompletaAsync(idActa)           → segunda lectura (estado final)
  ├─ 10. archivosService.GetCabeceraAsync + DownloadAsync
  ├─ 11. mapper.Map<ActaConsejoCompletaDto>(acta)
  └─ return resultado
```

---

## 4. Aspectos positivos

- **Patrón CQRS** bien aplicado: el command es `sealed record`, el handler llama al orchestrator que ya maneja el registro de historial de transiciones.  
- **Separación de responsabilidades**: la validación pre-transición está en `ValidarActaParaFinalizar`, la lógica de estado en `ActaConsejoBorradorAFinalizadoStrategy` y el procesamiento masivo en el handler.  
- **Lógica de eliminación selectiva de la relación** `ActaConsejoSolicitud` cuando el estado destino es `PENDIENTE_APROBACION_CONSEJO` es conceptualmente correcta: la solicitud puede quedar libre para ser incluida en otra acta futura.  
- **`FromCodigo`** mapea el código string almacenado en la tabla al `EstadoEnum` tipado antes de llamar a `GetNombre()`, evitando strings mágicos hardcodeados en el handler.  
- **Tests unitarios**: cobertura alta sobre el flujo feliz y la mayoría de los caminos de error (NotFound, BadRequest, InternalServerError, eliminación selectiva de relación, mapeo de estados destino).  
- **DI**: las tres registraciones nuevas en `IOCCoreSchema.cs` son correctas y están comentadas por grupo.

---

## 5. Observaciones

### 5.1 🔴 CRÍTICO — Sin transacción de base de datos

El handler actualiza el estado del acta (paso 7) y luego, en un loop, actualiza N solicitudes (paso 8) sin envolver el conjunto en una transacción. Si falla la transición de la solicitud Nº K, el acta ya tiene estado "Finalizado" en la BD pero las solicitudes 0…K-1 ya procesadas no se revierten.

**Archivo:** `Core.Application/Modules/Core/Commands/ActaConsejoFinalizarCommand.cs`

```csharp
// Paso 6-7: transiciona el acta y la actualiza
var resultTransicion = await _orchestrator.EjecutarTransicionAsync(context, cancellationToken);
// ...
await uow.Core.ActasConsejo.UpdateAsync(actaExistente, idUsuarioActual);

// Paso 8: transiciona cada solicitud — SIN transacción
await ProcesarTransicionSolicitudes(uow, idUsuarioActual, actaExistente, cancellationToken);
```

**Corrección sugerida:** Envolver los pasos 6-8 en una transacción de BD, o realizar todas las escrituras dentro del mismo `UnitOfWork` que soporte `BeginTransactionAsync` / `CommitAsync` / `RollbackAsync`.

---

### 5.2 🔴 BUG — Formato de hora 12h en lugar de 24h en el mapper

En `CoreMappingProfile.cs`, el formato `"hh\\:mm"` usa el especificador de 12 horas. Para un acta con hora `14:30`, el mapper produciría `"02:30"` en vez de `"14:30"`.

**Archivo:** `Core.Application/Modules/Core/Dtos/Mappers/CoreMappingProfile.cs`

```csharp
// INCORRECTO — hh es 12h
.ForMember(dest => dest.HoraInicioSesion, opt => opt.MapFrom(src =>
    src.HoraInicioSesion != null ? src.HoraInicioSesion.Value.ToString(@"hh\:mm") : null))
.ForMember(dest => dest.HoraFinalizacion, opt => opt.MapFrom(src =>
    src.HoraFinalizacion != null ? src.HoraFinalizacion.Value.ToString(@"hh\:mm") : null))
```

**Corrección:**

```csharp
// CORRECTO — HH es 24h (00-23)
.ForMember(dest => dest.HoraInicioSesion, opt => opt.MapFrom(src =>
    src.HoraInicioSesion != null ? src.HoraInicioSesion.Value.ToString(@"HH\:mm") : null))
.ForMember(dest => dest.HoraFinalizacion, opt => opt.MapFrom(src =>
    src.HoraFinalizacion != null ? src.HoraFinalizacion.Value.ToString(@"HH\:mm") : null))
```

---

### 5.3 🟡 IMPORTANTE — `ValidarActaParaFinalizar` no verifica campos obligatorios (RN03)

Según la US, los campos obligatorios para poder finalizar un acta son: N° Acta, Fecha de Sesión, Hora de Inicio, al menos un integrante del consejo y el redactor (Confeccionó). El método actual solo verifica que el estado sea Borrador y que exista un PDF.

**Archivo:** `Core.Application/Modules/Core/Commands/ActaConsejoFinalizarCommand.cs`

```csharp
private void ValidarActaParaFinalizar(Domain.Core.Entities.ActaConsejo? actaExistente)
{
    if (actaExistente == null)
        throw new CustomException(HttpStatusCode.NotFound, "Acta no encontrada.");

    // Verifica estado Borrador ✅
    // Verifica IdArchivo ✅

    // FALTAN: validaciones de RN03
    // - actaExistente.NumeroActa == null → BadRequest
    // - actaExistente.FechaSesion == null → BadRequest
    // - actaExistente.HoraInicioSesion == null → BadRequest
    // - actaExistente.IdRedactorActa == null → BadRequest
    // - actaExistente.Integrantes.Count == 0 → BadRequest
}
```

Sin estas validaciones, un acta en Borrador sin Número puede ser finalizada y registrada en BD en estado Finalizado con datos incompletos.

---

### 5.4 🟡 IMPORTANTE — Solicitudes con `EstadoDestino` nulo o código desconocido

Si una `ActaConsejoSolicitud` tiene `EstadoDestino = null` (o un código que `FromCodigo` no reconoce), el flujo produce:

1. `FromCodigo(null)` → `null`  
2. `null?.GetNombre()` → `null`  
3. `GetByIdTipoEntidadYEstadoNombre(Solicitud, null)` → retorna `null` (depende del SP)  
4. Excepción: `"No se encontró configuración de estado 'null' para Solicitud X."`

El error es confuso y ocurre a mitad del procesamiento (con el acta ya en estado Finalizado si no hay transacción).

**Corrección sugerida:** Validar antes del loop que todas las solicitudes tienen `EstadoDestino` con un código reconocido, o al menos mejorar el mensaje de error:

```csharp
if (string.IsNullOrWhiteSpace(actaSolicitud.EstadoDestino))
{
    throw new CustomException(
        HttpStatusCode.UnprocessableEntity,
        $"La solicitud {actaSolicitud.IdSolicitud} no tiene Estado Destino asignado. " +
         "Asigne el resultado antes de finalizar el acta.");
}

var estadoDestinoEnum = EstadoExtensions.FromCodigo(actaSolicitud.EstadoDestino);
if (estadoDestinoEnum == null)
{
    throw new CustomException(
        HttpStatusCode.UnprocessableEntity,
        $"El código de Estado Destino '{actaSolicitud.EstadoDestino}' de la solicitud " +
        $"{actaSolicitud.IdSolicitud} no es válido.");
}
```

---

### 5.5 🟡 IMPORTANTE — `GuardarComparacionCartaBancoResponseDto.FechaComparacion` nunca se asigna

El DTO de respuesta de `GuardarComparacionCartaBancoCommand` incluye:

```csharp
public class GuardarComparacionCartaBancoResponseDto
{
    // ...
    public DateTime FechaComparacion { get; set; }  // ← valor por defecto: DateTime.MinValue
}
```

El handler construye el response sin asignar `FechaComparacion`, por lo que el cliente recibe `0001-01-01T00:00:00`.

**Corrección:** Leer `FechaAlta` de `comparacionExistente` (viene de `EntityBase`), o asignarlo al momento de update, o eliminarlo del DTO si no es necesario.

```csharp
return new GuardarComparacionCartaBancoResponseDto
{
    IdComparacionCartaBancoResolucion = dto.IdComparacion,
    IdSolicitud = dto.IdSolicitud,
    ResultadoOk = comparacionExistente.ResultadoOk,
    Comentario = dto.Comentario,
    FechaComparacion = comparacionExistente.FechaAlta  // ← agregar
};
```

---

### 5.6 🟠 MEJORA — Mensaje de error confuso cuando `idArchivo` es null en `ObtenerArchivoPdfAsync`

```csharp
private async Task<ArchivoActaDto?> ObtenerArchivoPdfAsync(int? idArchivo, CancellationToken cancellationToken)
{
    if (!idArchivo.HasValue)
    {
        throw new CustomException(
            HttpStatusCode.InternalServerError,
            $"Error al obtener el archivo {idArchivo} del acta.");
            // ↑ idArchivo es null en este punto → mensaje: "Error al obtener el archivo  del acta."
    }
```

Dado que la validación `if (!actaExistente.IdArchivo.HasValue)` ya ocurrió en `ValidarActaParaFinalizar`, este guard debería ser tecnicamente irreachable. Pero si alguna vez se llega aquí, el mensaje es confuso.

**Corrección:**
```csharp
$"El acta DEBE tener un archivo PDF asociado antes de ser finalizada."
```

---

### 5.7 🟠 MEJORA — `IdTipoSocio = TipoSocioEnum.Protector` hardcodeado en el contexto para ActaConsejo

El `TransicionContext` del Acta tiene:
```csharp
IdTipoSocio = TipoSocioEnum.Protector,
```
El foco del orchestrator y de las estrategias de Empresas/Solicitudes usa `IdTipoSocio` para seleccionar el flujo GestionFlujos correcto (Protector vs Partícipe). Para ActaConsejo este valor no aplica conceptualmente. Si en el futuro alguna estrategia downstream usa `IdTipoSocio` para filtrar, podría producir comportamientos inesperados.

**Sugerencia:** Usar un valor neutral (ej: `TipoSocioEnum.Protector` está bien si el flujo `ACTA_CONSEJO` no depende del tipo de socio) pero documentarlo con un comentario explícito:

```csharp
IdTipoSocio = TipoSocioEnum.Protector, // No aplica para Actas; valor requerido por la firma del contexto
```

---

### 5.8 🟠 MEJORA — N+4 queries a BD por solicitud (sin batching)

Para cada `ActaConsejoSolicitud` en el acta el handler ejecuta secuencialmente:
1. `Solicitudes.GetByIdAsync`
2. `Empresas.GetByIdAsync`
3. `EstadosConfiguraciones.GetByIdTipoEntidadYEstadoNombre` (destino)
4. `EstadosConfiguraciones.GetByIdAsync` (origen)
5. `orchestrator.EjecutarTransicionAsync`
6. `ActasConsejoSolicitudes.DeleteAsync` (condicional)
7. `Solicitudes.UpdateAsync`

Para un acta con 30 solicitudes, esto implica ~200 operaciones de BD en serie. Si bien este escenario es funcional, en producción con actas grandes podría generar timeouts perceptibles.

**Sugerencia:** Pre-cargar los estados de configuración antes del loop (los destinos son finitos: Aprobado, Rechazado, Pendiente), y las empresas ya podrían estar en un dictionary por `IdEmpresa`.

---

### 5.9 🟠 OBSERVACIÓN — El feature de ComparacionCartaBancoResolucion pertenece a US 34022, no a este PR

El diff incluye los commands `ComparacionCartaBancoResolucionCommand`, `GuardarComparacionCartaBancoCommand`, el controlador `ComparacionCartaBancoController`, el servicio `ComparacionCartaBancoService` (560 líneas) y toda la infraestructura de `ParametrosSistema` y `ComparacionCartaBancoResolucion`. Estos artefactos corresponden al US 34022 (COR-OTG-CBC), que ya fue revisado en `feature/34525`.

Tener ambas features en la misma rama dificulta el rollback independiente y hace más difícil la revisión de cambios.

---

### 5.10 🟢 MENOR — `ActaConsejoBorradorAFinalizadoStrategy` hardcodea metadata como `true`

```csharp
var metadata = new MetadataTransicion
{
    documentacion_completa = true, // El acta tiene PDF asociado
    campos_obligatorios_completos = true // Validado antes de llamar a la estrategia
};
```

El comentario dice "validado antes de llamar a la estrategia" pero —ver 5.3— en realidad los campos obligatorios RN03 NO se validan. La metadata registrada en el historial de transiciones indicaría que todo es correcto cuando potencialmente no lo es.

---

## 6. Resumen de observaciones

| # | Severidad | Área | Descripción |
|---|---|---|---|
| 5.1 | 🔴 CRÍTICO | Command Handler | Sin transacción de BD — estado inconsistente si falla una solicitud |
| 5.2 | 🔴 BUG | Mapper | Formato hora `hh` (12h) en lugar de `HH` (24h): tardes como "02:30" |
| 5.3 | 🟡 IMPORTANTE | Command Handler | No valida campos obligatorios RN03 antes de finalizar |
| 5.4 | 🟡 IMPORTANTE | Command Handler | Solicitud con `EstadoDestino` null/inválido produce error en medio del proceso |
| 5.5 | 🟡 IMPORTANTE | Response DTO | `FechaComparacion` en `GuardarComparacionCartaBancoResponseDto` nunca se asigna |
| 5.6 | 🟠 MEJORA | Command Handler | Mensaje confuso cuando `idArchivo` es null (valor nulo en interpolación) |
| 5.7 | 🟠 MEJORA | Command Handler | `IdTipoSocio = Protector` hardcodeado sin documentación para ActaConsejo |
| 5.8 | 🟠 MEJORA | Performance | N+4 queries por solicitud; no hay batching para lotes grandes |
| 5.9 | 🟠 OBSERVACIÓN | Alcance | Feature COR-OTG-CBC (US 34022) incluida en este PR; debería estar separada |
| 5.10 | 🟢 MENOR | Estrategia | Metadata hardcodeada `true` aunque no se validan todos los campos RN03 |

---

## 7. Leyenda de severidades

| Ícono | Nivel | Descripción |
|---|---|---|
| 🔴 CRÍTICO | Bloquea el merge | Puede causar inconsistencia de datos o bug reproducible en producción |
| 🟡 IMPORTANTE | Debería corregirse antes del merge | Lógica de negocio incompleta o bug de presentación |
| 🟠 MEJORA | Recomendable | Calidad de código, performance o claridad; no bloquea |
| 🟢 MENOR | Opcional | Cosmético, nomenclatura o deuda técnica baja |
