# erp-business-logic-testing
Análisis funcional y diseño de pruebas de integración para el módulo de servicios (OT) y revalorización contable en SAP B1.

# ⚙️ Análisis Funcional: Corrección de Validación en Stored Procedure (SAP B1 TN)

Este repositorio documenta la investigación, detección de causa raíz y propuesta de solución técnica para un error de validación lógica dentro del `TransactionNotification` (TN) de SAP Business One.

## 📌 Reporte del Incidente: OT Tipo Reacondicionamiento - Tipo Traslado 'REACO'

### 📋 Información General
* **Sistema:** SAP Business One (SBO)
* **Objeto Afectado:** Transferencia de Stock (`OWTR` - `object_type 67`)
* **Componente:** `SBO_SP_TransactionNotification`
* **Sucursal / BPLId:** 14 (Matriz)
* **Categoría:** Error de Validación / Falso Positivo (Código de Error 134)

---

### 🔍 Análisis de Causa Raíz (Root Cause Analysis)
1. **Descarte de Permisos:** Se confirmó que el problema **no** corresponde a falta de autorizaciones o licencias del personal operativo.
2. **Inconsistencia de Mapeo en TN:** La regla de validación en el Stored Procedure evalúa la correspondencia estricta entre el tipo de traslado (`OWTR.U_TIPOTRASLADO`) y el tipo de llamada de la OT (`OSCL.callType`).
3. **Desfasaje en Tabla Maestra (`OSCT`):** Históricamente la base de datos sufrió cambios en su operativa:
   * `callTypeID = 1` → ARMADO DE CARRETAS
   * `callTypeID = 10` → REACONDICIONAMIENTO
4. **Fallo Detectado (Bug de Código Duro):** El Stored Procedure tenía asignado de forma fija (*hardcoded*) que las transferencias `REACO` debían validar contra `callType = 1`, provocando un falso positivo bloqueante (Error 134) ya que el ID real en producción migró a `10`.

---

### 💻 Solución Propuesta (Análisis de Código SQL)

#### ❌ Código Anterior (Con Error de Lógica)
```sql
IF @object_type = '67' AND @transaction_type IN ('A', 'U')
BEGIN
    IF EXISTS (
        SELECT 1 
        FROM OWTR T0 
        INNER JOIN OSCL T1 ON T0.U_NroServ = T1.callID
        WHERE T0.DocEntry = @list_of_cols_val_tab_del
          AND T0.BPLId = 14
          AND T0.U_TIPOTRASLADO = 'REACO'
          AND T1.callType <> 1 -- ❌ ERROR: Valida contra 1 (Armado de Carretas)
    )
    BEGIN
        SET @error = 134;
        SET @error_message = 'El tipo de servicio no corresponde a la OT';
    END
END
```

####  Código Corregido (Entregado a DEV SAP)
```sql
IF @object_type = '67' AND @transaction_type IN ('A', 'U')
BEGIN
    IF EXISTS (
        SELECT 1 
        FROM OWTR T0 
        INNER JOIN OSCL T1 ON T0.U_NroServ = T1.callID
        WHERE T0.DocEntry = @list_of_cols_val_tab_del
          AND T0.BPLId = 14
          AND T0.U_TIPOTRASLADO = 'REACO'
          AND T1.callType <> 10 --  CORRECCIÓN: Valida contra 10 (Reacondicionamiento)
    )
    BEGIN
        SET @error = 134;
        SET @error_message = 'El tipo de servicio no corresponde a la OT';
    END
END
```

### 📈 Impacto Operativo y Efecto del Cambio
Al corregir la condición a `T1.callType <> 10`, cuando el usuario genera una transferencia `REACO` ligada a una OT de Reacondicionamiento, la validación se cumple correctamente, dejando pasar la transacción de forma limpia. 

Esta solución eliminó la necesidad de que los usuarios usen "atajos" operativos (dejar el campo en blanco), restaurando la integridad del módulo de costos y la trazabilidad de la empresa.

---
### 💡 Habilidades Técnicas Demostradas
* **Database Testing:** Depuración de Stored Procedures en SQL Server.
* **Root Cause Analysis:** Diagnóstico profundo backend orientado a la resolución de bloqueos.
* **Gestión del Conocimiento (Notion):** Documentación técnica estandarizada para el puente Soporte ↔️ Desarrollo.

