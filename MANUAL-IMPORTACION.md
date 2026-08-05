# Manual de Importación de Archivos CSV / Excel

> Este manual explica cómo subir archivos (CSV, Excel) al sistema para que los datos queden guardados automáticamente en la base de datos.  
> Está escrito pensando en personas que no son programadoras. Si aparece alguna palabra en inglés, se explica al lado.

---

## 1. idea general

El sistema funciona así:

1. **Usted prepara un archivo** con los datos (puede ser CSV o Excel).
2. **Lo sube al sistema** mediante un programa llamado **curl** (se explica más abajo).
3. **El sistema lo procesa solo**: valida que los datos estén bien, los guarda en la base de datos y le avisa si todo salió bien o si hubo errores.

No necesita saber programar para usar este manual. Solo necesita seguir los pasos.

---

## 2. qué es un archivo CSV

CSV son las siglas en inglés de "valores separados por comas".  
Es un archivo de texto plano (como un bloc de notas) donde cada columna está separada por una coma `,`.

Ejemplo de un CSV:

```
basura,columna2,columna3,columna4,codigo,numero,glosa,cuenta
encabezado,2024-01-15,algo,otro,OP001,123456,Pago a proveedor,789-01
,2024-01-16,x,x,OP002,789012,Transferencia,789-02
```

- La **primera fila** suele tener basura o títulos que el sistema salta automáticamente.
- Las **filas siguientes** son los datos reales.

### ¿Qué es una fila?

Una **fila** es una línea completa del archivo. Cada fila representa un registro (por ejemplo, un pago, un cliente, etc.).

### ¿Qué es una columna?

Una **columna** es la posición del dato dentro de la fila. Por ejemplo, en la fila `,2024-01-16,x,x,OP002`, la fecha `2024-01-16` está en la **columna 2** (contando desde la izquierda), y el código `OP002` está en la **columna 5**.

---

## 3. cómo preparar el archivo

Antes de subir un archivo, el sistema necesita saber:

- **Cuántas filas saltarse al inicio**: muchas veces el archivo trae filas de encabezado, títulos o basura que no deben procesarse. El sistema las salta automáticamente.
- **En qué columna empiezan los datos**: a veces la primera columna es un número de fila o un checkbox que no nos interesa.
- **Qué significa cada columna**: por ejemplo, la columna 2 es la fecha, la columna 6 es el número de documento, etc.

Todo esto se configura una sola vez (ver sección 5).

### Reglas para el archivo

| Tipo de archivo | Tamaño máximo | Notas |
|----------------|---------------|-------|
| CSV             | Sin límite    | Usa comas para separar columnas |
| Excel .xlsx     | Sin límite    | El que usa Excel 2007 en adelante |
| Excel .xls      | 500 KB        | El formato viejo de Excel (2003) |

> **Nota sobre .xls**: Si su archivo es .xls (formato antiguo) y pesa más de 500 KB, debe convertirlo a .xlsx (Excel nuevo) o a CSV.

---

## 4. cómo subir el archivo (el paso más importante)

Para subir el archivo necesita usar un programa llamado **curl**.  
`curl` es un programa de línea de comandos que sirve para enviar archivos a través de internet.

### En Windows

1. Abra **Símbolo del sistema** (tecla Windows + R, escriba `cmd` y presione Enter).
2. Escriba el comando que se indica abajo.

### En Mac o Linux

1. Abra la **Terminal**.
2. Escriba el comando que se indica abajo.

### El comando mágico

Abra su terminal (símbolo del sistema en Windows) y escriba:

```bash
curl -X POST http://localhost:3000/api/import/nombre_de_la_importacion -F "file=@ruta/del/archivo.csv"
```

Reemplace:

- `nombre_de_la_importacion` → el nombre que se le dio a la configuración (ej: `extractos_bancarios`).
- `ruta/del/archivo.csv` → la ubicación de su archivo en su computadora.

**Ejemplo real:**

```bash
curl -X POST http://localhost:3000/api/import/extractos_bancarios -F "file=@C:\Users\Juan\Descargas\extracto.csv"
```

### ¿Qué significa cada parte?

| Parte | Significado |
|-------|-------------|
| `curl` | El programa que estamos usando |
| `-X POST` | Le dice al programa que vamos a **enviar** datos (no solo leer) |
| `http://localhost:3000/api/import/...` | La dirección del sistema (puerto 3000 = el sistema) |
| `-F "file=@..."` | Le dice que vamos a enviar un archivo |

---

## 5. Respuestas del sistema (qué significan)

### Si todo salió bien

El sistema responde con un mensaje como este:

```json
{
  "recordId": 1,
  "configName": "extractos_bancarios",
  "rowStart": 1001,
  "rowEnd": 1500,
  "totalRows": 500,
  "ftpPath": "/imports/extractos_bancarios/2026-06-03/12345_archivo.csv",
  "message": "Importación extractos_bancarios completada. IDs 1001 a 1500 (500 registros)."
}
```

| Campo | Significado |
|-------|-------------|
| `recordId` | Número de identificación de esta importación |
| `configName` | El nombre de la configuración que se usó |
| `rowStart` / `rowEnd` | El primer y último número de identificación (ID) que se asignaron en la base de datos |
| `totalRows` | Cuántas filas se importaron |
| `message` | Mensaje en español diciendo que todo salió bien |

### Si algo salió mal

