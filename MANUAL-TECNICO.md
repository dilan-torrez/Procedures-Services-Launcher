# Manual Técnico — Sistema de Importación CSV/Excel

## 1. Entity + DTO (lo mínimo que necesita una tabla)

### Entity

Define la estructura de la tabla en la base de datos. El `documentNumber` usa `bigint` para manejar números grandes.

```ts
// bank-statement.entity.ts
@Entity({ schema: 'collections', tableName: 'bank_statements' })
export class BankStatement {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: 'date' })
  date: string;

  @Column({ type: 'bigint' })
  documentNumber: number;

  @Column({ type: 'text' })
  gloss: string;

  @Column({ type: 'numeric', precision: 14, scale: 2 })
  credits: number;

  @Column({ type: 'text', nullable: true })
  state: string;
}
```

### DTO

Valida cada fila **antes** de insertarla. El sistema convierte cada fila del archivo a una instancia del DTO y ejecuta los decoradores `class-validator`.

```ts
// bank-statement.dto.ts
export class CreateBankStatementDto {
  @IsDateString()
  date: string;

  @Transform(({ value }) => String(value ?? ''))
  @IsString()
  operationCode: string;

  @Transform(({ value }) => value != null ? Number(value) : undefined)
  @IsNumber()
  documentNumber: number;

  @Transform(({ value }) => String(value ?? ''))
  @IsString()
  gloss: string;

  @IsOptional()
  @Transform(({ value }) => value != null ? String(value) : undefined)
  @IsString()
  transferredAccount?: string;

  @Transform(({ value }) => value != null ? Number(value) : undefined)
  @IsNumber()
  credits: number;

  @IsOptional()
  @IsEnum(ConciliationState)
  state?: ConciliationState;
}
```

**`@Transform` es obligatorio** cuando el archivo (Excel/CSV) entrega el valor en un tipo distinto al que espera el DTO:
- CSV manda números como `string` → el DTO espera `number` → `@Transform(({ value }) => Number(value))`
- CSV manda strings como `string` → el DTO espera `string` → `@Transform(({ value }) => String(value))`
- CSV manda fechas como `string` → el DTO espera `string` ISO → `parseDate()` lo convierte

### ¿Se puede usar el Entity como DTO?

Técnicamente sí, si se le agregan decoradores `class-validator` al entity. Pero **no se recomienda** porque:
- El entity tiene decoradores de TypeORM (`@Column`, `@JoinColumn`) que no aportan nada en validación
- El DTO puede tener validaciones distintas a la estructura de la tabla (`@IsOptional` en campos que en BD son `NOT NULL` porque tienen default)
- Mezclar responsabilidades dificulta el mantenimiento

Si igual se quiere usar el entity como DTO, el método `importBatch` acepta un `dtoClass` opcional. Si no se pasa, **no valida nada** e inserta directo.

---

## 2. Concurrencia (varias importaciones al mismo tiempo)

### Lock secuencial por config name

Cada importación usa `runSequentially(name, fn)` que garantiza que dos importaciones con el **mismo config name** NO se ejecuten en paralelo: la segunda espera a que la primera termine (éxito o error).

```ts
// En ImportService (Gateway)
private readonly importChains = new Map<string, Promise<any>>();

async runSequentially(configName: string, fn: () => Promise<any>): Promise<any> {
  const previous = this.importChains.get(configName) || Promise.resolve();
  const next = previous.then(() => fn(), () => fn()); // Siempre ejecuta fn()
  this.importChains.set(configName, next);
  return next;
}
```

**Importante:** Si la anterior falla, la siguiente igual se ejecuta (no se bloquea permanentemente).

### Escenario: 2 importaciones usan el mismo Config A

```
Tiempo ─────────────────────────────────────────────►

Config A, Import 1:
  runSequentially → ejecuta inmediatamente
  getMaxId → 1000
  batch 1 → INSERT → IDs 1001-1500
  batch 2 → INSERT → IDs 1501-2000
  COMPLETED → responde con IDs reales

Config A, Import 2:
  runSequentially → ESPERA a que Import 1 termine
  ( Import 1 termina )
  getMaxId → 2000 (valor correcto)
  batch 1 → INSERT → IDs 2001-2500
  batch 2 → INSERT → IDs 2501-3000
  COMPLETED → responde con IDs reales
```

