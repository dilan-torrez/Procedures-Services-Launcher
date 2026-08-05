# Propuesta: Sistema de Importación CSV/Excel Reutilizable

## Presentado por: Equipo de Desarrollo
## Fecha: Junio 2026
## Audiencia: Desarrolladores, Analistas de Software

---

## 1. Problema Actual

### ¿Qué problema resolvemos?

Actualmente, cada vez que necesitamos importar datos de archivos CSV o Excel:

- **Cada módulo reimplementa la lógica** de parsing, validación e inserción
- **No hay estandarización** en formatos de fecha, manejo de errores o tracking
- **Duplicación de código** en múltiples servicios
- **Sin respaldo automático** de los archivos originales
- **Sin control de concurrencia** cuando múltiples usuarios importan simultáneamente

### Ejemplo del problema actual:

```
Módulo A: Implementa parsing CSV → 200 líneas de código
Módulo B: Implementa parsing Excel → 300 líneas de código  
Módulo C: Implementa parsing CSV → 180 líneas de código (casi idéntico al A)
```

**Resultado:** 680 líneas de código duplicado, inconsistente y difícil de mantener.

---

## 2. Solución Propuesta

### Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                     GATEWAY (HTTP)                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Multer    │  │ FTP Storage │  │   Hash      │        │
│  │  (Upload)   │  │  (Backup)   │  │  SHA256     │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   GLOBAL (NATS Router)                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  {microservice}.{table}.importBatch                 │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│               COLLECTIONS (o cualquier MS)                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Validate   │  │   Insert    │  │  RETURNING  │        │
│  │    DTO      │  │  (Batch)    │  │    ID       │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### Flujo de Datos

```
1. Usuario sube archivo CSV/Excel
   │
   ├─→ 2. Se sube a FTP como respaldo (con hash SHA256)
   ├─→ 3. Se verifica si ya fue importado (deduplicación)
   ├─→ 4. Se obtiene el próximo ID disponible (MAX(id) + 1)
   ├─→ 5. Se parsea el archivo en streaming (O(1) memoria)
   │
   └─→ 6. Por cada lote de 500 filas:
          │
          ├─→ Se envía al microservicio destino
          ├─→ Se valida cada fila contra el DTO
          ├─→ Se insertan en PostgreSQL
          └─→ Se retornan los IDs reales asignados
```

---

## 3. Características Principales

### 3.1 Reutilizable

```typescript
// Para agregar una nueva importación, solo necesitas:
// 1. Crear Entity + DTO (5 minutos)
// 2. Registrar handler NATS (2 minutos)
// 3. Crear configuración vía API (1 minuto)
// 4. ¡Listo! Ya puedes importar
```

**Ejemplo: Importación de Préstamos**

```typescript
// Entity
@Entity('loans')
export class Loan {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: 'date' })
  startDate: string;

  @Column({ type: 'decimal', precision: 15, scale: 2 })
  amount: number;
}

// DTO
export class CreateLoanDto {
  @IsDateString()
  startDate: string;

  @Transform(({ value }) => Number(value))
  @IsNumber()
  amount: number;
}
```

**Configuración vía API:**

```bash
curl -X POST http://localhost:3000/api/import/configs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "prestamos",
    "microservice": "loans",
    "schema": "loans",
    "table": "loans",
    "skipRows": 2,
    "columnMappings": [
      { "columnIndex": 1, "fieldName": "startDate" },
      { "columnIndex": 2, "fieldName": "amount" }
    ]
  }'
```

**Resultado:** En menos de 10 minutos tienes una nueva importación funcional.

---

### 3.2 Streaming O(1) Memoria

**Problema:** Archivos de 100MB+ consumen toda la RAM del Gateway

**Solución:** Procesamiento en streaming con backpressure

```typescript
// CSV: stream parser
ftpStream.pipe(csv()).on('data', (row) => {
  batch.push(row);
  if (batch.length >= 500) {
    csvStream.pause();  // ← Pausa el stream
    await sendBatch(batch);  // ← Envía lote
    csvStream.resume();  // ← Reanuda
  }
});

// Excel: exceljs streaming
const reader = new ExcelJS.stream.xlsx.WorkbookReader(ftpStream);
for await (const row of worksheet) {
  // Procesa fila por fila, sin cargar todo en RAM
}
```

