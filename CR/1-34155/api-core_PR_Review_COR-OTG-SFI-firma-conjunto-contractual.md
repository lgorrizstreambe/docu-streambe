# PR Review — `COR-OTG-SFI-firma-conjunto-contractual`

> **Base:** `release/3.0` → **Head:** `COR-OTG-SFI-firma-conjunto-contractual`
> **US:** Solicitar y registrar firma de documentos contractuales
> **Fecha de revisión:** 16/03/2026

---

## 1. Resumen ejecutivo

El PR implementa el backend completo para la gestión del ciclo de firma del conjunto contractual de una solicitud. La implementación está bien estructurada, respeta la arquitectura del proyecto (Clean Architecture + CQRS) y cubre la mayor parte de los criterios de aceptación de la US. Se identifican algunos puntos menores que merecen atención antes del merge.

**Veredicto:** ✅ Aprobado con observaciones menores

---

## 2. Inventario de cambios

### Core.Domain

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core/Contracts/ICoreSchemaContext.cs` | Modificado | Agrega `IDocumentosSolicitudRepository` y `IFirmasDocumentosSolicitudRepository` |
| `Core/Contracts/IDocumentosSolicitudRepository.cs` | Nuevo | Interfaz del repositorio de documentos de solicitud (Insert, GetBySolicitud, GetByIdAndSolicitud, Delete) |
| `Core/Contracts/IFirmasDocumentosSolicitudRepository.cs` | Nuevo | Interfaz del repositorio de firmas (Insert, Gets, SolicitarFirma, RegistrarFirma, GetEstadoConjunto, InicializarConjunto) |
| `Core/Entities/DocumentoSolicitud.cs` | Nuevo | Entidad de dominio `DocumentoSolicitud` con constructor completo y campo nullable `IdDocumentoSolicitudReemplazado` |
| `Core/Entities/FirmaDocumentoSolicitud.cs` | Nuevo | Entidad de dominio `FirmaDocumentoSolicitud` con auditoría de firma (fecha, usuario, archivo) |
| `Core/Entities/ConjuntoContractualEstado.cs` | Nuevo | Entidad de dominio con `EsCompleto` como propiedad computada (`TotalDocumentos > 0 && TotalDocumentos == DocumentosFirmados`) |
| `Core/Entities/DbRows/DocumentoSolicitudDbRow.cs` | Nuevo | DbRow plano para Dapper con JOIN a `DocumentosTipos` |
| `Core/Entities/DbRows/FirmaDocumentoSolicitudDbRow.cs` | Nuevo | DbRow plano para Dapper con JOIN a `DocumentosSolicitud` y `DocumentosTipos` |
| `Maestros/Enums/EstadoFirmaDocumentoEnum.cs` | Nuevo | Enum con estados `NoSolicitado(1)`, `PendienteFirma(2)`, `Firmado(3)` y extensión `GetNombre()` |

### Core.Application

| Archivo | Tipo | Descripción |
|---|---|---|
| `Commands/InicializarConjuntoContractualCommand.cs` | Nuevo | Inicializa atómicamente 3 documentos + 3 firmas. Delega en SP idempotente |
| `Commands/SolicitarFirmaDocumentoSolicitudCommand.cs` | Nuevo | Transición `NoSolicitado(1) → PendienteFirma(2)` con validación de ownership y concurrencia |
| `Commands/RegistrarFirmaDocumentoSolicitudCommand.cs` | Nuevo | Transición `PendienteFirma(2) → Firmado(3)` con auditoría y evaluación de completitud del conjunto |
| `Queries/ConjuntoContractualEstadoGetQuery.cs` | Nuevo | Consulta estado agregado del conjunto contractual |
| `Queries/DocumentosSolicitudBySolicitudGetQuery.cs` | Nuevo | Lista documentos activos de una solicitud con tipo de documento |
| `Dtos/Requests/` (3 DTOs) | Nuevo | Request DTOs para inicializar, solicitar y registrar firma |
| `Dtos/Responses/ConjuntoContractualEstadoDto.cs` | Nuevo | Respuesta con conteos, `EsCompleto` y lista de pendientes |
| `Dtos/Responses/DocumentoSolicitudDto.cs` | Nuevo | Respuesta de documentos de solicitud (sin estado de firma individual) |
| `Dtos/Validators/` (2 validators) | Nuevo | FluentValidation para `SolicitarFirma` y `RegistrarFirma` |

### Core.Infrastructure

| Archivo | Tipo | Descripción |
|---|---|---|
| `Repositories/Core/DocumentosSolicitudRepository.cs` | Nuevo | Implementación Dapper — SPs: `Insert`, `BySolicitud_Select`, `ByIdAndSolicitud_Select`, `Delete` |
| `Repositories/Core/FirmasDocumentosSolicitudRepository.cs` | Nuevo | Implementación Dapper — SPs: `Insert`, `ByDocumentoSolicitud_Select`, `BySolicitud_Select`, `SolicitarFirma_Update`, `RegistrarFirma_Update`, `InicializarConjunto_Insert`. `GetEstadoConjuntoAsync` calculado en memoria |
| `Schemas/CoreSchemaContext.cs` | Modificado | Agrega lazy-init de los 2 nuevos repositorios |

### Core.Presentation

| Archivo | Tipo | Descripción |
|---|---|---|
| `DocumentosSolicitudesController.cs` | Modificado | 5 nuevos endpoints en región `#region Conjunto Contractual` |
| `IOCCoreSchema.cs` | Modificado | Registra 2 validators, 3 command handlers, 2 query handlers y 2 repositorios en DI |