**¿Qué pasa con los IDs?**

No hay intercalado porque la segunda importación espera a que la primera termine. `getMaxId` ve el valor real después de que los datos ya fueron insertados.

### Escenario: Config A y Config B importando al mismo tiempo

Son flujos completamente independientes. Cada uno tiene:
- Su propia configuración (`import_configs`)
- Su propio pattern NATS (`collections.bank_statements` vs `collections.loans`)
- Su propia tabla destino (`bank_statements` vs `loans`)
- Su propio `batchTracker` (key = importId, único)

No hay ningún punto de contención entre ellos. Se ejecutan en paralelo sin interferencia.

### Escenario: un mismo archivo se sube 2 veces

**Deduplicación por SHA256 hash:**

```ts
// En processFileInternal (Gateway)
if (file.fileHash) {
  const existing = await this.recordRepo.findOne({
    where: {
      target: config.name,
      fileHash: file.fileHash,
      status: ImportStatus.COMPLETED,
    },
  });
  if (existing) {
    throw new BadRequestException(
      `El archivo "${file.originalname}" ya fue importado correctamente el ${existing.createdAt?.toISOString().split('T')[0]}.`
    );
  }
}
```

El hash se computa durante el streaming a FTP (sin lectura adicional del archivo):

```ts
// En ftp-storage.ts
const hash = createHash('sha256');
const counter = new Transform({
  transform(chunk, _encoding, callback) {
    fileSize += chunk.length;
    hash.update(chunk);  // ← computa hash durante el pipe
    callback(null, chunk);
  },
});
file.stream.pipe(counter);

ftpService.uploadStream(counter, ftpPath).then(() => {
  cb(null, { ftpPath, fileHash: hash.digest('hex'), ... });
});
```

**Reglas de deduplicación:**
- Solo aplica si el registro anterior tiene `status = COMPLETED`
- Solo compara `target` (nombre del config) + `fileHash` + `status`
- Si el archivo se importó en otro target (distinto config name), no hay deduplicación
- Para reimportar, el admin debe eliminar el registro de importación primero

---

## 3. Formato de fechas en CSV/Excel

### parseDate en Collections-Service

El `ImportService` en Collections-Service tiene una función `parseDate` que maneja múltiples formatos de entrada:

```ts
parseDate(value: any): string {
  if (!value) return '';
  // Strings numéricos (ej: "46097" desde CSV)
  if (typeof value === 'string' && /^\d+(\.\d+)?$/.test(value.trim())) {
    value = Number(value);
  }
  if (typeof value === 'number') {
    // Serial de Excel → ISO 8601
    const excelEpoch = new Date(Date.UTC(1899, 11, 30));
    const date = new Date(excelEpoch.getTime() + value * 86400000);
    return date.toISOString();
  }
  if (value instanceof Date) {
    return value.toISOString();
  }
  const str = String(value).trim();
  // YYYY-MM-DD... → completar con T00:00:00.000Z
  if (/^\d{4}-\d{2}-\d{2}/.test(str)) {
    if (str.includes('T')) return str;
    return `${str}T00:00:00.000Z`;
  }
  // DD/MM/YYYY o DD-MM-YYYY
  // MM/DD/YYYY (si primer valor > 12, asumimos DD-MM-YYYY)
  return str;
}
```

### Formatos soportados

| Formato | Ejemplo | Resultado |
|---------|---------|-----------|
| Serial Excel (número) | `46097` | `2026-03-14T00:00:00.000Z` |
| Serial Excel (string) | `"46097"` | `2026-03-14T00:00:00.000Z` |
| ISO 8601 completo | `2024-01-15T10:30:00.000Z` | `2024-01-15T10:30:00.000Z` |
| YYYY-MM-DD | `2024-01-15` | `2024-01-15T00:00:00.000Z` |
| DD/MM/YYYY | `15/01/2024` | `2024-01-15T00:00:00.000Z` |
| DD-MM-YYYY | `15-01-2024` | `2024-01-15T00:00:00.000Z` |
| MM/DD/YYYY (cuando día ≤ 12) | `01/15/2024` | `2024-01-15T00:00:00.000Z` |
| Objeto Date | `new Date(2024, 0, 15)` | `2024-01-15T00:00:00.000Z` |

