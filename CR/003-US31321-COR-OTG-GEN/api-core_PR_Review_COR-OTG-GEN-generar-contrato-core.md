# PR Review — `feature/COR-OTG-GEN-Generar-contrato-desde-CORE`

**Fecha de revisión:** 2026-03-20
**Branch:** `feature/COR-OTG-GEN-Generar-contrato-desde-CORE`
**Commits del PR:**
- `a842ed9` Add generic contract templates repo and converter
- `30a8f85` Add generic contract generation and PDF support

**Scope:** `Api.Core / releases/v3.0` (scripts 049–052) · 48 archivos · +3775 / −38 líneas
**User Story:** US31321 · COR-OTG-GEN – Generar contrato desde CORE

---

## 1. Archivos del PR

### 1.1 Base de datos (scripts 049–052)

| # | Script | Contenido |
|---|---|---|
| 049 | `049_documentos_tipos_pf_pj_y_alter_sp_inicializar_conjunto.sql` | Seed de 4 nuevos `DocumentosTipos` bank-specific: GF PF ARS, GF PJ ARS, GF PF USD, GF PJ USD |
| 050 | `050_create_table_templates_contrato_generico.sql` | Tabla `[Parametrizaciones].[TemplatesContratoGenerico]` + FKs + índice UNIQUE + 2 SPs + Seed (GF ARS, GF USD, Fianza, Pagaré) |
| 051 | `051_update_template_gf_ars_placeholders.sql` | Update del template `CONT-GEN-GF-ARS` con placeholders `{{CAMPO}}` consistentes con USD |
| 052 | `052_create_table_contratos_genericos_contenido.sql` | Tabla `[Core].[ContratosGenericosContenido]` + FKs + índice UNIQUE |

### 1.2 Código — Domain

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.Domain/Core/Contracts/IContratoDataResolverService.cs` | Interface | Contrato del servicio de resolución de datos (carta banco > solicitud) |
| `Core.Domain/Core/Contracts/IComparacionCartaBancoService.cs` | Interface | Contrato del servicio comparador de carta banco |
| `Core.Domain/Core/Contracts/IContratosGenericosContenidoRepository.cs` | Interface | Repositorio de contenido editable de contratos genéricos |
| `Core.Domain/Core/Contracts/ICoreSchemaContext.cs` | Interface | Contexto de schema Core (agregado `ContratosGenericosContenido`) |
| `Core.Domain/Core/Dtos/ContratoDataDto.cs` | DTO | Datos resueltos para placeholders (RazonSocial, Cuit, Monto, MontoLetras, etc.) |
| `Core.Domain/Core/Entities/ContratoGenericoContenido.cs` | Entidad | Contenido editable + audit del contrato genérico |
| `Core.Domain/Core/Entities/DbRows/ContratoGenericoContenidoDbRow.cs` | DbRow | Resultado de SP para lectura del contrato |
| `Core.Domain/Parametrizaciones/Contracts/IParametrizacionesSchemaContext.cs` | Interface | Contexto de schema Parametrizaciones (agregado `TemplatesContratoGenerico`) |
| `Core.Domain/Parametrizaciones/Contracts/ITemplatesContratoGenericoRepository.cs` | Interface | Repositorio de templates por moneda + tipo documento |
| `Core.Domain/Parametrizaciones/Entities/TemplateContratoGenerico.cs` | Entidad | Template genérico con sus FKs |
| `Core.Domain/Parametrizaciones/Entities/DbRows/TemplateContratoGenericoDbRow.cs` | DbRow | Resultado de SP |
| `Core.Domain/Parametrizaciones/Entities/MatrizCertificadoAval.cs` | Entidad | (Actualizada) |
| `Core.Domain/Parametrizaciones/Entities/DbRows/MatrizCertificadoAvalDbRow.cs` | DbRow | (Actualizada) |

### 1.3 Código — Application

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.Application/Modules/Core/Commands/GenerarContratoCommand.cs` | Command | Genera el conjunto contractual completo (GF, Fianza, Pagaré). Requiere `DocumentosSolicitud` previamente inicializados |
| `Core.Application/Modules/Core/Commands/ActualizarContenidoContratoGenericoCommand.cs` | Command | Actualiza el contenido editable de un contrato genérico ya generado |
| `Core.Application/Modules/Core/Commands/GenerarPdfContratoGenericoCommand.cs` | Command | Genera el PDF a partir del `ContenidoEditable` HTML vía servicio PDF |
| `Core.Application/Modules/Core/Commands/ConfirmarContratoGarantiaCommand.cs` | Command | Confirma el contrato de garantía con datos del formulario (firmante, cuentas, fecha) |
| `Core.Application/Modules/Core/Commands/PrevisualizarContratoGarantiaCommand.cs` | Command | Preview con interpolación completa (datos backend + formulario frontend) |
| `Core.Application/Modules/Core/Queries/ContratoGenericoByDocumentoSolicitudGetQuery.cs` | Query | Consulta el contenido vigente de un contrato genérico |
| `Core.Application/Modules/Core/Queries/DatosContratoGarantiaGetQuery.cs` | Query | Retorna los datos resueltos del contrato (para el formulario front) |
| `Core.Application/Modules/Core/Helpers/ContratoGarantiaInterpolationHelper.cs` | Helper | Fuente única de verdad: placeholders, nombres de DocumentoTipos, lógica de interpolación |
| `Core.Application/Modules/Core/Helpers/ContratoGenericoPdfUploadHelper.cs` | Helper | Sube el PDF generado al servicio de archivos |
| `Core.Application/Modules/Core/Helpers/DocumentoScopeResolver.cs` | Helper | Resuelve el scope del documento (banco / empresa) |
| `Core.Application/Modules/Core/Dtos/Requests/ActualizarContenidoContratoGenericoRequestDto.cs` | DTO | Payload para actualizar contenido |
| `Core.Application/Modules/Core/Dtos/Requests/ContratoGarantiaFormularioRequestDto.cs` | DTO | Payload para confirmar/previsualizar (firmante, cuentas bancarias, fecha firma) |
| `Core.Application/Modules/Core/Dtos/Responses/ContratoGenericoGeneradoDto.cs` | DTO | Response de generación (lista de contratos generados) |
| `Core.Application/Modules/Core/Dtos/Responses/ContratoGenericoVigenteDto.cs` | DTO | Response de consulta (contenido editable + archivos) |
| `Core.Application/Modules/Core/Dtos/Responses/DatosContratoGarantiaDto.cs` | DTO | Response de datos para formulario front |
| `Core.Application/Modules/Core/Dtos/Validators/...` | Validadores | FluentValidation para ambos RequestDtos |