### Testing

| Archivo | Tipo | Descripción |
|---|---|---|
| `UnitTesting/Application/Core/Commands/InicializarConjuntoContractualCommandHandlerTests.cs` | Nuevo | 3 tests: happy path, idempotencia, propagación de excepción |
| `UnitTesting/Application/Core/Commands/SolicitarFirmaDocumentoSolicitudCommandHandlerTests.cs` | Nuevo | 6 tests: happy path, NotFound (doc y firma), Conflict (PendienteFirma, Firmado, updateCero) |
| `UnitTesting/Application/Core/Commands/RegistrarFirmaDocumentoSolicitudCommandHandlerTests.cs` | Nuevo | 6 tests: happy path, NotFound, Conflict (EstadoNoSolicitado, EstadoFirmado), conjunto completo e incompleto |
| `UnitTesting/Application/Core/Queries/ConjuntoContractualEstadoGetQueryHandlerTests.cs` | Nuevo | 4 tests: sin documentos, parcial, completo, lista de pendientes |
| `IntegrationTesting/Infrastructure/Persistence/Repositories/Core/ConjuntoContractualTests.cs` | Nuevo | 7 tests (5 read-only + 2 destructivos) |

**Total de archivos modificados/nuevos:** 30

---

## 3. Cobertura de Criterios de Aceptación

| Criterio | Estado | Observación |
|---|---|---|
| **CA1** – Solicitar firma → estado `PendienteFirma` | ✅ Cubierto | `SolicitarFirmaDocumentoSolicitudCommand` + endpoint `PUT solicitar-firma` |
| **CA2** – Registrar firma → `Firmado`, fecha/usuario/versión | ✅ Cubierto | `RegistrarFirmaDocumentoSolicitudCommand` audita `FechaRegistroFirma`, `IdUsuarioRegistroFirma`, `IdArchivoFirmado` |
| **CA3** – Bloquear avance a facturación con firma parcial | ⚠️ Parcial | El backend expone `EsCompleto`, pero la validación efectiva depende de que el endpoint de solicitud de facturación consulte este flag. **No está implementado en este PR.** |
| **CA4** – Conjunto completo → `EsCompleto = true` | ✅ Cubierto | `ConjuntoContractualEstado.EsCompleto` calculado como propiedad computada (`TotalDocumentos > 0 && total == firmados`) |

### Cobertura de Reglas de Negocio

| Regla | Estado | Observación |
|---|---|---|
| **RN01** – Firma independiente por documento | ✅ | Cada documento tiene su propia fila en `FirmasDocumentosSolicitud` |
| **RN02** – Firma asociada a versión vigente | ✅ | `IdArchivoFirmado` referencia el archivo activo; el SP trabaja sobre el documento vigente |
| **RN03/RN04** – Avance requiere todos los docs firmados | ⚠️ | La señal `EsCompleto` está disponible; el bloqueo real debe implementarse en el endpoint de facturación |

---

## 4. Puntos positivos