### ¿Por qué importa el formato de fecha?

El campo `date` en el DTO tiene `@IsDateString()` que valida ISO 8601 estricto:
- `2024-01-15` → ❌ (sin timezone)
- `2024-01-15T00:00:00.000Z` → ✅

Si el CSV tiene fechas en otro formato (DD/MM/YYYY, MM/DD/YYYY), `parseDate` las convierte automáticamente.

### Validación de fechas inválidas

`parseDate` valida que las fechas sean reales antes de convertirlas:
- `2024-13-45` → ❌ Error controlado: "Fecha inválida"
- `2024-02-30` → ❌ Error controlado: "Fecha inválida" (29 feb en año no bisiesto)
- `2024-01-15` → ✅ Conversión exitosa

Esto previene que fechas inválidas se inserten en la base de datos.

---

## 4. Pérdida de conexión

### FTP falla durante la subida

Ocurre antes de parsear. No hay datos en riesgo. El Gateway lanza `BadRequestException`.

### NATS timeout durante sendBatch

El `NatsService.firstValue()` tiene timeout de **30 segundos**. Si no hay respuesta:
1. Gateway captura el error
2. Llama a `rollbackImport` vía NATS
3. Se actualiza el `ImportRecord` a `FAILED`

Si el timeout ocurre en el primer batch, no hay nada que rollbackear (aún no se insertó nada).

**¿Por qué es útil este timeout?**

1. **Evita bloqueos indefinidos:** Si un microservicio se cuelga, el Gateway no queda colgado para siempre
2. **Permite rollback automático:** Si no hay respuesta en 30s, se intenta deshacer los cambios
3. **Protección contra bucles muertos:** Evita que una cadena de fallos se propague indefinidamente

**Cuándo sería problemático:**

1. **Lotes muy grandes:** Si un batch de 500 filas tarda más de 30s en procesarse
2. **Microservicio lento:** Si el Collections-Service está sobrecargado
3. **Red lenta:** Si hay latencia alta entre Gateway y Collections

**Recomendación:** Mantener el timeout de 30s es razonable para la mayoría de casos. Si se necesita procesar lotes muy grandes, se puede aumentar a 60s o 90s.

### Gateway se cae durante processFile

El `batchTracker` está en el Collections-Service, no en el Gateway. Si el Gateway se cae:
- Los lotes ya enviados al Collections quedaron insertados
- El `ImportRecord` queda como `PROCESSING` (no COMPLETED ni FAILED)
- No hay forma automática de retomar o deshacer
- Un admin debe revisar manualmente

### Collections-Service se cae durante importBatch

- Si fue antes del INSERT → el batch falla, Gateway recibe error, llama a rollback
- Si fue después del INSERT pero antes de responder → el Gateway timeout, llama a rollback
- Si rollback también falla (Collections caído) → datos huérfanos + record FAILED

---

## 5. Arquitectura y flujo de datos

```
Gateway (HTTP)
  │
  ├─ 1. Recibe archivo → lo sube a FTP (respaldo)
  │     └─ Computa SHA256 durante streaming (sin carga adicional)
  ├─ 2. Crea ImportRecord (PROCESSING) con fileHash
  ├─ 3. Chequeo de duplicados: ¿existe COMPLETED con mismo target + fileHash?
  ├─ 4. getMaxId → NATS → Microservicio Destino (MAX(id) real)
  ├─ 5. Parsea CSV/XLSX en streaming
  ├─ 6. Por cada lote de 500 filas:
  │      sendBatch → NATS → Microservicio Destino
  │      └─ Microservicio: valida DTO → parseDate → INSERT → RETURNING id
  │      └─ Gateway: guarda IDs reales en ImportRecord
  └─ 7. COMPLETED → responde rowStart / rowEnd reales
```

### ¿Por qué se envía directo al microservicio?

El Gateway construye el patrón NATS `{microservice}.{table}.importBatch` directamente desde la config de importación. Esto elimina el saltro innecesario por Global, reduciendo latencia y complejidad.

### ¿Cómo funciona el lock secuencial?

`runSequentially` crea una cadena de promesas por config name. Cada nueva importación se encadena a la anterior:

