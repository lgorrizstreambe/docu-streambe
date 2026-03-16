# PR Review — `COR-OTG-SFI-firma-conjunto-contractual`

**Fecha de revisión:** 2026-03-16  
**Branch:** `COR-OTG-SFI-firma-conjunto-contractual`  
**Commit:** `1ed6e00 Add DocumentosSolicitud and Firmas tables & SPs`  
**Scope:** `Api.Core / releases/v3.0` (scripts 014–017)

---

## 1. Archivos del PR

| # | Archivo | Contenido |
|---|---|---|
| 014 | `014_create_table_documentos_solicitud.sql` | Tabla `[Core].[DocumentosSolicitud]` + FKs + índices + MS_Description + seed de 4 tipos de documento |
| 015 | `015_create_table_firmas_documentos_solicitud.sql` | Tabla `[Core].[FirmasDocumentosSolicitud]` + FK + índices + MS_Description |
| 016 | `016_create_sp_documentos_solicitud.sql` | 4 SPs: `_Insert`, `_BySolicitud_Select`, `_ByIdAndSolicitud_Select`, `_Delete` (soft) |
| 017 | `017_create_sp_firmas_documentos_solicitud.sql` | 7 SPs: `_Insert`, `_ByDocumentoSolicitud_Select`, `_BySolicitud_Select`, `_SolicitarFirma_Update`, `_RegistrarFirma_Update`, `_ConjuntoCompleto_Select`, `_InicializarConjunto_Insert` |

---

## 2. Modelo de datos implementado

```
[Core].[DocumentosSolicitud]
  IdDocumentoSolicitud  (PK identity)
  IdSolicitud           FK → Core.Solicitudes
  IdDocumentoTipo       FK → Parametrizaciones.DocumentosTipos
  IdArchivo             nullable (no siempre hay archivo al crear)
  IdEstadoConfiguracion FK → Parametrizaciones.EstadosConfiguraciones
  IdDocumentoSolicitudReemplazado  FK autoreferencial (versionado)
  IdDocumentoFuente     FK → Parametrizaciones.DocumentosFuentes
  IdDocumentoForma      FK → Parametrizaciones.DocumentosFormas
  [auditoría estándar]

[Core].[FirmasDocumentosSolicitud]
  IdFirmaDocumentoSolicitud  (PK identity)
  IdDocumentoSolicitud       FK → Core.DocumentosSolicitud
  EstadoFirma                INT (1=NoSolicitado | 2=PendienteFirma | 3=Firmado)
  FechaSolicitudFirma / IdUsuarioSolicitudFirma
  FechaRegistroFirma  / IdUsuarioRegistroFirma / IdArchivoFirmado
  [auditoría estándar]
```

**Relación:** 1:1 entre documento y firma (constraint único filtrado por FechaBaja IS NULL).  
Extensible a 1:N removiendo `UQ_FirmasDocumentosSolicitud_Documento`.

---

## 3. Tipos de documento seedeados (014)

| Nombre | Título |
|---|---|
| `ContratoGarantiaFinancieraARS` | Contrato de Garantia Financiera ARS |
| `ContratoGarantiaFinancieraUSD` | Contrato de Garantia Financiera USD |
| `ContratoFianza` | Contrato de Fianza |
| `NotaPagare` | Nota / Pagaré |

Seed idempotente con `IF NOT EXISTS`.

---

## 4. Cobertura de Criterios de Aceptación

| CA | Descripción | SP que lo cubre | Estado |
|---|---|---|---|
| CA1 | Solicitar firma → estado Pendiente | `FirmasDocumentosSolicitud_SolicitarFirma_Update` (1→2, valida `EstadoFirma=1` en WHERE) | ✅ |
| CA2 | Registrar firmado → estado Firmado + auditoría | `FirmasDocumentosSolicitud_RegistrarFirma_Update` (2→3, recibe `@pIdArchivoFirmado`, registra fecha y usuario) | ✅ |
| CA3 | Firma parcial bloquea avance | `FirmasDocumentosSolicitud_ConjuntoCompleto_Select` retorna `EsCompleto=0`; la API usa este resultado para bloquear | ✅ |
| CA4 | Conjunto completo habilita avance | Mismo SP retorna `EsCompleto=1` cuando todos tienen `EstadoFirma=3` | ✅ |