- **Máquina de estados clara y bien protegida.** Las transiciones `1→2` y `2→3` son las únicas permitidas. Cualquier otro estado devuelve `409 Conflict` con el nombre legible del estado actual (gracias a `GetNombre()`).
- **Protección ante concurrencia.** Ambos commands verifican `filasAfectadas == 0` después del UPDATE del SP, lo que bloquea dobles clicks o requests paralelos.
- **Ownership check.** `GetByIdAndSolicitudAsync` valida que el documento pertenezca a la solicitud antes de cualquier operación. Previene manipulación de IDs ajenos.
- **SP idempotente para inicialización.** `InicializarConjuntoContractualCommand` delega en un SP que no duplica si el conjunto ya existe.
- **Tests unitarios exhaustivos.** Cubren Happy Path, ambos NotFound, ambos Conflict de estado y el caso de concurrencia (filasAfectadas=0). El patrón de inyección de `_unitOfWork` vía reflection es consistente con el resto del proyecto.
- **Sin SQL inline.** Todo se delega a SPs, consistente con la convención del proyecto.
- **BOM cleanup** en `ICoreSchemaContext.cs` y `CoreSchemaContext.cs` — limpieza cosmética bienvenida.

---

## 5. Observaciones / Issues

### 5.1 ⚠️ `InicializarConjuntoContractualRequestDto` es dead code

**Archivo:** `Core.Application/Modules/Core/Dtos/Requests/InicializarConjuntoContractualRequestDto.cs`

El DTO fue creado pero el controller solo usa `[FromRoute] int idSolicitud` y nunca instancia este DTO. No está referenciado en ningún test ni en ningún otro lugar.

**Recomendación:** Eliminar el archivo o integrarlo al endpoint si se necesita validación adicional.

---

### 5.2 ⚠️ Validator de `RegistrarFirmaDocumentoSolicitudRequestDto` valida campos que vienen de la ruta

**Archivo:** `RegistrarFirmaDocumentoSolicitudRequestDtoValidator.cs`

```csharp
RuleFor(x => x.IdSolicitud).GreaterThan(0);
RuleFor(x => x.IdDocumentoSolicitud).GreaterThan(0);
```

En el controller, `dto.IdSolicitud` e `dto.IdDocumentoSolicitud` son seteados **después** de que FluentValidation ya corrió (la validación ocurre en el pipeline de model binding). Si el caller no envía estos campos en el body JSON (lo cual es probable, dado que son parámetros de ruta), el validator fallará con 400 Bad Request antes de llegar al action.

La misma observación aplica para el validator de `SolicitarFirmaDocumentoSolicitudRequestDtoValidator`:

```csharp
RuleFor(x => x.IdSolicitud).GreaterThan(0);
RuleFor(x => x.IdDocumentoSolicitud).GreaterThan(0);
```

Pero `SolicitarFirma` no tiene body (`[FromBody]`), así que el validator nunca se invoca para ese DTO. La DI está registrada pero no hay `[FromBody]` en el endpoint.

**Recomendación:**
- Para `RegistrarFirmaDocumentoSolicitudRequestDtoValidator`: remover las reglas para `IdSolicitud` e `IdDocumentoSolicitud` (son de ruta), dejar solo `IdArchivoFirmado > 0`.
- Para `SolicitarFirmaDocumentoSolicitudRequestDtoValidator`: el registro en DI es inocuo (el validator no se invoca), pero puede removerse para evitar confusión.

---

### 5.3 ℹ️ `GetEstadoConjuntoAsync` calcula estado en memoria (2 viajes a DB)

**Archivo:** `FirmasDocumentosSolicitudRepository.cs`

```csharp
public async Task<ConjuntoContractualEstado> GetEstadoConjuntoAsync(int idSolicitud)
{
    var firmas = await GetBySolicitudAsync(idSolicitud); // viaje 1
    // ... LINQ en memoria para contar estados
}
```

Dado que el conjunto siempre tiene exactamente 3 documentos, el impacto es despreciable. Sin embargo, si en el futuro se escala a más documentos, un SP dedicado con `COUNT + GROUP BY` sería más eficiente. Dejar como deuda técnica o comentar en el código.

---

### 5.4 ℹ️ No hay endpoint que exponga el estado de firma individual por documento

La interfaz `IFirmasDocumentosSolicitudRepository` declara `GetBySolicitudAsync` que retorna `IEnumerable<FirmaDocumentoSolicitudDbRow>` con `EstadoFirma` por documento, pero este método no está expuesto vía ningún endpoint.

La US menciona **"Visualización del estado de firma por documento"**, lo cual con las APIs actuales requeriría del frontend combinar:
- `GET /documentos-solicitud` → lista de docs (sin estado de firma)
- `GET /conjunto-contractual/estado` → conteos agregados + nombres de pendientes

No hay una respuesta unificada por documento que muestre `NombreDocumento + EstadoFirma`.