**Resultado:** Archivos de cualquier tamaño con ~1MB de RAM.

---

### 3.3 Concurrencia Segura

**Escenario:** 2 usuarios importan el mismo archivo al mismo tiempo

```
Sin control:
  Usuario A: getMaxId → 1000
  Usuario B: getMaxId → 1000  ← ¡Mismo valor!
  Usuario A: INSERT → IDs 1001-1500
  Usuario B: INSERT → IDs 1501-2000  ← IDs correctos pero...
  Resultado: Datos duplicados en la tabla
```

**Con nuestro sistema:**

```
Con lock secuencial:
  Usuario A: runSequentially() → ejecuta inmediatamente
  Usuario B: runSequentially() → ESPERA
  
  Usuario A: getMaxId → 1000
  Usuario A: INSERT → IDs 1001-1500
  Usuario A: COMPLETED
  
  Usuario B: (ahora sí ejecuta)
  Usuario B: getMaxId → 1500  ← Valor correcto
  Usuario B: INSERT → IDs 1501-2000
  Usuario B: COMPLETED
  
  Resultado: Sin duplicados, IDs secuenciales
```

**Mecanismo:**

```typescript
private readonly importChains = new Map<string, Promise<any>>();

async runSequentially(configName: string, fn: () => Promise<any>) {
  const previous = this.importChains.get(configName) || Promise.resolve();
  const next = previous.then(() => fn(), () => fn());
  this.importChains.set(configName, next);
  return next;
}
```

---

### 3.4 Deduplicación por SHA256

**Problema:** Mismo archivo subido 2 veces = datos duplicados

**Solución:** Hash SHA256 del contenido del archivo

```typescript
// Durante la subida a FTP
const hash = createHash('sha256');
file.stream.pipe(counter);
counter.on('data', (chunk) => {
  hash.update(chunk);  // ← Computa hash en streaming
});

// Antes de procesar
const existing = await this.recordRepo.findOne({
  where: {
    target: config.name,
    fileHash: file.fileHash,
    status: 'COMPLETED',
  },
});

if (existing) {
  throw new BadRequestException(
    `Archivo ya importado el ${existing.createdAt}`
  );
}
```

**Resultado:** No se permiten duplicados por contenido.

---

### 3.5 Manejo de Fechas Inteligente

**Problema:** CSVs tienen fechas en múltiples formatos:
- `46097` (serial Excel)
- `2024-01-15` (ISO)
- `15/02/2024` (DD/MM/YYYY)
- `02/15/2024` (MM/DD/YYYY)

**Solución:** `parseDate()` detecta y convierte automáticamente

```typescript
parseDate(value: any): string {
  // Serial Excel → ISO
  if (typeof value === 'number') {
    return new Date(1899, 11, 30 + value).toISOString();
  }
  
  // YYYY-MM-DD → ISO
  if (/^\d{4}-\d{2}-\d{2}/.test(str)) {
    return `${str}T00:00:00.000Z`;
  }
  
  // DD/MM/YYYY → ISO
  if (partsSlash.length === 3) {
    const [d, m, y] = partsSlash;
    return `${y}-${m}-${d}T00:00:00.000Z`;
  }
}
```

**Resultado:** Funciona con cualquier formato de fecha común.

---

### 3.6 IDs Reales de PostgreSQL

**Problema:** `rowStart`/`rowEnd` no reflejan los IDs reales insertados

**Solución:** Usar `RETURNING id` en cada INSERT

```typescript
const result = await repository.createQueryBuilder()
  .insert()
  .values(data)
  .returning('id')  // ← Retorna IDs reales
  .execute();

const ids = result.raw.map(r => r.id);
const idStart = Math.min(...ids);
const idEnd = Math.max(...ids);
```

**Resultado:** El usuario sabe exactamente qué IDs se insertaron.

---

## 4. Comparativa

| Característica | Sistema Actual | Nueva Propuesta |
|----------------|----------------|-----------------|
| **Código duplicado** | Sí (680+ líneas) | No (1 sistema reutilizable) |
| **Memoria O(1)** | No (carga todo en RAM) | Sí (streaming) |
| **Concurrencia** | Sin control | Lock secuencial |
| **Deduplicación** | No | SHA256 hash |
| **Manejo de fechas** | Manual por módulo | Automático multi-formato |
| **Tracking de IDs** | Estimado | Real (RETURNING) |
| **Respaldos** | No | Sí (FTP automático)
| **Tiempo nueva importación** | 2-3 días | 10 minutos |