---

## 5. Cobertura de Reglas de Negocio

| RN | Descripción | Implementación |
|---|---|---|
| RN01 | Firma independiente por documento | Una fila en `FirmasDocumentosSolicitud` por documento; each update es individual |
| RN02 | Firma corresponde a versión vigente | `IdArchivoFirmado` se captura en `RegistrarFirma_Update` al momento de la firma |
| RN03 | Avance requiere todos firmados | `ConjuntoCompleto_Select` → `EsCompleto` BIT; API lo evalúa antes de avanzar |
| RN04 | No se admite firma parcial para avanzar | Mismo mecanismo que RN03 |

---

## 6. SPs implementados — resumen

### DocumentosSolicitud (016)
| SP | Tipo | Descripción |
|---|---|---|
| `DocumentosSolicitud_Insert` | DML | Inserta un documento. `OUTPUT INSERTED.IdDocumentoSolicitud`. |
| `DocumentosSolicitud_BySolicitud_Select` | SELECT | Todos los documentos de una solicitud con JOIN a `DocumentosTipos`. |
| `DocumentosSolicitud_ByIdAndSolicitud_Select` | SELECT | Ownership check: valida que el doc pertenezca a la solicitud. |
| `DocumentosSolicitud_Delete` | DML | Soft delete. Retorna `RowsAffected`. |

### FirmasDocumentosSolicitud (017)
| SP | Tipo | Descripción |
|---|---|---|
| `FirmasDocumentosSolicitud_Insert` | DML | Inserta registro de firma (estado inicial=1). |
| `FirmasDocumentosSolicitud_ByDocumentoSolicitud_Select` | SELECT | Firma de un documento específico. |
| `FirmasDocumentosSolicitud_BySolicitud_Select` | SELECT | Todas las firmas de una solicitud con JOIN a docs y tipos. |
| `FirmasDocumentosSolicitud_SolicitarFirma_Update` | DML | Transición 1→2. Registra `FechaSolicitudFirma` y usuario. |
| `FirmasDocumentosSolicitud_RegistrarFirma_Update` | DML | Transición 2→3. Registra `FechaRegistroFirma`, usuario e `IdArchivoFirmado`. |
| `FirmasDocumentosSolicitud_ConjuntoCompleto_Select` | SELECT | Evalúa completitud. Retorna `TotalDocumentos`, `DocumentosFirmados`, `EsCompleto` (BIT). |
| `DocumentosSolicitud_InicializarConjunto_Insert` | DML | Atómico e idempotente. Inserta los 3 documentos contractuales + 3 firmas iniciales según moneda de la solicitud. |

---

## 7. Checklist estándar Garantizar

| Regla | 014 tabla | 015 tabla | 016 SPs | 017 SPs |
|---|---|---|---|---|
| Nomenclatura (plural PascalCase, schema) | ✅ | ✅ | ✅ | ✅ |
| PK `Id<Entidad>` INT IDENTITY(1,1) | ✅ | ✅ | N/A | N/A |
| Auditoría 6 columnas + `DF_*_FechaAlta` | ✅ | ✅ | N/A | N/A |
| MS_Description tabla + columnas (idempotente) | ✅ | ✅ | N/A | N/A |
| `CREATE OR ALTER PROCEDURE` | N/A | N/A | ✅ | ✅ |
| `SET NOCOUNT ON` | N/A | N/A | ✅ | ✅ |
| `TRY/CATCH` + `sp_error` + `THROW` | N/A | N/A | ✅ | ✅ |
| `SET XACT_ABORT ON` en DML | N/A | N/A | ✅ | ✅ |
| NOLOCK condicional por `@@TRANCOUNT` | N/A | N/A | ✅ | ✅ |
| Sin `SELECT *` | N/A | N/A | ✅ | ✅ |
| Constraints nombradas (PK/FK/DF/UQ/IX) | ✅ | ✅ | N/A | N/A |
| GRANT `_ro` solo `_Select`, `_wr` cualquiera | N/A | N/A | ✅ | ✅ |
| GRANT en batch separado (post `GO`) | N/A | N/A | ✅ | ✅ |
| FKs e índices idempotentes (IF NOT EXISTS) | ⚠️ | ⚠️ | N/A | N/A |
| Seed `IdUsuarioAlta` en `DocumentosTipos` | ⚠️ | N/A | N/A | N/A |