### 1.4 Código — Infraestructura

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.Infraestructure/Services/Core/ContratoDataResolverService.cs` | Service | Implementa `IContratoDataResolverService`: prioriza carta banco, fallback a solicitud. Domicilio siempre desde Empresa |
| `Core.Infraestructure/Services/Core/ComparacionCartaBancoService.cs` | Service | Obtiene metadata de carta banco para la solicitud |
| `Core.Infraestructure/Services/Core/MontoEnLetrasConverter.cs` | Service | Convierte decimal a texto en español (ARS/USD) |
| `Core.Infraestructure/Persistence/Repositories/Core/ContratosGenericosContenidoRepository.cs` | Repo | CRUD vía SP: Create, GetByDocumentoSolicitud, Update, SoftDelete |
| `Core.Infraestructure/Persistence/Repositories/Parametrizaciones/TemplatesContratoGenericoRepository.cs` | Repo | SP `ByMonedaDocumentoTipo_Select`, `ByDocumentoTipo_Select` |
| `Core.Infraestructure/Persistence/Schemas/CoreSchemaContext.cs` | Schema | Registro de `ContratosGenericosContenidoRepository` |
| `Core.Infraestructure/Persistence/Schemas/ParametrizacionesSchemaContext.cs` | Schema | Registro de `TemplatesContratoGenericoRepository` |
| `Core.Infraestructure/Persistence/Schemas/MaestrosSchemaContext.cs` | Schema | (Actualizado) |

### 1.5 Código — Presentation

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.Presentation/Api/Core/Controllers/v1/ContratosGenericosController.cs` | Controller | 5 endpoints (POST generar, GET consulta, PUT actualizar, POST PDF, POST confirmar/preview) |
| `Core.Presentation/Api/Extensions/IOCCoreSchema.cs` | DI | Registro de `IContratoDataResolverService` y `IComparacionCartaBancoService` |

### 1.6 Tests