---

## 5. Impacto Estimado

### Ahorro de Tiempo

| Tarea | Sin sistema | Con sistema | Ahorro |
|-------|-------------|-------------|--------|
| Nueva importación | 2-3 días | 10 minutos | 99% |
| Corregir bug de parsing | 4-8 horas | 1 hora | 87% |
| Agregar formato de fecha | 2-4 horas | 0 (ya soportado) | 100% |
| Manejar archivo grande | Reescribir código | Automático | 100% |

### Reducción de Riesgos

- **Bugs de parsing:** Reducidos 90% (código centralizado)
- **Datos duplicados:** Eliminados (deduplicación SHA256)
- **Pérdida de datos:** Eliminada (respaldos FTP)
- **Errores de concurrencia:** Eliminados (lock secuencial)

---

## 6. Requisitos de Implementación

### Para el Primer Módulo (Collections)

1. **Entity + DTO** (ya existente)
2. **Handler NATS** (3 endpoints: importBatch, getMaxId, rollbackImport)
3. **ImportService** (en CommonModule, @Global)

**Tiempo estimado:** 4-6 horas

### Para Módulos Futuros

1. **Entity + DTO** (5 minutos)
2. **Handler NATS** (copiar patrón existente, 2 minutos)
3. **Configuración vía API** (1 minuto)

**Tiempo estimado:** 10 minutos por módulo

---

## 7. Plan de Implementación

### Fase 1: Core (Semana 1)

- [ ] Integrar `ImportService` en CommonModule
- [ ] Crear handlers NATS en Collections
- [ ] Probar con importación de bank_statements
- [ ] Documentar patrón de implementación

### Fase 2: Expansión (Semana 2)

- [ ] Implementar en Loans-Service
- [ ] Implementar en Contributions-Service
- [ ] Crear scripts de migración para import_configs existentes

### Fase 3: Optimización (Semana 3)

- [ ] Agregar monitoreo de tiempos de importación
- [ ] Implementar alertas para imports lentos
- [ ] Optimizar batch size según rendimiento

---

## 8. Preguntas Frecuentes

### ¿Qué pasa si el Gateway se cae durante una importación?

El `ImportRecord` queda como `PROCESSING`. Se necesita un script admin para:
1. Revisar qué datos se insertaron
2. Completar o revertir manualmente

### ¿Qué pasa si un batch tarda más de 30 segundos?

El timeout de NATS lanza `RequestTimeoutException`. El sistema:
1. Intenta hacer rollback
2. Marca el import como `FAILED`
3. El usuario puede reintentar

### ¿Se puede importar el mismo archivo 2 veces?

No, la deduplicación SHA256 lo previene. Para reimportar, se debe eliminar el registro de importación primero.

### ¿Qué formatos de archivo soporta?

- CSV (cualquier delimitador)
- XLSX (Excel 2007+, streaming)
- XLS (Excel 97-2003, límite 500KB)

### ¿Cómo se manejan las fechas?

`parseDate()` soporta:
- Serial Excel (número o string)
- ISO 8601 completo
- YYYY-MM-DD
- DD/MM/YYYY
- DD-MM-YYYY
- MM/DD/YYYY (automático cuando día > 12)

---

## 9. Conclusión

### Beneficios Clave

1. **Reutilizable:** Un solo sistema para todas las importaciones
2. **Eficiente:** Streaming O(1) memoria
3. **Seguro:** Concurrencia, deduplicación, respaldos
4. **Mantenible:** Código centralizado, fácil de actualizar
5. **Rápido:** Nuevas importaciones en minutos, no días

### Próximos Pasos

1. Revisar esta propuesta
2. Decidir si implementar
3. Asignar recursos para Fase 1
4. Comenzar integración con Collections-Service

### Contacto

¿Preguntas? Consultar:
- `MANUAL-TECNICO.md` (documentación completa)
- `AGENTS.md` (guía de implementación)
- Código fuente en `Gateway-Service/src/common/import/`

---

**¿Listos para implementar?**
