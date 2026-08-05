# Sistema de Importación CSV/Excel

## Arquitectura

```
Gateway (HTTP)  →  Microservicio Destino (NATS Direct)
    │                      │
    ├── FTP               ├── getMaxId (SELECT MAX(id))
    ├── ImportConfig      ├── importBatch (INSERT + RETURNING id)
    └── ImportRecord      └── rollbackImport (DELETE WHERE id IN (...))
```

**Schema:** Las tablas de importación (`import_configs`, `import_records`) viven en el schema `global` de PostgreSQL.

## Cómo agregar una nueva importación (4 pasos)

### Paso 1 — Crear la entidad y DTO en el microservicio destino

En `Collections-Service` (o el que corresponda):

```
src/<tu-modulo>/
├── entities/<tu-entidad>.entity.ts    → @Entity() con @PrimaryGeneratedColumn()
├── dto/<tu-dto>.dto.ts                → CreateDto con class-validator
├── <tu-modulo>.service.ts             → CRUD + delegar en ImportService
└── <tu-modulo>.controller.ts          → @MessagePattern('{microservice}.{table}.importBatch')
```

El `documentNumber` debe ser `number` con `@Transform`:

```ts
@Transform(({ value }) => value != null ? Number(value) : undefined)
@IsNumber()
documentNumber: number;
```

Campos string desde Excel necesitan `@Transform`:

```ts
@Transform(({ value }) => String(value ?? ''))
@IsString()
gloss: string;
```

### Paso 2 — Registrar el handler NATS

En el controlador del microservicio:

```ts
@MessagePattern('collections.mi_tabla.importBatch')
async handleImportBatch(@Payload() data: any) {
  return this.service.importBatch(data);
}

@MessagePattern('collections.getMaxId')
async handleGetMaxId(@Payload() data: { tableName: string; schema?: string }) {
  const maxId = await this.importService.getMaxId(data.tableName, data.schema);
  return { maxId };
}

@MessagePattern('collections.rollbackImport')
async handleRollbackImport(@Payload() data: { importId: number }) {
  await this.importService.rollbackImport(data.importId);
  return { rolledBack: true };
}
```

El service delega en `ImportBatchService` (`common/import/import.service.ts`):

```ts
async importBatch(data: ImportBatchDto) {
  return this.importBatchService.importBatch(
    this.repo,
    data.data,
    data.isLastBatch,
    data.importId,
  );
}
```

### Paso 3 — Crear la config vía API (no seeds)

```bash
curl -X POST http://localhost:3000/api/import/configs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "mi_importacion",
    "microservice": "collections",
    "schema": "collections",
    "table": "mi_tabla",
    "skipRows": 2,
    "startColumn": 2,
    "delimiter": ",",
    "columnMappings": [
      { "columnIndex": 2, "fieldName": "date" },
      { "columnIndex": 5, "fieldName": "operationCode" },
      { "columnIndex": 6, "fieldName": "documentNumber" },
      { "columnIndex": 14, "fieldName": "state" }
    ]
  }'
```

Usar POST para crear o actualizar (upsert por `name`). Ver todos:

```bash
curl http://localhost:3000/api/import/configs
```

### Paso 4 — Subir archivo

```bash
curl -X POST http://localhost:3000/api/import/mi_importacion \
  -F "file=@/ruta/datos.csv"
```

Respuesta exitosa:

```json
{
  "recordId": 1,
  "configName": "mi_importacion",
  "rowStart": 1001,
  "rowEnd": 1500,
  "totalRows": 500,
  "ftpPath": "/imports/mi_importacion/2026-06-03/12345_datos.csv",
  "message": "Importación mi_importacion completada. IDs 1001 a 1500 (500 registros)."
}
```

## Endpoints CRUD para import_configs

| Método | Ruta | Descripción |
|--------|------|-------------|
| POST   | `/api/import/configs` | Crear o actualizar (upsert por `name`) |
| GET    | `/api/import/configs` | Listar todas |
| GET    | `/api/import/configs/:name` | Ver una |
| PUT    | `/api/import/configs/:name` | Editar |
| DELETE | `/api/import/configs/:name` | Eliminar |

## Reglas del sistema

- **Solo imports COMPLETED persisten**: FAILED → rollback automático DELETE WHERE id IN (...)
- **IDs reales desde RETURNING id**: `rowStart`/`rowEnd` reflejan los IDs asignados por PostgreSQL
- **Streaming O(1)**: CSV y .xlsx usan streams, sin cargar todo en RAM
- **.xls limitado a 500KB**: SheetJS necesita carga completa en RAM
- **Patrón NATS dinámico**: `{microservice}.{table}.importBatch` — derivado de import_configs
- **Timeout 30s**: NatsService lanza RequestTimeoutException si el microservicio no responde
- **Archivo original a FTP**: siempre se sube como respaldo antes de parsear
- **Lock secuencial por config name**: `runSequentially()` garantiza que 2 imports con el mismo `name` NO se ejecuten en paralelo
- **Deduplicación por SHA256**: antes de procesar, verifica si ya existe un COMPLETED con el mismo `target` + `fileHash`
- **startColumn**: `mapRowByConfig` aplica slice antes del mapeo (útil para archivos con columnas basura iniciales)
- **Auth guards**: todos los endpoints de import requieren `@UseGuards(AuthGuard)`
- **SQL injection protection**: `validateSqlIdentifier()` valida nombres de tablas/schemas con regex

## stack.key

El patrón `{microservice}.{table}.importBatch` se deriva así:
- `microservice`: de `import_configs.microservice` (ej: `collections`)
- `table`: de `import_configs.table` (ej: `bank_statements`)
- Resultado: `collections.bank_statements.importBatch`

No se usa `target` ni `routeMap`. El Global deriva el patrón dinámicamente desde los datos del lote.

## Solución de problemas

**Error: "(0 , csv_parser_1.default) is not a function"**
- Usar `import csv from 'csv-parser'` (default import) con `esModuleInterop: true` en tsconfig

**Error: "Microservice Unavailable"**
- Verificar que el microservicio destino esté corriendo: `docker compose ps`
- Verificar que el handler NATS esté registrado en el microservicio destino

**Error: "documentNumber must be a number"**
- Agregar `@Transform(({ value }) => value != null ? Number(value) : undefined)` en el DTO

**Error: "getaddrinfo ENOTFOUND nats-server-dev"**
- Restart nats: `docker compose restart nats-server-dev`

**Error: "date must be a valid ISO 8601 date string"**
- El CSV tiene fechas en formato DD/MM/YYYY o serial Excel como string
- El `parseDate` en Collections-Service maneja múltiples formatos automáticamente
- Verificar que el `parseDate` esté retornando formato ISO completo: `YYYY-MM-DDTHH:mm:ss.sssZ`

**Error: "state must be one of: CONCILIADO, NO_CONCILIADO"**
- Verificar que el `columnIndex` para `state` apunte a la columna correcta
- El CSV tiene 14 columnas: si el mapeo usa `columnIndex: 14`, debe haber al menos 14 columnas en los datos

**Error: POST a `/api/import/configs` retorna "Archivo requerido"**
- Conflicto de rutas: `import/:name` intercepta `/import/configs`
- Solución: registrar `ImportConfigController` antes de `ImportController` en `ImportModule`

**La importación toma datos de columnas incorrectas**
- Verificar `skipRows` para asegurar que el encabezado se salte correctamente
- Verificar `columnIndex` en los mappings: son 1-indexados (primera columna = 1)
- Ejemplo: si el CSV tiene encabezado en fila 3, usar `skipRows: 3`