| Archivo | Tipo | Descripción |
|---|---|---|
| `Core.UnitTesting/Infraestructure/Services/Core/ContratoDataResolverServiceTests.cs` | Unit | Tests de resolución de datos (carta banco / solicitud), 285 líneas |
| `Core.UnitTesting/Infraestructure/Services/Core/MontoEnLetrasConverterTests.cs` | Unit | Tests de conversión a letras ARS y USD, 336 líneas |
| `Core.UnitTesting/Application/Core/Validators/ActualizarContenidoContratoGenericoRequestDtoValidatorTests.cs` | Unit | Tests de validadores |
| `Core.UnitTesting/Application/Helper/DocumentoTipoEntidadResolverTests.cs` | Unit | Tests del helper de scope |
| `Core.UnitTesting/Infrastructure/Persistence/Repositories/Core/ContratosGenericosContenidoRepositoryTests.cs` | Unit | Tests del repositorio de contenido, 322 líneas |
| `Core.UnitTesting/Infrastructure/Persistence/Repositories/Parametrizaciones/TemplatesContratoGenericoRepositoryTests.cs` | Unit | Tests del repositorio de templates, 222 líneas |
| `Core.IntegrationTesting/Infrastructure/Persistence/Repositories/Core/ContratosGenericosContenidoTests.cs` | Integration | Tests de integración del repositorio de contenido |
| `Core.IntegrationTesting/Infrastructure/Persistence/Repositories/Parametrizaciones/TemplatesContratoGenericoTests.cs` | Integration | Tests de integración del repositorio de templates |

---

## 2. Modelo de datos implementado

```
[Parametrizaciones].[TemplatesContratoGenerico]          (050 – creación + seed)
  IdTemplateContratoGenerico   PK identity
  IdMoneda                     FK → Parametrizaciones.Monedas  |  nullable (transversales: Fianza, Pagaré)
  IdDocumentoTipo              FK → Parametrizaciones.DocumentosTipos  |  NOT NULL
  NombreTemplate               NVARCHAR(200)  (e.g. "CONT-GEN-GF-ARS")
  ContenidoTemplate            NVARCHAR(MAX)  HTML con placeholders {{CAMPO}}
  Activo + auditoría estándar
  UNIQUE filtrado: (IdDocumentoTipo, IdMoneda) WHERE Activo=1 AND FechaBaja IS NULL

[Core].[ContratosGenericosContenido]                     (052 – creación)
  IdContratoGenericoContenido  PK identity
  IdDocumentoSolicitud         FK → Core.DocumentosSolicitud  |  UNIQUE activo (1:1)
  IdTemplateContratoGenerico   FK → Parametrizaciones.TemplatesContratoGenerico  |  nullable
  ContenidoEditable            NVARCHAR(MAX)  HTML post-interpolación, editable
  DatosAutocompletados         NVARCHAR(MAX)  JSON con los datos resueltos (trazabilidad)
  IdArchivo                    FK → Archivos (DOCX)   |  nullable
  IdArchivoPdf                 FK → Archivos (PDF)    |  nullable
  Auditoría estándar
```

---

## 3. Flujo de generación (RN01–RN04)

```
POST /v1/solicitudes/{id}/conjunto-contractual/generar
  → GenerarContratoCommandHandler
       ├─ Valida que DocumentosSolicitud del conjunto estén inicializados
       ├─ ContratoDataResolverService.ResolveAsync()
       │     ├─ CartaBancoService.GetMetadata() → prioridad carta banco
       │     └─ fallback: solicitud.MontoSolicitado / solicitud.NombreEntidadFinanciera
       ├─ Para cada documento contractual (GF, Fianza, Pagaré):
       │     ├─ TemplatesContratoGenericoRepo.ByMonedaDocumentoTipo (GF ARS/USD)
       │     │   o TemplatesContratoGenericoRepo.ByDocumentoTipo (Fianza/Pagaré)
       │     ├─ ContratoGarantiaInterpolationHelper.Interpolate(template, datos)
       │     └─ ContratosGenericosContenidoRepo.CreateAsync() con soft-delete del anterior
       └─ Retorna ContratoGenericoGeneradoDto (lista con IdDocumentoSolicitud + IdTemplate)

PUT /v1/solicitudes/{id}/contratos-genericos/{idDoc}      → actualizar ContenidoEditable
POST /v1/solicitudes/{id}/contratos-genericos/{idDoc}/pdf → generar PDF (PdfService)
POST /v1/solicitudes/{id}/contratos-genericos/{idDoc}/confirmar → ConfirmarContratoGarantia
POST /v1/solicitudes/{id}/contratos-genericos/{idDoc}/previsualizar → preview completo
```

---

## 4. Análisis técnico

### ✅ Fortalezas

