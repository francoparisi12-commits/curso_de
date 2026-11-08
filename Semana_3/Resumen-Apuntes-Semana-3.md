# Semana 3 -- Arquitectura Medallion y Delta Lake

> Bronze/Silver/Gold en profundidad, Delta Lake (ACID, Time Travel), formatos de archivo, ingesta Full vs Incremental, y primer pipeline real.

---

## Tabla de contenidos

1. Arquitectura Medallion: Bronze, Silver y Gold
2. Delta Lake, Lakehouse y Transacciones ACID
3. Formatos de Archivo: CSV vs Parquet vs Delta
4. Ingesta: Full Load, Incremental, Batch y Streaming
5. Pipeline Bronze -> Silver: Transformaciones
6. Nomenclatura, Metadata y DDL

---

## 1. Arquitectura Medallion: Bronze, Silver y Gold

La arquitectura Medallion es un **patrón de diseño**, no una tecnología. Es el estándar actual para organizar datos en capas de menor a mayor valor.

**Bronze** --- Datos crudos exactamente como llegan. No se hacen transformaciones de negocio. Bronze es nuestro seguro para **reprocesar** cuando algo falla --- es un proceso irreversible si transformamos acá.

**Silver** --- Datos limpios, validados y estandarizados. Es la **base confiable** donde ML y Analytics van a ver datos granulares antes de las agregaciones.

**Gold** --- Datos agregados, KPIs, campos calculados y modelado dimensional. Es donde el negocio encuentra valor.

Las capas son **dependientes**: Gold <- Silver <- Bronze. Esto habilita el **linaje**: recorrer datos hacia atrás para encontrar dónde se rompió una transformación.

> Pueden existir variantes: `Landing` antes de Bronze, `Semantic` después de Gold. El concepto siempre es: crudos -> limpios -> listos para consumir.

---

## 2. Delta Lake, Lakehouse y Transacciones ACID

**Delta Lake** está basada en **Parquet** pero agrega una capa de abstracción:

- **Transacciones ACID** --- Atomicidad, Consistencia, Aislamiento, Durabilidad
- **Time Travel** --- Versionado de datos, llamar a versiones anteriores
- **Schema Evolution** --- El esquema puede cambiar sin intervención manual
- **Schema Enforcement** --- Rechaza datos con esquema incorrecto
- **Batch + Streaming unificado** --- En la misma plataforma

El **Lakehouse** es el concepto: Data Warehouse + Data Lake integrados. Delta Lake es la tecnología que lo habilita.

> Pregunta común de entrevista: *¿Qué es el Lakehouse? ¿Qué es Delta Lake? ¿Cómo se relacionan?*

---

## 3. Formatos de Archivo: CSV vs Parquet vs Delta

**CSV/JSON** --- Legibles por humanos, se leen fila por fila, baja compresión. Ideales para ingesta en Bronze.

**Parquet** --- Formato **columnar** y binario. Se lee columna a columna, optimiza consultas analíticas. No es legible por humanos.

**Delta** = Parquet + superpoderes. Transacciones ACID, Time Travel, Schema Evolution, metadata inteligente. Cuando creás tabla sin especificar formato, Databricks usa **Delta por defecto**.

| Formato | Lectura | Compresión | Uso típico |
|---------|---------|------------|------------|
| CSV/JSON | Fila por fila | Baja | Ingesta en Bronze |
| Parquet | Columnar | Alta | Bronze/Silver/Gold |
| Delta | Columnar + metadata | Alta + ACID | Recomendado en todo |

---

## 4. Ingesta: Full Load, Incremental, Batch y Streaming

**Full Load** --- Carga completa de todos los datos. Simple pero ineficiente para tablas grandes. Para migraciones iniciales o tablas pequeñas.

**Incremental** --- Solo datos nuevos o modificados (la "diferencia"). Requiere **watermarks** (timestamps) para comparar. Preferido para tablas grandes.

**Batch** --- Cargas programadas en lotes periódicos: cada 15 min, 1 hora, 1 día. Ejemplo: cauciones financieras con **cron** en Databricks.

**Streaming** --- Flujo continuo de baja latencia (cada segundos). También llamado **micro-batch**.

> La diferencia fundamental entre Batch y Streaming es la **latencia**, no el concepto.

---

## 5. Pipeline Bronze -> Silver: Transformaciones

Pipeline usando **CTEs** para organizar cada paso:

**1. Retomar limpieza de Semana 2:** Recrear la cadena `propiedades_clean` -> `propiedades_clean_2` -> `bronze_EDA` como punto de partida
**2. Tipado:** `CAST()` a tipos finales (DECIMAL(15,2), INT, BOOLEAN) desde `bronze_EDA`
**3. Deduplicación:** `ROW_NUMBER() OVER (PARTITION BY precio, url)` para detectar duplicados
**4. Campos derivados:** `partido` (municipio estandarizado) y `region` (4 regiones: capital federal, gba zona norte/oeste/sur) a partir del valor crudo de Bronze. También `precio_por_m2 = precio / metros_cuadrados`
**5. INSERT** en Silver con todos los pasos encadenados

> Validación: Bronze siempre debe tener **más registros** que Silver (se filtraron inválidos).

---

## 6. Nomenclatura, Metadata y DDL

**Nomenclatura** --- Cada tabla y campo debe ser **tan claro que la documentación sea la propia nomenclatura**.
- Sufijos/prefijos para capas: `propiedades_bronze`, `propiedades_silver`
- Nombres largos pero claros > nombres cortos y crípticos
- No cambiar nombres en producción: si 200 procesos corren queries sobre una tabla, renombrarla rompe todo

**Metadata de auditoría:**
- `timestamp_insercion`: cuándo se insertó
- `fuente`: de dónde vienen los datos
- `version`: para trackear cambios

**DDL con COMMENT:**
```sql
CREATE TABLE silver.propiedades (
  precio DECIMAL(15,2) COMMENT 'Precio publicado, > 0',
  moneda STRING COMMENT 'ARS o USD',
  antiguedad INT COMMENT '999 = no disponible'
);
```
El DDL se versiona con **GitHub** para trackear cambios de estructura.

---

## Conceptos clave de la semana

- **Arquitectura Medallion: Bronze, Silver y Gold:** La arquitectura Medallion es un patrón de diseño, no una tecnología. Es el estándar actual para organizar datos en capas de menor a mayor valor.
- **Delta Lake, Lakehouse y Transacciones ACID:** Delta Lake está basada en Parquet pero agrega una capa de abstracción:
- **Formatos de Archivo: CSV vs Parquet vs Delta:** CSV/JSON — Legibles por humanos, se leen fila por fila, baja compresión. Ideales para ingesta en Bronze.
- **Ingesta: Full Load, Incremental, Batch y Streaming:** Full Load — Carga completa de todos los datos. Simple pero ineficiente para tablas grandes. Para migraciones iniciales o tablas pequeñas.
- **Pipeline Bronze -> Silver: Transformaciones:** Pipeline usando CTEs para organizar cada paso:
- **Nomenclatura, Metadata y DDL:** Nomenclatura — Cada tabla y campo debe ser tan claro que la documentación sea la propia nomenclatura.