```json
{
  "statusCode": 400,
  "message": "Datos inválidos: Fila 1: documentNumber must be a number"
}
```

Esto significa que en la fila 1 hay un dato que no es válido. Por ejemplo, el campo `documentNumber` (número de documento) debe ser un número, pero en el archivo viene como texto.

**Errores comunes:**

| Mensaje de error | Significado |
|-----------------|-------------|
| `must be a number` | El sistema esperaba un número pero encontró letras o símbolos |
| `must be a valid ISO 8601 date string` | La fecha no está en formato correcto (debe ser `AAAA-MM-DD`, ejemplo: `2024-01-15`) |
| `must be one of the following values` | El valor no está en la lista de opciones permitidas |
| `Microservice Unavailable` | El sistema interno no está disponible. Avisar al administrador. |
| `Unexpected file format` | El archivo no es CSV ni Excel. Revisar la extensión. |

---

## 6. cómo crear una nueva configuración (para administradores)

Si necesita importar un tipo de archivo nuevo (por ejemplo, "planillas de préstamos"), debe crear una **configuración** primero.

### ¿Qué es una configuración?

Es un conjunto de reglas que le dicen al sistema:

- Cómo se llama esta importación (ej: "planilla_prestamos")
- A qué tabla de la base de datos van los datos
- Qué columnas del archivo corresponden a qué campos

### Cómo crearla con curl

```bash
curl -X POST http://localhost:3000/api/import/configs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "planilla_prestamos",
    "microservice": "collections",
    "schema": "collections",
    "table": "loans",
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

### Explicación de cada campo

| Campo | Qué significa |
|-------|---------------|
| `name` | Nombre que le ponemos a esta importación. Se usa después en la URL para subir archivos |
| `microservice` | El sistema interno que va a guardar los datos (normalmente es `collections`) |
| `schema` | El esquema de la base de datos (normalmente es el mismo nombre del microservicio) |
| `table` | El nombre de la tabla donde se guardarán los datos |
| `skipRows` | Cuántas filas del archivo debe saltarse el sistema antes de empezar a leer datos |
| `startColumn` | Desde qué columna debe empezar a leer (la columna 1 a veces no sirve) |
| `delimiter` | El separador de columnas en el CSV (casi siempre es la coma) |
| `columnMappings` | La lista de "la columna X del archivo corresponde al campo Y". Cada entrada tiene `columnIndex` (número de columna, empezando en 1) y `fieldName` (nombre del campo en el sistema) |

### Cómo listar las configuraciones existentes

```bash
curl http://localhost:3000/api/import/configs
```

Esto muestra todas las configuraciones que ya existen.

### Cómo ver una configuración específica

```bash
curl http://localhost:3000/api/import/configs/extractos_bancarios
```

### Cómo eliminar una configuración

```bash
curl -X DELETE http://localhost:3000/api/import/configs/extractos_bancarios
```

---

## 7. consejos útiles

### Si el archivo tiene muchas filas

No se preocupe. El sistema procesa el archivo por partes (lotes de 500 filas) para no quedarse sin memoria. Usted sube el archivo completo y el sistema lo parte solo.

### Si el archivo tiene errores en algunas filas

El sistema valida todas las filas antes de guardarlas. Si encuentra un error en la fila 501, las 500 primeras filas que ya se guardaron se eliminan automáticamente para no dejar datos a medias. Es decir, **todo o nada**: si algo falla, no queda ningún dato guardado.

### Formato de fecha

Las fechas deben escribirse como `AÑO-MES-DÍA`. Ejemplos:
- `2024-01-15` → 15 de enero de 2024
- `2024-12-31` → 31 de diciembre de 2024

### Números de documento

Los números de documento (cédula, RUC, etc.) deben ser solo números, sin puntos, guiones ni letras.
- Correcto: `1234567`
- Incorrecto: `1.234.567` o `1234567-8` o `AB123456`

---

## 8. palabras técnicas explicadas

| Palabra | Explicación |
|---------|-------------|
| Puerto | Un número que identifica al sistema en su computadora. El sistema usa el puerto **3000** |
| Servidor | La computadora donde está instalado el sistema |
| Base de datos (DB) | El lugar donde se guardan los datos de forma permanente |
| Tabla | Una "hoja" dentro de la base de datos que contiene un tipo específico de datos |
| API | Un conjunto de comandos que el sistema entiende. Las URLs como `/api/import/...` son parte de la API |
| JSON | Un formato para escribir datos que el sistema entiende fácilmente. Usa `{}` para agrupar y `,` para separar |
| ID | Número de identificación único que el sistema asigna a cada registro |
| Rollback | Acción de deshacer: si algo sale mal, el sistema borra todo lo que ya había guardado |
| Curl | Programa que permite enviar archivos y comandos a través de la red |
| Endpoint | (se pronuncia "end-point") Es una dirección o ruta específica del sistema, como `/api/import/configs` |

---

## 9. resumen rápido

```
1. Asegurarse de que la configuración exista
   → curl http://localhost:3000/api/import/configs

2. Subir el archivo
   → curl -X POST http://localhost:3000/api/import/mi_importacion -F "file=@archivo.csv"

3. Revisar la respuesta
   → Si sale "completada", todo bien
   → Si sale "Datos inválidos", revisar el archivo y corregir
```

Si tiene dudas, pregunte al administrador del sistema.