| Aspecto | Detalle |
|---|---|
| **RN completas** | Las 4 reglas de negocio (conjunto completo · modelo por moneda · editable · versionado) están implementadas y validadas |
| **Prioridad carta banco** | `ContratoDataResolverService` implementa exactamente AC3/AC4: carta banco > solicitud para todos los campos. Domicilio siempre desde Empresa (correcto) |
| **Fuente única de verdad** | `ContratoGarantiaInterpolationHelper` centraliza todos los nombres de DocumentoTipos y los placeholders `{{CAMPO}}`, evitando magic strings dispersos |
| **Trazabilidad** | `DatosAutocompletados` persiste el JSON con la fuente de datos utilizada (`FuenteDatos: "CartaBanco"/"Solicitud"`), cumpliendo la nota técnica |
| **Soft delete en re-generación** | Al regenerar, el handler hace soft-delete del contenido anterior antes de crear el nuevo (RN04: versionado) |
| **Cobertura de tests** | 4 unit test files + 2 integration test files, ~1165 líneas de tests en total. Cubre resolución de datos, repositorios, converters y validadores |
| **`MontoEnLetrasConverter`** | Implementación correcta para ARS y USD con manejo de centavos |
| **Índice UNIQUE filtrado** | Tanto en `TemplatesContratoGenerico` como en `ContratosGenericosContenido` usa `WHERE FechaBaja IS NULL`, compatibilidad correcta con soft-delete |

### ⚠️ Observaciones / Mejoras sugeridas

#### 1. Identificación de DocumentosTipos por nombre string (fragile matching)
`ContratoGarantiaInterpolationHelper.EsGarantiaFinanciera()` identifica documentos por `Contains("Garantía Financiera ARS", ...)` sobre el `NombreDocumentoTipo`. Cualquier cambio en el nombre del tipo de documento en la DB rompe silenciosamente la lógica.
```csharp
// Actual (frágil):
return NombresDocumentoTipoGf.Any(n =>
    nombreDocumentoTipo.Contains(n, StringComparison.OrdinalIgnoreCase));

// Sugerido: comparar por Nombre (código) del DocumentoTipo, no por Titulo
// e.g. NombreDocumentoTipo = "ContratoGarantiaFinancieraARS" (campo Nombre de la tabla)
```
**Impacto:** medio. No bloquea esta US pero es deuda técnica.

#### 2. `ConfirmarContratoGarantiaCommand` — ¿No cambia estado?
La US dice explícitamente "No implica firma ni cambio de estado". El endpoint `confirmar` recibe datos del formulario (firmante, cuentas bancarias, fecha), interpola el template completo y genera el PDF. Verificar que el handler no modifica el estado de la solicitud ni del conjunto contractual más allá del contenido editable.

#### 3. Script 049 — `DocumentosTipos` bank-specific en este PR
El script 049 crea 4 tipos bank-specific (GF PF ARS, GF PJ ARS, GF PF USD, GF PJ USD) que son para la US `COR-OTG-CER` (contratos bank-specific), no para `COR-OTG-GEN`. Aparecen aquí porque el SP `InicializarConjunto` los referencia. Correcto incluirlos, pero documentarlo en el PR para evitar confusión inter-US.

#### 4. Fianza y Pagaré sin template real
Los templates de Fianza y Pagaré se insertan en el seed de script 050 con `ContenidoTemplate = ''` (placeholder vacío) ya que la US los declara como "documentos obligatorios a generar sin modelo actual". El `GenerarContratoCommand` debería manejar este caso con un fallback explícito (e.g., `ContenidoEditable = ''` y log de advertencia) en lugar de potencialmente propagar un template vacío sin advertir.

---

## 5. Base de datos — análisis scripts 049–052

### Script 049 — Seed DocumentosTipos
✅ Usa `WHERE NOT EXISTS` para idempotencia  
✅ Valida que existan `IdDocumentoFuente`, `IdFormaAutomatica` e `IdDocConfigPdfJpgPng` antes de insertar  
✅ Usa `XACT_ABORT ON` + transacción con `SET @tran`

### Script 050 — TemplatesContratoGenerico
✅ `IF OBJECT_ID ... IS NULL` en CREATE TABLE  
✅ `IF NOT EXISTS` en cada FK y en el índice UNIQUE  
✅ Índice `FILLFACTOR = 90` con `WHERE (Activo = 1 AND FechaBaja IS NULL)` — correcto para soft-delete  
✅ Seed de los 4 templates (GF ARS, GF USD, Fianza, Pagaré) con `WHERE NOT EXISTS`  
⚠️ El seed de Fianza y Pagaré inserta `ContenidoTemplate = N''` — documentado en la US como intencional, pero debería tener un comentario explícito en el script

