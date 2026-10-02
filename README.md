# ⚙️ Análisis Funcional y Pruebas de Integración en ERP (SAP B1)

Este repositorio documenta la investigación, detección de causa raíz (RCA) y propuesta de solución técnica para errores de validación lógica dentro del `TransactionNotification` (TN) de SAP Business One. Demuestra la aplicación de metodologías de QA Funcional y pruebas de integración backend en entornos de misión crítica.

---

## 📌 Caso de Estudio 1: Desfasaje de IDs en Stored Procedure (Validación de OTs)

### 📋 Información General
* **Objeto Afectado:** Transferencia de Stock (`OWTR` - `object_type 67`)
* **Componente:** `SBO_SP_TransactionNotification`
* **Sucursal / BPLId:** 14 (Matriz)
* **Categoría:** Error de Validación / Falso Positivo (Código de Error 134)

### 🔍 Análisis de Causa Raíz (Root Cause Analysis)
1. **Inconsistencia de Mapeo en TN:** La regla de validación en el Stored Procedure evalúa la correspondencia estricta entre el tipo de traslado (`OWTR.U_TIPOTRASLADO`) y el tipo de llamada de la Orden de Trabajo / Servicio (`OSCL.callType`).
2. **Desfasaje en Tabla Maestra (`OSCT`):** Históricamente la base de datos sufrió cambios en su operativa por la evolución de la empresa:
   * `callTypeID = 1` → ARMADO DE CARRETAS
   * `callTypeID = 10` → REACONDICIONAMIENTO
3. **Fallo Detectado (Bug de Código Duro):** El Stored Procedure tenía asignado de forma fija (*hardcoded*) que las transferencias `REACO` debían validar contra `callType = 1`, provocando un falso positivo bloqueante (Error 134) ya que el ID real en producción migró a `10`. Los usuarios utilizaban "atajos" operativos (dejar el campo en blanco) para saltar el bloqueo, afectando la integridad del módulo de costos.

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

---

## 📌 Caso de Estudio 2: Error (739) – Truncamiento de Variable por Excepción de Activos Fijos

### 📋 Información General
* **Objeto Afectado:** Presupuesto / Oferta de Ventas (`OQUT` / `object_type 23`)
* **Componente:** `SBO_SP_TransactionNotification`
* **Categoría:** Bug de Backend / Defecto de Diseño por Cobertura Incompleta (Código de Error 739)

### 🔍 Análisis de Causa Raíz (Root Cause Analysis)
1. **El Origen del Cambio:** Se introdujo una funcionalidad nueva en el `TransactionNotification` para validar que las unidades de flota cargadas en los presupuestos de taller tuvieran un número de stock legal en la tabla maestra `OITM`.
2. **Definición Incompleta del Requerimiento:** El análisis inicial asumió que el sistema solo procesaría vehículos para la venta tradicional, identificados con el formato estándar `C18654` (6 caracteres). Basado en esto, se declaró la variable interna limitando su longitud: `DECLARE @Stock VARCHAR(6)`.
3. **Mecanismo de la Falla (La Excepción del Negocio Omitida):** La operativa de la empresa contempla que algunos vehículos de flota se registren como artículos estándar, mientras que otros se clasifican contablemente como **Activos Fijos** bajo el código especial `AF00210` (7 caracteres: dos letras y 5 números). Al no relevarse esta regla contable en el diseño del TN, el sistema truncaba el código de 7 dígitos a 6, intentando buscar un artículo deformado (`AF0021`) en `OITM` y bloqueando la transacción por falso positivo: *"DEBE CARGAR UN NRO DE STOCK VALIDO"*.

### 🧪 Lección Aprendida para QA (Análisis de Impacto)
Este caso demuestra la importancia crítica de realizar **Análisis de Impacto** y **Pruebas de Regresión** exhaustivas. No se debe probar únicamente el "camino feliz" o el formato principal de datos; se deben mapear todas las excepciones del catálogo (como la doble naturaleza de los activos fijos) vigentes en la operativa corporativa.

### ✅ Acción Correctiva (Soporte ↔️ SAPDEV)
Se elevó el informe técnico detallando el desajuste de longitud y formatos. El equipo de desarrollo (SAPDEV) aplicó la corrección ampliando la longitud de la variable a un tamaño seguro en el Stored Procedure, restituyendo la fluidez en la facturación del taller.

---

## 💡 Habilidades Técnicas Demostradas
* **Database Testing:** Depuración de Stored Procedures y análisis estructural en SQL Server.
* **Root Cause Analysis (RCA):** Diagnóstico profundo backend orientado a la resolución de bloqueos operativos.
* **Business Analysis:** Comprensión e integración de reglas complejas de negocio (Logística, Taller y Contabilidad de Costos / Activos Fijos).
* **Gestión del Conocimiento (Notion):** Estandarización de documentación técnica para optimizar la comunicación entre Soporte y Desarrollo.