---

## 8. Observaciones / Issues

### ⚠️ Issue 1 — FKs e índices sin guarda de idempotencia (014 y 015) — Medio

Los `ALTER TABLE ADD CONSTRAINT` y `CREATE INDEX` están fuera del bloque `IF OBJECT_ID IS NULL`. Si el script se re-ejecuta con la tabla ya existente, los FKs e índices fallarán con "already exists".

**Corrección mínima** — envolver cada FK en:
```sql
IF NOT EXISTS (SELECT 1 FROM sys.foreign_keys WHERE name = 'FK_...' AND parent_object_id = OBJECT_ID(N'[Core].[DocumentosSolicitud]'))
BEGIN
    ALTER TABLE [Core].[DocumentosSolicitud] WITH CHECK ADD CONSTRAINT [FK_...] ...
END
GO
```
Y cada índice en:
```sql
IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = 'IX_...' AND object_id = OBJECT_ID(N'[Core].[DocumentosSolicitud]'))
BEGIN
    CREATE NONCLUSTERED INDEX [IX_...] ...
END
GO
```

### ⚠️ Issue 2 — Seed de `DocumentosTipos` sin `IdUsuarioAlta` (014) — Bajo

```sql
INSERT INTO [Parametrizaciones].[DocumentosTipos] ([Nombre], [Titulo])
VALUES ('ContratoGarantiaFinancieraARS', '...');
```

Si `IdUsuarioAlta` es `NOT NULL` en `DocumentosTipos`, el INSERT falla. Verificar esquema de esa tabla; si corresponde, agregar `IdUsuarioAlta = 1` al INSERT.

### ℹ️ Observación — `IdDocumentoFuente=3` e `IdDocumentoForma=1` hardcodeados en `InicializarConjunto_Insert`

Los valores `3 (Contractual)` y `1 (Digital)` se insertan hardcodeados. Funciona bien si esos IDs son estables en la tabla de parámetros; si cambian en algún ambiente, el SP insertaría fuente/forma incorrectos. Considerar resolverlos por código/nombre si se requiere más robustez.

### ℹ️ Observación — `ConjuntoCompleto_Select` cuenta todos los documentos activos de la solicitud

El SP no filtra por los 3 tipos obligatorios específicos; cuenta todos los docs activos. Si en el futuro se agregan documentos adicionales a la solicitud (no contractuales) que también tengan entrada en `FirmasDocumentosSolicitud`, el conteo cambiaría. Por ahora es correcto dado que `InicializarConjunto_Insert` inserta exactamente 3 docs.

---

## 9. Veredicto

**✅ Aprobado con observaciones menores.**

La implementación cubre el 100% de los criterios de aceptación y reglas de negocio de la US. El modelo de datos es correcto y extensible. Los SPs siguen el estándar Garantizar. Los dos issues identificados son de robustez operativa (re-ejecución del script), no afectan la funcionalidad en primera ejecución.

**Recomendación:** Corregir Issue 1 (idempotencia de FKs e índices) antes de mergear, especialmente si el pipeline de DbUp puede re-intentar scripts fallidos.