### Script 051 — Update template GF ARS
✅ El template ARS ahora usa placeholders `{{CAMPO}}` consistentes con USD  
✅ `IF @@ROWCOUNT = 0 → PRINT 'ADVERTENCIA'` para detectar si no encontró el registro  
✅ Transacción con `XACT_ABORT ON`

### Script 052 — ContratosGenericosContenido
✅ `IF OBJECT_ID ... IS NULL` en CREATE TABLE  
✅ FKs a `DocumentosSolicitud` y `TemplatesContratoGenerico` con `IF NOT EXISTS`  
✅ Índice UNIQUE filtrado `WHERE FechaBaja IS NULL` — permite re-generación via soft-delete  
✅ `IdArchivo` e `IdArchivoPdf` nullable — correcto, el PDF se genera en paso separado

---

## 6. Análisis: ¿Unificar `TemplatesContratoGenerico` con `Plantillas`?

### Contexto
El script 038 creó `[Parametrizaciones].[Plantillas]` para templates de Actas de Consejo y Memorándums:
```
Plantillas: resolución por TipoPlantilla.Codigo (ACTA_CONSEJO, MEMORANDUM)
            + Version + JsonSchemaVariables (schema dinámico de variables)
```
El script 050 crea `[Parametrizaciones].[TemplatesContratoGenerico]`:
```
TemplatesContratoGenerico: resolución por IdMoneda + IdDocumentoTipo
                           sin JsonSchemaVariables
```

### Análisis
| Criterio | `Plantillas` | `TemplatesContratoGenerico` |
|---|---|---|
| **Mecanismo de resolución** | `TipoPlantilla.Codigo` (string) | `IdMoneda + IdDocumentoTipo` (FKs tipadas) |
| **Versionado** | `EsVigente` + `Version` numérico | Sin versión explícita |
| **Schema de variables** | `JsonSchemaVariables` (dinámico) | Sin schema (placeholders fijos `{{CAMPO}}`) |
| **Dependencia** | Agnóstico a moneda/documento | Acoplado a `Monedas` y `DocumentosTipos` |
| **Dominio** | Documentos de gobierno interno (Actas, Memos) | Contratos de garantía financiera |

### Conclusión
**No se recomienda unificar en este PR.** Los mecanismos de resolución son estructuralmente distintos: `Plantillas` requiere un Tipo genérico + versión, `TemplatesContratoGenerico` requiere una clave compuesta moneda + tipo de documento. Unificar requeriría agregar columnas nullable (`IdMoneda`, `IdDocumentoTipo`) a `Plantillas` o crear una abstracción intermedia, aumentando la complejidad sin beneficio inmediato.

**Recomendación futura (deuda técnica):** evaluar si a largo plazo conviene una tabla `TemplatesBase` con discriminador de tipo, pero no en esta iteración.

---

## 7. Criterios de aceptación — cobertura

| AC | Escenario | Estado |
|---|---|---|
| AC1 | Generación contrato ARS → modelo `CONT-GEN-GF-ARS` | ✅ Implementado vía `TemplatesContratoGenericoRepo.ByMonedaDocumentoTipo` |
| AC2 | Generación contrato USD → modelo `CONT-GEN-GF-USD` | ✅ Mismo mecanismo, `IdMoneda` diferente |
| AC3 | Autocompletado desde carta banco | ✅ `ContratoDataResolverService`: `usaCartaBanco = cartaBanco != null` |
| AC4 | Autocompletado desde solicitud (fallback) | ✅ Fallback a `solicitud.MontoSolicitado`, `solicitud.NombreEntidadFinanciera` |
| AC5 | Sin selección manual de modelo | ✅ Selección automática, no hay selector en el request |

---

## 8. Resumen ejecutivo

| | |
|---|---|
| **Archivos** | 48 (+3775 / −38) |
| **DB scripts** | 4 (049–052) |
| **Tests** | 6 archivos de tests, ~1165 líneas |
| **Aprobación** | ✅ Aprobar con observaciones |

**Observaciones que requieren seguimiento:**
1. (Media) Identificación de DocumentosTipos por nombre string → evaluar migración a comparación por `Nombre` (código) del tipo
2. (Baja) Templates de Fianza/Pagaré vacíos → agregar log de advertencia en `GenerarContratoCommand` cuando el template está vacío
3. (Baja) Script 049 incluye tipos bank-specific de COR-OTG-CER → documentar en el cuerpo del PR

**No bloquea el merge.** La US31321 está completamente implementada respecto a RN01–RN04 y todos los criterios de aceptación.