```ts
// Ejecución secuencial por config name
const previous = importChains.get('extractos_bancarios') || Promise.resolve();
const next = previous.then(() => fn(), () => fn()); // Siempre ejecuta fn()
importChains.set('extractos_bancarios', next);
return next;
```

El Map `importChains` es privado del servicio y se limpia automáticamente cuando la promesa se resuelve.

---

## 6. batchTracker (memoria RAM)

El tracker en Collections-Service guarda **los IDs exactos** de cada lote:

```ts
Map<importId, Array<{ ids: number[] }>>
// Ej: Map(2) {
//   5 => [ { ids: [1001, 1002, ..., 1500] } ],
//   6 => [ { ids: [1501, 1502, ..., 2000] } ]
// }
```

**No guarda rangos** (`1001-1500`) porque si entre medio se insertaron datos manuales en la tabla, un DELETE con `WHERE id BETWEEN 1001 AND 1500` borraría datos legítimos. Guarda el array explícito de IDs.

Al terminar (`isLastBatch: true`), se elimina la entrada del tracker.

---

## 7. Column mapping (import_configs)

### Estructura del config

```json
{
  "name": "extractos_bancarios",
  "microservice": "collections",
  "schema": "collections",
  "table": "bank_statements",
  "skipRows": 3,
  "startColumn": 2,
  "delimiter": ",",
  "columnMappings": [
    { "columnIndex": 2, "fieldName": "date" },
    { "columnIndex": 5, "fieldName": "operationCode" },
    { "columnIndex": 6, "fieldName": "documentNumber" },
    { "columnIndex": 7, "fieldName": "gloss" },
    { "columnIndex": 8, "fieldName": "transferredAccount" },
    { "columnIndex": 11, "fieldName": "credits" },
    { "columnIndex": 14, "fieldName": "state" }
  ]
}
```

### skipRows

Número de filas a saltar desde el inicio del archivo. Las primeras `skipRows` filas se ignoran completamente.

- `skipRows: 0` → procesa desde la fila 1
- `skipRows: 2` → salta filas 1-2, procesa desde fila 3
- `skipRows: 3` → salta filas 1-3, procesa desde fila 4

**Importante:** El encabezado del CSV también debe estar en las filas saltadas. Si el archivo tiene un encabezado en la fila 3, use `skipRows: 3`.

### columnIndex

Índice de columna **1-indexado**. La primera columna del CSV es índice 1, la segunda es 2, etc.

Ejemplo con CSV simple:
```
Col1,Col2,Col3,Col4,Col5,Col6,Col7
,46097,COMPRA,50274,1000000002,1,23055
```

- `columnIndex: 1` → "Col1" (o vacío en la fila de datos)
- `columnIndex: 2` → "46097" (fecha como serial Excel)
- `columnIndex: 5` → "1000000002" (operationCode)
- `columnIndex: 6` → "1" (documentNumber)
- `columnIndex: 7` → "23055" (gloss)

### fieldName

Nombre del campo en el DTO. Debe coincidir exactamente con el nombre de la propiedad en el DTO.

### ¿Qué pasa si una columna está fuera de rango?

Si `columnIndex` apunta a una columna que no existe en la fila (ej: `columnIndex: 20` pero el CSV solo tiene 7 columnas), el valor será `undefined` y:
- Si el campo tiene `@IsOptional()` → se ignora
- Si el campo es requerido → la validación del DTO falla

---

## 8. Resumen de riesgos

| Riesgo | Impacto | Mitigación |
|--------|---------|------------|
| 2 imports mismo config | Intercalado de IDs | `runSequentially` evita paralelismo por config name |
| Mismo archivo subido 2 veces | Duplicados en BD | Deduplicación por SHA256 hash + status COMPLETED |
| Reinicio de Collections | Tracker perdido, rollback imposible | Usar BD compartida para tracker (futuro) |
| Gateway caído | ImportRecord queda PROCESSING | Script admin para reconciliar |
| FTP caído | No se puede subir | Error inmediato, no hay datos perdidos |
| Timeout NATS 30s | Rollback automático | Aumentar timeout si el batch es muy grande |
| Fecha en formato incorrecto | DTO rechaza la fila | `parseDate` maneja múltiples formatos |
| CSV sin encabezado | Fila de headers procesada como data | Ajustar `skipRows` para saltar encabezado |