**Recomendación:** Si el frontend necesita visualización individual (lo cual es probable según la US), agregar ya sea un query `FirmasBySolicitudGetQuery` o enriquecer `DocumentoSolicitudDto` con `EstadoFirma`. Puede quedar como tarea del siguiente sprint si se acordó con front.

---

### 5.5 ℹ️ Tests de integración con `IdSolicitudTest = 1` hardcodeado

```csharp
private const int IdSolicitudTest = 1;
```

El test `FlujoCompleto_Inicializar_Consultar_Solicitar_Registrar` es **destructivo** (soft-deletes documentos del `IdSolicitud = 1`). Si ese registro no existe en el entorno de integración, los tests fallarán silenciosamente (algunos returns anticipados) o con excepciones de FK.

**Recomendación:** Documentar en el fixture o README de integration testing cuál es el IdSolicitud válido para el entorno, o usar un setup que lo cree desde cero.

---

### 5.6 ℹ️ Nombre de rama no sigue la convención del proyecto

La convención del proyecto es `feature/{idUS}-T{idTask}-descripcion-kebab-case`.
La rama actual es `COR-OTG-SFI-firma-conjunto-contractual`.

No bloquea el merge, pero vale la pena alinearlo para el seguimiento en el board.

---

## 6. SPs esperados en base de datos

El PR asume la existencia de los siguientes SPs (no incluidos en este diff — se esperan en los scripts SQL):

| SP | Acción |
|---|---|
| `Core.DocumentosSolicitud_Insert` | Inserta un documento de solicitud |
| `Core.DocumentosSolicitud_BySolicitud_Select` | Lista documentos activos de una solicitud |
| `Core.DocumentosSolicitud_ByIdAndSolicitud_Select` | Busca documento por Id + solicitud (ownership) |
| `Core.DocumentosSolicitud_Delete` | Baja lógica (soft delete) |
| `Core.FirmasDocumentosSolicitud_Insert` | Inserta registro de firma |
| `Core.FirmasDocumentosSolicitud_ByDocumentoSolicitud_Select` | Obtiene firma de un documento |
| `Core.FirmasDocumentosSolicitud_BySolicitud_Select` | Lista todas las firmas de una solicitud |
| `Core.FirmasDocumentosSolicitud_SolicitarFirma_Update` | Transición `1→2` con WHERE sobre estado actual |
| `Core.FirmasDocumentosSolicitud_RegistrarFirma_Update` | Transición `2→3` con WHERE sobre estado actual |
| `Core.DocumentosSolicitud_InicializarConjunto_Insert` | Crea atómicamente 3 docs + 3 firmas (idempotente) |

⚠️ **Verificar** que todos estos SPs estén incluidos en los scripts de migración de la release.

---

## 7. Checklist de revisión

| Criterio | Estado |
|---|---|
| Sigue Clean Architecture (capas y dependencias correctas) | ✅ |
| Sigue CQRS (Commands/Queries/Handlers separados) | ✅ |
| Naming conventions del proyecto | ✅ |
| Sin SQL inline (todo vía SPs) | ✅ |
| FluentValidation registrada en DI | ✅ (con observación 5.2) |
| Handlers registrados en DI (IOCCoreSchema) | ✅ |
| Repositorios registrados en DI | ✅ |
| Ownership validation (doc pertenece a solicitud) | ✅ |
| Protección ante concurrencia | ✅ |
| Manejo de errores con `CustomException` y HTTP codes correctos | ✅ |
| Unit tests (happy path + error paths) | ✅ |
| Integration tests | ✅ (con observación 5.5) |
| Dead code eliminado | ⚠️ (observación 5.1) |
| Bloqueo avance a facturación implementado | ⚠️ (CA3 — fuera de scope en este PR) |
| Script SQL de SPs incluido | ❓ (no visible en este diff) |

---

## 8. Conclusión

La implementación es sólida, coherente con la arquitectura del proyecto y cubre correctamente la lógica de negocio de la US. Los puntos críticos a resolver antes del merge son:

1. **Issue 5.2 (Validator + campos de ruta):** Puede provocar 400 Bad Request inesperados en producción para el endpoint `RegistrarFirma`. **Requiere corrección.**
2. **Issue 5.1 (Dead code):** Limpiar `InicializarConjuntoContractualRequestDto`. Menor.
3. **Confirmar scripts SQL** de los 10 SPs esperados.

El punto 5.4 (endpoint individual de estado de firma por documento) puede quedar como seguimiento con el equipo de frontend según lo que realmente necesiten visualizar.
